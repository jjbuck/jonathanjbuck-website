---
title: "Migrating Mongodb to Dynamodb"
date: 2019-12-15T14:35:47-08:00
draft: true
---

# Migrating from Mongo to Dynamo

I recently worked through the final project in a MOOC called The Results-Oriented Web Developer Course on [Udemy](https://www.udemy.com/course/result-oriented-web-developer-course/). The final project walks through the creation of a Node/Express web application, and deploys it to Herkou and uses MongoDB for the data store.

I wanted to migrate this application to Elastic Beanstalk and DynamoDB so that I'd have more control over the deployment and hosting of the application. Also, I work at AWS, so I'm a little more familiar with AWS services in general. Having said that, I've never migrated an application from MongoDB to DynamoDB, so I wanted to keep a record of the process and the things I learned. Changing from Herkou to Elastic Beanstalk is pretty straightforward, so this blog focuses on the MongoDB-to-DynamoDB transition.

## DynamoDB Local
To facilitate local testing, I use ["DynamoDB Local"](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.html) - a downloadable version of DynamoDB that runs locally .

### Getting Setup



DynamoDB local is available in a few different flavors. I'm using the [Docker version](https://hub.docker.com/r/amazon/dynamodb-local).

Once I've downloaded the Docker image with `docker pull amazon/dynamodb-local`, I can spin up a container
with `docker run -p 8000:8000 amazon/dynamodb-local`. 

To verify that it's up and running, I can type

    aws dynamodb list-tables --endpoint-url http://localhost:8000

...which returns the following.

    {
        "TableNames": []
    }


#### Creating Local Tables
To create some of the local tables that are needed for this application, I can use the CLI as shown below.

    aws dynamodb create-table --attribute-definitions AttributeName=email,AttributeType=S --table-name users --key-schema AttributeName=email,KeyType=HASH --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5 --endpoint-url http://localhost:8000

To delete the table I can type the following.

    aws dynamodb delete-table --table-name users --endpoint-url http://localhost:8000


For simplicity, I don't have to type the whole table definition at the command line. Instead, I can write the JSON object and refer to it in the request.

    aws dynamodb create-table --cli-input-json file://../dynamodb/users.json --endpoint-url http://localhost:8000


Note that DynamoDB is schema-less. This means we don't need to include any non-key attributes in AttributeDefinitions. If we include "id", "email," and "password" in the AttributeDefinitions, it will expect us to provide those same attributes in the KeySchema field. It's perfectly satisfactory to include only the primary/partition key. For our users, this will be the email. For posts, callback requests, and email requests, this will be a dynamically generated unique ID.


## Creating Production Tables
To create the production tables, I can re-use the previous requests and remove the `--endpoint_url` parameter.

    aws dynamodb create-table --cli-input-json file://../dynamodb/users.json --region us-west-2

    aws dynamodb create-table --cli-input-json file://../dynamodb/posts.json --region us-west-2

Because this leads to a growing number of CLI inputs, I'd rather just define the whole collection of tables with CloudFormation.

## Creating Tables with CloudFormation.
foo

To deploy, I type 

    aws cloudformation deploy --template-file ./dynamodb/dynamodb_cloudformation.template.yaml --stack-name TravelApp

Now let's look at each of these tables in more detail.

## Users Table
I have a Users model defined with a Mongoose schema like so:

    let mongoose = require('mongoose');
    let Schema = mongoose.Schema;

    let userSchema = new Schema({
        email: String,
        password: String
    })

    let User = mongoose.model('User', userSchema, 'users');

    module.exports = {
        User: User
    } 


The definition for my DynamoDB users table is admittedly more verbose:

    {
        "AttributeDefinitions": [
            {
                "AttributeName": "email",
                "AttributeType": "S"
            }
        ],
        "TableName": "Users",
        "KeySchema": [
            {
                "KeyType": "HASH",
                "AttributeName": "email"
            }
        ],
        "ProvisionedThroughput": {
            "WriteCapacityUnits": 5, 
            "ReadCapacityUnits": 5
        }
    }


### Create User
PutItem

### Get User
I can read from the table directly

    aws dynamodb get-item --table-name Users --key '{"email": {"S": "foo@test.com"}}'

## Posts Table

I have a posts schema defined like so:

    let mongoose = require('mongoose');
    let Schema = mongoose.Schema;

    let postSchema = new Schema({
        id: String,
        title: String,
        date: Date,
        description: String,
        text: String,
        country: String,
        imageUrl: String
    })

    let Post = mongoose.model('Post', postSchema);

    module.exports = {
        Post: Post
    } 

Transforming this to the format for DynamoDB


### Create Post

One thing I struggle with here was passing any server-side errors back to the client. The "description" field of a post is implicitly populated by selecting all the text up to the first period in the user-supplied "text" field when creating a new post. In the event that the text provided by the user doesn't contain a period, then the "description" field will be an empty string. DynamoDB throws an error when you try to store an empty string. Here's what it looks like inside posts.js:

    docClient.put(params, function (err, data) {
        if (err) {
            console.log(err);
            console.error("Unable to add item. Error JSON:", JSON.stringify(err, null, 2));
            resp.status(400);
            resp.send(err.message);
        } else {
            console.log("Added item:", JSON.stringify(data, null, 2));
            resp.sendStatus(200);
        }
    })

What I want to do is propagate `err.message` to the client. Here's what create-posts.js looks like, where the POST request is made. The key thing was `let msg = await response.text()`. Without that, the response would not actually contain the error message from the server.

    fetch('/posts', {
        method: 'POST',
        body: data
    }).then(async (response) => {
        let msg = await response.text()
        console.log(msg);
        if(response.status === 400) {
            console.log('Received 400.');
            throw new Error(msg);
        }
        return msg;
    }).then((data) => window.history.go()
    ).catch((err) => {
        console.log('Inside catch.');
        alert(`${err}`);
    })

### Get Post
posts.js originally fetched the post with.

    let post = await Post.findOne({id: id});
    resp.send(post);
    
The Dynamo version is again a little more verbose:

    docClient.get(params, async function(err, data) {
        if (err) {
            console.error("Unable to read item. Error JSON:", JSON.stringify(err, null, 2));
        } else {
            console.log("GetItem succeeded:", JSON.stringify(data, null, 2));
            //let user = JSON.stringify(data.Item, null, 2);
            let post = data.Item;
            console.log(post);
        }
    })

One of the funny things about this application is that the GET /posts/:id route isn't actually used. There's a separate route /sights that is used in main.js to populate the list of locations on the Home page. So I go to `app.js`, where I find something identical to `posts.js`:

    app.get('/sight', async (req, resp) => {
        let id = req.query.id;
        let post = await Post.findOne({
            id: id
        })
        ...
    )}

Now I copy over the changes that I made to posts.js. 

Ultimately I need to update the root route of /posts/. This is used on the home page and the admin page to return an array of all posts, which is then iterated over to populate each of the individual fields.

    router.get('/', async (req, resp) => {
        let posts = await Post.find();
        resp.send(posts);
    })


To get all the items from a DynamoDB table, we have to use the `scan` operation [https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB/DocumentClient.html#scan-property]. The update for DynamoDB looks like this:

    router.get('/', async (req, resp) => {
        var params = {
            TableName : table
        };
        docClient.scan(params, function(err, data) {
            if (err) {
                console.log(err);
            } else {
                console.log(data);
                posts = data.Items;
                console.log(posts);
                resp.send(posts);
            }
        })
    })


### Delete Post

Now that we've got create and get updated, also need to update the delete route [https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB/DocumentClient.html#delete-property]. 

    router.delete('/:id', authMiddleware, async (req, resp) => {
    let id = req.params.id;
    //await Post.deleteOne({id: id});
    var params = {
        TableName: table,
        Key: {
            'id': id
        }
    };
    docClient.delete(params, function(err, data) {
        if (err) console.log(err);
        else console.log(data);
      });
    resp.send('Deleted.');
})


## Callback Requests
This is the table to store information from customers who want to receive a phone call with more information. The model schema is shown below.

    let callbackRequestSchema = new Schema({
        id: String,
        phoneNumber: String,
        date: Date
    })

### Get Callback Requests

The root route for callback requests in `/routes/callback-requests.js` gets all entries from the database.

    router.get('/', authMiddleware, async (req,resp) => {
        resp.send(await CallbackRequest.find());
    });

To update this, I model the change made in the root route for posts, where a similar request is made.

### Create Callback Request
The original route is shown below.

    let reqBody = req.body;
        let newRequest = new CallbackRequest({
            id: uniqid(),
            phoneNumber: reqBody.phoneNumber,
            date: new Date()
        })

The change here is similar to the change we made in the POST method for the /posts route.

    router.post('/', async (req,resp) => {
        let reqBody = req.body;
        let newCallbackRequest = {
                'id': uniqid(),
                'phoneNumber': reqBody.phoneNumber,
                'date': String(new Date())
            };
            let params = {
                TableName: table,
                Item: newCallbackRequest
            };
            console.log("Adding a new callback request...");
            console.log(params)

            docClient.put(params, function (err, data) {
                if (err) {
                    console.log(err);
                    console.error("Unable to add item. Error JSON:", JSON.stringify(err, null, 2));
                    resp.status(400);
                    resp.send(err.message);
                } else {
                    console.log("Added item:", JSON.stringify(data, null, 2));
                    resp.sendStatus(200);
                }
            })
        });


## Emails Table

The last table we have to update is the table that stores email addresses of customers who wish to be contacted with more information. This is again very similar to the approach required to update the Posts table.


# Other Caveats

## Issues with bcrypt

I had to add a .npmrc file with the following:
`unsafe-perm=true` to prevent issues with bcrypt

npm ERR! errno -2
npm ERR! enoent ENOENT: no such file or directory, open '/var/app/current/package.json'
npm ERR! enoent This is related to npm not being able to find a file.
npm ERR! enoent 
internal/modules/cjs/loader.js:807
  return process.dlopen(module, path.toNamespacedPath(filename));
                 ^

Error: /var/app/current/node_modules/bcrypt/lib/binding/bcrypt_lib.node: invalid ELF header
    at Object.Module._extensions..node (internal/modules/cjs/loader.js:807:18)
    at Module.load (internal/modules/cjs/loader.js:653:32)
    at tryModuleLoad (internal/modules/cjs/loader.js:593:12)
    at Function.Module._load (internal/modules/cjs/loader.js:585:3)
    at Module.require (internal/modules/cjs/loader.js:692:17)
    at require (internal/modules/cjs/helpers.js:25:18)
    at Object.<anonymous> (/var/app/current/node_modules/bcrypt/bcrypt.js:6:16)
    at Module._compile (internal/modules/cjs/loader.js:778:30)
    at Object.Module._extensions..js (internal/modules/cjs/loader.js:789:10)
    at Module.load (internal/modules/cjs/loader.js:653:32)
    at tryModuleLoad (internal/modules/cjs/loader.js:593:12)
    at Function.Module._load (internal/modules/cjs/loader.js:585:3)
    at Module.require (internal/modules/cjs/loader.js:692:17)
    at require (internal/modules/cjs/helpers.js:25:18)
    at Object.<anonymous> (/var/app/current/routes/users.js:4:14)
    at Module._compile (internal/modules/cjs/loader.js:778:30)
internal/modules/cjs/loader.js:807
  return process.dlopen(module, path.toNamespacedPath(filename));




## Port
Apparently nginx's default port is 8081. I couldn't figure out how to change this in elastic beanstalk, so I had to 
change my app to use that port in the app.js file.
`const port = process.env.PORT || 3000`

This also necessitates changes throughout the app. Rather than define the full route http://localhost:3000/<resource>, we delete the prefix and use a relative route.


## Note for Modern Mac OSX

For Modern macos/OSX, you need to find your ~/Library/Python/$version/bin directory and add it to your $PATH. This will help you locate the one where aws got installed.

$ ls -d ~/Library/Python/*/bin/aws
/Users/bbronosky/Library/Python/3.6/bin/aws
So based on that I added this line to my .bashrc

export PATH=$HOME/Library/Python/3.6/bin:$PATH

## Body is disturbed or locked

I ran into this error a lot. It seems to be thrown when in the context of parsing the response from fetch. In particular, when I would attempt to call then(async (response) => {
        let msg = await response.text()

response.text() multiple times. The solution is to call .text() once and assign it to a variable (in this case `msg`), and then reference the variable going forward.

## Deploying to Elastic Beanstalk

Rather than deploy to Heroku, I wanted to deploy to Elastic Beanstalk. Since I went about the environment creation and deployment in a sort of ad hoc way, the role autogenerated by the Elastic Beanstalk CLI doesn't have permissions to invoke any DynamoDB resources. Since this was just a toy project, I just manually edited this role.


# Useful Resource

* https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GettingStarted.NodeJs.html
* https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB/DocumentClient.html
* https://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/the-response-object.html
* https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb-cli3.html