## How to create a trigger for DynamoDB in AWS Amplify

### Add a lambda function

```bash
amplify add function
? Select which capability you want to add: Lambda function (serverless function)
? Provide a friendly name for your resource to be used as a label for this category in the project: updateUser
? Provide the AWS Lambda function name: updateUser
? Choose the runtime that you want to use: NodeJS
? Choose the function template that you want to use: Lambda trigger
? What event source do you want to associate with Lambda trigger? Amazon DynamoDB Stream
? Choose a DynamoDB event source option Use API category graphql @model backed DynamoDB table(s) in the current Ampl
Selected resource myWebApp
? Choose the graphql @model(s) Item
? Do you want to access other resources in this project from your Lambda function? No
? Do you want to invoke this function on a recurring schedule? No
? Do you want to configure Lambda layers for this function? No
? Do you want to edit the local lambda function now? Yes
```

`myWebApp/amplify/backend/function/updateUser/src/index.js` 에 sample code가 추가되었다

### Implement the lambda function

We can find a auto-generated sample code like:

```javascript
exports.handler = function (event, context) {
  console.log(JSON.stringify(event, null, 2));
  event.Records.forEach((record) => {
    console.log(record.eventID);
    console.log(record.eventName);
    console.log('DynamoDB Record: %j', record.dynamodb);
  });
  context.done(null, 'Successfully processed DynamoDB record');
};
```
This code simply iterate through changed records and log the content.

If you push just as it is, actual log can be seen via AWS CloudWatch Logs.
(https://docs.aws.amazon.com/lambda/latest/dg/with-ddb.html)
```
9f9c1b07462c6eb1f29498739b44112d
MODIFY
{
    "Records": [
        {
            "eventID": "9f9c1b07462c6eb1f29498739b44112d",
            "eventName": "MODIFY",
            "eventVersion": "1.1",
            "eventSource": "aws:dynamodb",
            "awsRegion": "ap-northeast-2",
            "dynamodb": {
                "ApproximateCreationDateTime": 1604238382,
                "Keys": {
                    "id": {
                        "S": "faaa66"
                    }
                },
                "NewImage": {
                    "createdAt": {
                        "S": "2020-10-09T13:05:57.353Z"
                    },
                    "userId": {
                        "S": "46459f"
                    },
                    "name": {
                        "S": "Jane Doe"
                    },
                    ...
                },
                "OldImage": {
                    "createdAt": {
                        "S": "2020-10-09T13:05:57.353Z"
                    },
                    "userID": {
                        "S": "46459f"
                    },
                    ...
                    "name": {
                        "S": "John Doe"
                    },
                    ...
                },
                "SequenceNumber": "...",
                "SizeBytes": 442,
                "StreamViewType": "NEW_AND_OLD_IMAGES"
            },
            "eventSourceARN": "arn:aws:dynamodb:ap-northeast-2:..."
        }
    ]
}
```

We wanna use updateItem of dynamodb API (https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB.html#updateItem-property)

API requires mainly `params` field. We will set params as following.
```javascript
var params = {
  ExpressionAttributeNames: {
   "#NI": "NumItem", 
   "#P": "Price"
  }, 
  ExpressionAttributeValues: {
   ":one": {
     N: "1"
    }, 
   ":price": {
     N: record.dynamodb.NewImage.price.N  // N indicates number type
    }
  }, 
  Key: {
   "id": {
     S: "1"
    }
  }, 
  ReturnValues: "ALL_NEW", 
  TableName: "User", 
  UpdateExpression: "SET #NI = #NI + :one, #P = #P + :price"
 };
```

Since DynamoDB support incremental operation we can write a query like the last line.

As below we can push our query to DynamoDB.
```javascript
const AWS = require("aws-sdk");
var dynamodb = new AWS.DynamoDB();

...

dynamodb.updateItem(params, function (err, data) {
    if (err) console.log(err, err.stack);
    // an error occurred
    else console.log(data); // successful response
});
```

### Deploy

```bash
amplify push
```
Done!
