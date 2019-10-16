# Workshop for Lambda Overview

## Pre-reqs for this workshop

### Running this on local machine:

- Install the AWS SAM CLI
  - [Guide](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html)

## Workshop

### Initializing a new Lambda function

1. To create a new Lambda function you will need to run this command:

```bash
$ sam init -r nodejs
```

This command will create a node application that will look something like this:

```bash
$ tree sam-app/
sam-app/
├── events
│   └── event.json
├── hello-world
│   ├── app.js
│   ├── package.json
│   └── tests
│       └── unit
│           └── test-handler.js
├── README.md
└── template.yaml
```

The `template.yaml` file contains a SAM template file describing the infrastructure for the application we just created. The `app.js` will contain all the Lambda code that we will be writing.

### Adding your dependencies 

Now we are going to add a dependency for our application to allow us to add dates to our Hello World application.

To do this we will need to cd into our new application directory and install the dependency.

```bash
$ cd sam-app/hello-world/
$ npm install axios moment
```

You should now see a new `node_modules/` folder in your `hello_world` application folder.

### Diving into the code

Now that our enviroment is set up we are going to dive into the `app.js` code. We are going to add the `moment` package we just installed to our existing application by changing the message from `Hello World` to `Today is <insert day>`. If you want to use axios functionality to return the ip address uncomment the lines or use the code below.

<details>
  <summary>Code example:</summary>
  
  This is an example of what your code could look like:

  ```javascript
  const moment = require('moment');

  const axios = require('axios')
  const url = 'http://checkip.amazonaws.com/';
  let response;

  exports.lambdaHandler = async (event, context) => {
      try {
          const ret = await axios(url);
          response = {
              'statusCode': 200,
              'body': JSON.stringify({
                  message: `Today is ${ moment().format('dddd')}`,
                  location: ret.data.trim()
              })
          }
      } catch (err) {
          console.log(err);
          return err;
      }

      return response
  };
  ```
</details>

### Locally testing the function

Now it is time to test our function out. To do this we will first create a test event file. For this example we will create a file called `event.json` in our `sam-app` folder. (This is different then our other `event.json` for simplicity)

```bash
$ cd ../
$ pwd
# Confirm this is sam-app
$ echo '{}' > event.json
```

Now we should have an `event.json` file in `sam-app`.

Now we will do a test invoke on our function and see something similiar:

```bash
$ sam local invoke "HelloWorldFunction" -e event.json

{"statusCode":200,"body":"{\"message\":\"Today is Thursday\",\"location\":\"54.208.174.178\"}"}
```

### Stepping through and debugging your lambda function

**Note:** These steps required you to have VSCode installed on your machine. If you don't have it installed please get the most recent version from [here](https://code.visualstudio.com/).

Open up VSCode for your project that your are current running. And select the debug icon on the right hand side. (It is the bug icon)

