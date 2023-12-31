# AWS-Serverless
Project Overview And High Level Design
Let's start with the High Level Design. High Level Design An Amazon API Gateway is a collection of resources and methods. For this project , I created one resource (DynamoDBManager) and define one method (POST) on it. The method is backed by a Lambda function (LambdaFunctionOverHttps). That is, when you call the API through an HTTPS endpoint, Amazon API Gateway invokes the Lambda function.

The POST method on the DynamoDBManager resource supports the following DynamoDB operations:

Create, update, and delete an item.
Read an item.
Scan an item.
Other operations (echo, ping), not related to DynamoDB, that you can use for testing.
The request payload you send in the POST request identifies the DynamoDB operation and provides necessary data. For example:

The following is a sample request payload for a DynamoDB create item operation:

{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1",
            "name": "Bob"
        }
    }
}
The following is a sample request payload for a DynamoDB read item operation:

{
    "operation": "read",
    "tableName": "lambda-apigateway",
    "payload": {
        "Key": {
            "id": "1"
        }
    }
}
SET UP
Create Lambda IAM Role
Create the execution role that gives your function permission to access AWS resources.

To create an execution role

Open the roles page in the IAM console.
Choose Create role.
Create a role with the following properties.
Trusted entity – Lambda.
Role name – lambda-apigateway-role.
Permissions – Custom policy with permission to DynamoDB and CloudWatch Logs. This custom policy has the permissions that the function needs to write data to DynamoDB and upload logs.
{
"Version": "2012-10-17",
"Statement": [
{
  "Sid": "Stmt1428341300017",
  "Action": [
    "dynamodb:DeleteItem",
    "dynamodb:GetItem",
    "dynamodb:PutItem",
    "dynamodb:Query",
    "dynamodb:Scan",
    "dynamodb:UpdateItem"
  ],
  "Effect": "Allow",
  "Resource": "*"
},
{
  "Sid": "",
  "Resource": "*",
  "Action": [
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents"
  ],
  "Effect": "Allow"
}
]
}
Create Lambda Function
To create the function

Click "Create function" in AWS Lambda Console
Create function

Select "Author from scratch". Use name LambdaFunctionOverHttps , select Python 3.7 as Runtime. Under Permissions, select "Use an existing role", and select lambda-apigateway-role that we created, from the drop down

Click "Create function"

Lambda basic information

Replace the boilerplate coding with the following code snippet and click "Save"
Example Python Code

from __future__ import print_function

import boto3
import json

print('Loading function')


def lambda_handler(event, context):
    '''Provide an event that contains the following keys:

      - operation: one of the operations in the operations dict below
      - tableName: required for operations that interact with DynamoDB
      - payload: a parameter to pass to the operation being performed
    '''
    #print("Received event: " + json.dumps(event, indent=2))

    operation = event['operation']

    if 'tableName' in event:
        dynamo = boto3.resource('dynamodb').Table(event['tableName'])

    operations = {
        'create': lambda x: dynamo.put_item(**x),
        'read': lambda x: dynamo.get_item(**x),
        'update': lambda x: dynamo.update_item(**x),
        'delete': lambda x: dynamo.delete_item(**x),
        'list': lambda x: dynamo.scan(**x),
        'echo': lambda x: x,
        'ping': lambda x: 'pong'
    }

    if operation in operations:
        return operations[operation](event.get('payload'))
    else:
        raise ValueError('Unrecognized operation "{}"'.format(operation))
Lambda Code

Test Lambda Function
Let's test our newly created function. We haven't created DynamoDB and the API yet, so we'll do a sample echo operation. The function should output whatever input we pass.

Click the arrow on "Select a test event" and click "Configure test events"
Configure test events

Paste the following JSON into the event. The field "operation" dictates what the lambda function will perform. In this case, it'd simply return the payload from input event as output. Click "Create" to save
{
    "operation": "echo",
    "payload": {
        "somekey1": "somevalue1",
        "somekey2": "somevalue2"
    }
}
Save test event

Click "Test", and it will execute the test event. You should see the output in the console
Execute test event

We're all set to create DynamoDB table and an API using our lambda as backend!

Create DynamoDB Table
Create the DynamoDB table that the Lambda function uses.

To create a DynamoDB table

Open the DynamoDB console.
Choose Create table.
Create a table with the following settings.
Table name – lambda-apigateway
Primary key – id (string)
Choose Create.
create DynamoDB table

Create API
To create the API

Go to API Gateway console
Click Create API
create API

Scroll down and select "Build" for REST API
Build REST API

Give the API name as "DynamoDBOperations", keep everything as is, click "Create API"
Create REST API

Each API is collection of resources and methods that are integrated with backend HTTP endpoints, Lambda functions, or other AWS services. Typically, API resources are organized in a resource tree according to the application logic. At this time you only have the root resource, but let's add a resource next
Click "Actions", then click "Create Resource"

Create API resource

Input "DynamoDBManager" in the Resource Name, Resource Path will get populated. Click "Create Resource"
Create resource

Let's create a POST Method for our API. With the "/dynamodbmanager" resource selected, Click "Actions" again and click "Create Method".
Create resource method

Select "POST" from drop down , then click checkmark
Create resource method

The integration will come up automatically with "Lambda Function" option selected. Select "LambdaFunctionOverHttps" function that we created earlier. As you start typing the name, your function name will show up.Select and click "Save". A popup window will come up to add resource policy to the lambda to be invoked by this API. Click "Ok"
Create lambda integration

Our API-Lambda integration is done!

Deploy the API
In this step, you deploy the API that you created to a stage called prod.

Click "Actions", select "Deploy API"
Deploy API

Now it is going to ask you about a stage. Select "[New Stage]" for "Deployment stage". Give "Prod" as "Stage name". Click "Deploy"
Deploy API to Prod Stage

We're all set to run our solution! To invoke our API endpoint, we need the endpoint url. In the "Stages" screen, expand the stage "Prod", select "POST" method, and copy the "Invoke URL" from screen
Copy Invoke Url

Running our solution
The Lambda function supports using the create operation to create an item in your DynamoDB table. To request this operation, use the following JSON:
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1234ABCD",
            "number": 5
        }
    }
}
To execute our API from local machine, we are going to use Postman.

To run this from Postman, select "POST" , paste the API invoke url. Then under "Body" select "raw" and paste the above JSON. Click "Send". API should execute and return "HTTPStatusCode" 200.

Execute from Postman

To validate that the item is indeed inserted into DynamoDB table, go to Dynamo console, select "lambda-apigateway" table, select "Items" tab, and the newly inserted item should be displayed.

Dynamo Item

To get all the inserted items from the table, we can use the "list" operation of Lambda using the same API. Pass the following JSON to the API, and it will return all the items from the Dynamo table
{
    "operation": "list",
    "tableName": "lambda-apigateway",
    "payload": {
    }
}
List Dynamo Items

We have successfully created a serverless API using API Gateway, Lambda, and DynamoDB!