Click on the gear icon in the top right and select nodejs. It should open up a `launch.json` file. Replace the contents in that file from the defaults to this:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Attach to SAM CLI",
            "type": "node",
            "request": "attach",
            "address": "localhost",
            "port": 5858,
            "localRoot": "${workspaceRoot}/hello-world",
            "remoteRoot": "/var/task",
            "protocol": "inspector",
            "stopOnEntry": false
        }
    ]
}
```

Now in a terminal window run the Lambda function locally one more time but this time include an additional flag like so:

```bash
$ sam local invoke "HelloWorldFunction" -e event.json -d 5858
```

### Adding layers to your function

Now that we have a functional function we will now add in a Lambda layer to manage our dependancies that we added to our function.

First lets create a Lambda layer directory in `sam-app`.

```bash
$ mkdir -p dependencies/nodejs
$ touch dependencies/nodejs/package.json
```

Now open up `package.json` that we just created in `dependencies/nodejs/` and add our depedencies for our application. Should look something like this:

```json
{
  "dependencies": {
    "axios": "^0.19.0",
    "moment": "^2.24.0"
  }
}
```

Now that we have our new dependencies we can now run `npm install` in that directory. Like so:

```bash
$ cd dependencies/nodejs/
$ npm install
```

Now that we have moved our `node_modules` from our application to a layer we can now go ahead and delete our `node_modules` folder and we can remove the dependencies section from our `package.json`. 

```bash
$ cd ../../hello-world/
$ rm -r node_modules
$ open package.json
```

<details>
  <summary>Your `package.json` should look something like this:</summary>

  This is an example `package.json`

  ```json
  {
    "name": "hello_world",
    "version": "1.0.0",
    "description": "hello world sample for NodeJS",
    "main": "app.js",
    "repository": "https://github.com/awslabs/aws-sam-cli/tree/develop/samcli/local/init/templates/cookiecutter-aws-sam-hello-nodejs",
    "author": "SAM CLI",
    "license": "MIT",
    "scripts": {
      "test": "mocha tests/unit/"
    },
    "devDependencies": {
      "chai": "^4.2.0",
      "mocha": "^6.1.4"
    }
  }
  ```
</details>


After the directory is setup we need to update the `template.yaml` with the correct information for the Lambda to find the layer. We will be making two changes. The first change we will be adding is a new resource, the resource we are making is the Lambda Layer. The second change is adding a `Layers` property in our Lambda function that references the new Layer resource we create. An example of the new template can be found below:

<details>
  <summary>Example template:</summary>

  This is an example template:

  ```yaml
  AWSTemplateFormatVersion: '2010-09-09'
  Transform: AWS::Serverless-2016-10-31
  Description: >
    sam-app

    Sample SAM Template for sam-app
    
  Globals:
    Function:
      Timeout: 3

  Resources:
    HelloWorldFunction:
      Type: AWS::Serverless::Function
      Properties:
        CodeUri: hello-world/
        Handler: app.lambdaHandler
        Layers:
          - !Ref HelloWorldDepLayer
        Runtime: nodejs10.x
        Events:
          HelloWorld:
            Type: Api
            Properties:
              Path: /hello
              Method: get
    HelloWorldDepLayer:
      Type: AWS::Serverless::LayerVersion
      Properties:
          LayerName: sam-app-dependencies
          Description: Dependencies for sam app
          ContentUri: dependencies/
          CompatibleRuntimes:
            - nodejs6.10
            - nodejs8.10
            - nodejs10.x
          LicenseInfo: 'MIT'
          RetentionPolicy: Retain

  Outputs:
    HelloWorldApi:
      Description: "API Gateway endpoint URL for Prod stage for Hello World function"
      Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/hello/"
    HelloWorldFunction:
      Description: "Hello World Lambda Function ARN"
      Value: !GetAtt HelloWorldFunction.Arn
    HelloWorldFunctionIamRole:
      Description: "Implicit IAM Role created for Hello World function"
      Value: !GetAtt HelloWorldFunctionRole.Arn

  ```
</details>

Now that we have setup our directory correctly and updated our template we can verify that we can still run our Lambda locally.

```bash
$ sam local invoke "HelloWorldFunction" -e event.json
```

### Deploying your Lambda function

Now that we can run our functions locally and we have added our layers so now we will go ahead and deploy our Lambda function and our API (created during `sam init`).

First we will need to create a resource bucket where our generated CloudFormation can live. This can be done simply by using the command below:

```bash
$ aws s3api create-bucket –bucket <your unique bucket name>
```

Now once we have your bucket name we will need to package up our Lambda function like so:

```bash
$ sam package --template-file template.yaml --s3-bucket <your bucket> --output-template-file out.yaml
```

Once that has succeeded we can now run `sam deploy` to push our function out!

```bash
$ sam deploy --template-file ./out.yaml --stack-name <your stack name> --capabilities CAPABILITY_IAM
```


## Finishing up

If you don't want the function any more in your account you can use this command to clean up your enviroment:

```bash
aws cloudformation delete-stack --stack-name <your stack name>
```

If there is any issues please open up an issue on this repo :)

