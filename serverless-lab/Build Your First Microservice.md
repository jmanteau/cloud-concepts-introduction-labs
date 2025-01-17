# Build Your First Microservice (NodeJS)

---
**Level:** Intermediate (200)

**Duration:** 45 min

**Estimated Cost:** < $0.1



### Lab Overview



In this lab we will create a full microservice, by using a database (Amazon DynamoDB), a computing service (AWS Lambda), and an API service to expose the service to users and other microservices.

You will build, deploy, and test the microservice.



#### Prerequisites

You need:

-   an IAM user with proper permissions to create IAM roles, DynamoDB tables, AWS Lambda functions, API Gateway resources.
-   If you are sharing the same AWS account AND region with someone else, be sure to include a `<prefix>` on the name of your resources to avoid name collisions. Use your initials, or something else unique for the pair {account, region}.
    -   If you are running the lab alone, disregard the `<prefix>` on the name of the resources.
-   Be sure of selecting an AWS region where API Gateway, Lambda, and DynamoDB are available.

___



#### Context



In this lab we are going to create a simple microservice that computes statistics for the top 10 players of a game.

The architecture is shown in the diagram below  
![](Build%20Your%20First%20Microservice/microservice.architecture.png)

-   We have the GameTopX DynamoDB table, which stores the data for the top 10 players of the game.
-   The AWS Lambda function GameTopXStats is responsible for reading the data from the DynamoDB table, and for computing the required statistics, which at this point is simply to compute the gamer performance (score/shots).
-   The GameAPI is the API Gateway that we are going to create to trigger the lambda function, and to return the data to the users (humans or consumer applications).
    -   See that we are intending to create the resource topxstatistics, and to use the querystring sessionId to provide parameters so the Lambda function can find the appropriate record on the DynamoDB table.
-   We need to create the appropriate polices for the resources, so they can access the required resources:
    -   For the Lambda function, we are going to create a GameTopXStatsLambdaRole that will allow the lambda function to read data from the table.

___



### Task: Creating the DynamoTB table



#### Step 1 - Create the table



1.  Visit the [DynamoDB console](https://console.aws.amazon.com/dynamodb/) .
2.  Visit the `Tables` menu option, and then click on `Create table`.
3.  At the Create DynamoDB table:
    1.  For `Table name`, input `<prefix>GameTopX`.
    2.  For `Partition key`, input `SessionId`.
4.  Leave everything else as it is.
5.  Click on `Create table`.

Wait until the table is created. We will use this task to insert some data into this table. After the creation is finished, you will be able to click on the table's name.



#### Step 2 - Inserting the data



1.  Go to DynamoDB, then Tables, and click on GameTopX Be sure that you have the table GameTopX selected on the DynamoDB console.
2.  Click on the button `Explore table items` button.
3.  Click on the button `Create item`. DynamoDB will open a window so you can input data on it.
4.  On the `Create item` window, be sure of working with the JSON view, not Form view.
5.  Be sure that `View DynamoDB JSON` is selected.
6.  Copy-and-paste the content below, replacing what is in that window:

```javascript
{
  "SessionId": {
    "S": "TheTestSession"
  },
  "TopX": {
    "L": [
      {
        "M": {
          "Level": {
            "N": "5"
          },
          "Lives": {
            "N": "3"
          },
          "Nickname": {
            "S": "alanis"
          },
          "Score": {
            "N": "1500"
          },
          "Shots": {
            "N": "372"
          },
          "Timestamp": {
            "S": "2019-06-21T00:07:01.328Z"
          }
        }
      },
      {
        "M": {
          "Level": {
            "N": "2"
          },
          "Lives": {
            "N": "0"
          },
          "Nickname": {
            "S": "blanka"
          },
          "Score": {
            "N": "300"
          },
          "Shots": {
            "N": "89"
          },
          "Timestamp": {
            "S": "2019-06-26T08:28:52.264Z"
          }
        }
      },
      {
        "M": {
          "Level": {
            "N": "1"
          },
          "Lives": {
            "N": "0"
          },
          "Nickname": {
            "S": "bugsbunny"
          },
          "Score": {
            "N": "200"
          },
          "Shots": {
            "N": "88"
          },
          "Timestamp": {
            "S": "2019-06-26T08:49:19.049Z"
          }
        }
      },
      {
        "M": {
          "Level": {
            "N": "1"
          },
          "Lives": {
            "N": "0"
          },
          "Nickname": {
            "S": "billythekid"
          },
          "Score": {
            "N": "195"
          },
          "Shots": {
            "N": "46"
          },
          "Timestamp": {
            "S": "2019-06-25T22:43:32.050Z"
          }
        }
      },
      {
        "M": {
          "Level": {
            "N": "1"
          },
          "Lives": {
            "N": "0"
          },
          "Nickname": {
            "S": "nobodyknows"
          },
          "Score": {
            "N": "65"
          },
          "Shots": {
            "N": "17"
          },
          "Timestamp": {
            "S": "2019-06-26T08:33:47.951Z"
          }
        }
      },
      {
        "M": {
          "Level": {
            "N": "1"
          },
          "Lives": {
            "N": "0"
          },
          "Nickname": {
            "S": "mordorlord"
          },
          "Score": {
            "N": "5"
          },
          "Shots": {
            "N": "1"
          },
          "Timestamp": {
            "S": "2019-06-26T08:29:29.639Z"
          }
        }
      },
      {
        "M": {
          "Level": {
            "N": "1"
          },
          "Lives": {
            "N": "3"
          },
          "Nickname": {
            "S": "naruto"
          },
          "Score": {
            "N": "0"
          },
          "Shots": {
            "N": "0"
          },
          "Timestamp": {
            "S": "2019-06-26T14:24:17.748Z"
          }
        }
      },
      {
        "M": {
          "Level": {
            "N": "1"
          },
          "Lives": {
            "N": "3"
          },
          "Nickname": {
            "S": "ramon"
          },
          "Score": {
            "N": "0"
          },
          "Shots": {
            "N": "0"
          },
          "Timestamp": {
            "S": "2019-06-24T17:36:38.646Z"
          }
        }
      },
      {
        "M": {
          "Level": {
            "N": "1"
          },
          "Lives": {
            "N": "3"
          },
          "Nickname": {
            "S": "bruceb"
          },
          "Score": {
            "N": "0"
          },
          "Shots": {
            "N": "0"
          },
          "Timestamp": {
            "S": "2019-06-24T12:24:12.238Z"
          }
        }
      },
      {
        "M": {
          "Level": {
            "N": "1"
          },
          "Lives": {
            "N": "3"
          },
          "Nickname": {
            "S": "hackz"
          },
          "Score": {
            "N": "1"
          },
          "Shots": {
            "N": "0"
          },
          "Timestamp": {
            "S": "2019-06-24T17:36:38.646Z"
          }
        }
      }
    ]
  }
}
```

7.  Click on the `Create item` button. Check that the record was properly created.
8.  Click on the `View table details` button.
9.  Expand the `Additional info` section, then find and copy the `Amazon Resource Name (ARN)` for the table to an auxiliary text file.

___



### Task: Creating the Lambda function



#### Step 1: Creating the Lambda function and inputting the code



1.  Visit the [AWS Lambda page](https://console.aws.amazon.com/lambda/home)  on the AWS Console
    
2.  Click on the button `Create function`. If the button is not visible, check on the console for a menu on the left, with the label Functions, and then hit on the `Create function` button
    
3.  Select `Author from scratch`.
    
4.  Under the section `Basic information`:
    
    1.  For `Function Name`, input `<prefix>GameTopXStats`
    2.  For `Runtime`, select the `Node.js 20.x`
    3.  For `Architecture`, select `arm64` 
    4.  For `Permissions`, we need to give permissions to our Lambda function to get data from the DynamoDB table that we have just created. Expand the section `Change default execution role`, and select `Use a role` and use `LabRole`.
    5.  Click on the `Create function` button. You are going to get transported to the Designer page for the function.
    6.  Scroll down to the section `Code source`.
    7.  You should see a index.mjs tab with some code in it. Use the **Lambda code snippet** below, copy it, and paste it replacing what's in index.mjs:
        -   Invest a few minutes reading the Lambda code. Even if you are not a programmer, it will be helpful to understand the overall logic behind the code: Start from the bottom, on the handler function. This is the entry point for your lambda function, where it receives the event (sent by API Gateway, or by other caller) and the context (variable which holds the information about the environment). Then follow the steps to drill down into the other functions.
    8.  Click on `Deploy`

```js
'use strict';

import { DynamoDBClient  } from "@aws-sdk/client-dynamodb"; 
import { DynamoDBDocumentClient, GetCommand } from "@aws-sdk/lib-dynamodb";

const DDBClient = new DynamoDBClient();
const DynamoDB = DynamoDBDocumentClient.from(DDBClient);



/**
* This function reads from the database. 
* You can replace this with the code depending on your database
*/
const readTopxDataFromDatabase = async function (sessionId) {
    let result = null;
    console.log('PROVIDED SESSION ID FOR DATABASE QUERY IS:', sessionId);
    let params = {
        "TableName": process.env.TABLENAME,
        "Key": {
            "SessionId": sessionId
        }
    };
    let getCommand = new GetCommand(params);
    try {
        let data = await DynamoDB.send(getCommand);
        if (data && data.Item)
            result = data.Item;
        else result = {};
    }
    catch(exception) {
        result = exception;
    }
    return result;
};
 
/**
* This function does the required computation.
*/
const computeStatisticsForSession = async function (sessionId) {
    // let's start by reading the session data from the database
    // retrieving the record attached to 'sessionId'
    try {
        let topXSessionData = await readTopxDataFromDatabase(sessionId);
        if (topXSessionData instanceof Error) {
            return topXSessionData;
        } else {
            if (!topXSessionData.TopX)
                // Table is empty. No data found.
                return null;
            else {
                let statistics = [];
                let position = 1;
                // Make the computations
                topXSessionData.TopX.forEach((item) => {
                    let itemStatistics = {};
                    itemStatistics['Nickname'] = item.Nickname;
                    itemStatistics['Position'] = position++;
                    if (item.Shots != 0) {
                        itemStatistics['Performance'] = item.Score / item.Shots;
                    } else {
                        if (item.Score != 0) itemStatistics['Performance'] = -1;
                        else itemStatistics['Performance'] = 0;
                    }
                    statistics.push(itemStatistics);
                });
                return statistics;
            }
        }
    } catch (exception) {
      console.log('Error');
      console.log(exception);
      return exception;
    }
};

/**
* This function formats the response in accordance to what is
* expected with Lambda Integration. You may need to change it
* if you move the code out of Lambda
*/
const formatResponse = function (data) {
    let response = {
        isBase64Encoded: false,
        statusCode: null,
        body: null,
        headers: {
            'Content-Type': null
        }
    };
    if (data instanceof Error) {
        console.log('ERROR');
        console.log(data);
        response.statusCode = data.statusCode;
        response.body = data.message;
        response.headers['Content-Type'] = 'text/plain';
    } else {
        console.log('SUCCESS');
        console.log(data);
        if (data == null) {
            response.statusCode = 404;
            response.headers['Content-Type'] = 'text/plain';
            response.body = 'Inexistent session';
            // If you want to test the base64 encoding, comment the line above, and uncomment those two below
            //response.isBase64Encoded = true;
            //response.body = 'SW5leGlzdGVudCBzZXNzaW9u';
        } else {
            response.statusCode = 200;
            response.headers['Content-Type'] = 'application/json';
            response.body = JSON.stringify(data);
        }
    }
    return response;
};
 
/**
* This function validates the event, and must
* be rewritten if you move this out of Lambda
*/
const validateEvent = function (event) {
    let sessionId = null;
    let isProxyIntegration = null;
    try {
        // if this structure is present, we have a direct integration
        sessionId = event.params.querystring.sessionId;
        isProxyIntegration = false;
    } catch (_) {
        try {
            // if this structure is present, we have a Proxy integration
            sessionId = event.queryStringParameters.sessionId;
            isProxyIntegration = true;
        } catch (_) {
            sessionId = null;
            isProxyIntegration = null;
        }
    }
    let result = {
        "sessionId": sessionId,
        "isProxyIntegration": isProxyIntegration
    };
    console.log(result);
    return result;
};

/**
* This is the entry-point for the Lambda function
*/
export const handler = async (event, context) => {
    let response=null;
    // Let's show the received event
    console.log(event);
    // Let's validate the event, to extract the sessionId (if we have one)
    // eventValidation will have the form { "sessionId" : sessionIdValue, "isProxyIntegration" : [ true | false ] }
    // isProxyIntegration tells us about the API Gateway integration (if any) to this lambda function
    let eventValidation = validateEvent(event);
    let sessionId = eventValidation.sessionId;
    if (!sessionId)
         // let's format the response to what is expected by an API Gateway integration
        response = formatResponse(null);
    else {
        // call the function that computes the statistics
        let statistics = await computeStatisticsForSession(sessionId);
        // let's format the response to what is expected by an API Gateway integration
        response = formatResponse(statistics);
    }
    return response;
};

```



#### Step 2 : Configuring the Environment variables section

Visit the code of your Lambda function (that's in the index.mjs), and check that the function readTopxDataFromDatabase (it is at the top) specifies the table to query by reading the variable `process.env.TABLENAME`. Let's configure it with the name of the table that we have created:

1.  Click on the tab `Configuration`
2.  On the left, select `Environment variables`
3.  Click on `Edit`
4.  Click on `Add environment variable`. You will see two fields, one for Key, and another one for Value
5.  For `Key`, input `TABLENAME`
6.  For `Value`, input the name that you used to define your DynamoDB table (`<prefix>GameTopX`)
7.  Click on the button `Save`

___



### Task: Configuring events and testing your Lambda function



Before moving forward, let's be sure that our lambda function is working properly. As we are intending to integrate the Lambda Function with an (to be created) API Gateway component, let's create some events that will help us to check if our code is correctly implemented.

1.  If you are not already on it, get back to your Lambda function.
    
2.  Select the tab `Test`.
    
3.  Make sure that `Create new event` is selected.
    
4.  On the field `Event name`, input and appropriate name (for the first one, the suggestion is _DirectIntegration_).
    
5.  Leave everything else as it is.
    
6.  Copy-and-paste the following JSON to the body of the event, replacing what is in there
    
    ```
    {
      "params": {
        "path": {},
        "querystring": {
          "sessionId": "TheTestSession"
        }
      }
    }
    ```
    
7.  Click on the button `Save` at the top of this section.
    
8.  Click on the `Test` button. You should see a green box entitled `Execution result: succeeded(logs)`. - Expand the Details section. You should to get the response below:
    
    ```
     {
     "isBase64Encoded": false,
     "statusCode": 200,
     "body": "[{\"Nickname\":\"alanis\",\"Position\":1,\"Performance\":4.032258064516129},{\"Nickname\":\"blanka\",\"Position\":2,\"Performance\":3.3707865168539324},{\"Nickname\":\"bugsbunny\",\"Position\":3,\"Performance\":2.272727272727273},{\"Nickname\":\"billythekid\",\"Position\":4,\"Performance\":4.239130434782608},{\"Nickname\":\"nobodyknows\",\"Position\":5,\"Performance\":3.823529411764706},{\"Nickname\":\"mordorlord\",\"Position\":6,\"Performance\":5},{\"Nickname\":\"naruto\",\"Position\":7,\"Performance\":0},{\"Nickname\":\"ramon\",\"Position\":8,\"Performance\":0},{\"Nickname\":\"bruceb\",\"Position\":9,\"Performance\":0},{\"Nickname\":\"hackz\",\"Position\":10,\"Performance\":-1}]",
         "headers": {
             "Content-Type": "application/json"
         }
     }
    ```
    

**Repeat the steps 1 to 8 above**, for the following test event configurations. Be sure of selecting `Create new event` for every instance otherwise you will be rewriting always the same record. Observe the results for every new test event created:

| Test event name | body of the event |
| --- | --- |
| DirectMissingSession | `{ "params": { "path": {}, "querystring": {} } }` |
| DirectWrongSession | `{"params": {"path": {}, "querystring": {"sessionId": "WRONG"} } }` |
| ProxyIntegration | `{ "queryStringParameters": { "sessionId": "TheTestSession" } }` |
| ProxyWrongSession | `{ "queryStringParameters": { "sessionId": "WRONG" } }` |
| ProxyMissingSession | `{ "queryStringParameters": {} }` |

If at this point everything went well, then we have our Lambda function working properly. Now, let's expose it via an API.



### Task: Creating your API Gateway to consume the lambda function



#### Step 1 - Creating the API



1.  Visit the [API Gateway console page](https://console.aws.amazon.com/apigateway/home) .
    
2.  Depending on the state of your account, you can find a `Create API` or a `Get Started` button. Click on the one that has appeared to you. You are going to be taken to a Create API page.
    
    1.  Click on `Create API`.
    2.  For the section `Choose an API type`, select `REST API`, and click on `Build`. You are going to be taken to the API creation page.
    3.  In the `Create new API` section, select `New API`.
    4.  For the section `Settings`, input the following:
        1.  For `API name*`, input `<prefix>GameAPI`
        2.  For `Description`, input `API for the Game environment`
        3.  For `Endpoint Type`, select `Regional`
        4.  Click on the button `Create API`

You are going to be taken to a page where you can configure the resources of your API.



#### Step 2 - Creating the resource for your API



1.  On the Resources page, click on `Create resource` A section entitled _Resource details_ will be shown
    
2.  On the section `Resource details`
    
    1.  Leave the dropdown box named `Resource path` as it is.
    2.  For `Resource Name`, input `topxstatistics`.
    3.  Mark the box in front of the field `CORS (Cross Origin Resource Sharing)`.
    4.  Click on `Create Resource`. You will see that the Resources section of the page will be updated with this new resource, with the `OPTIONS` method already created under that resource.



#### Step 3 - Creating the GET method

1.  Select the resource topxstatistics.
2.  On the section `Methods`, click on `Create method`. You are going to be taken to a page where you will configure a new method for this resource.
3.  On the drop-down list `Method type`, select `GET`.
4.  For `Integration type`, select `Lambda function`.
5.  On the `Lambda function` section, make sure that the region is the region where you have created your function, and on the search box at its right side start inputting the name of the Lambda Function that you have created. Select the function.
6.  Make sure that `Lambda proxy integration` is turned off.
7.  Scroll down and create on `Create method`.
8.  A section like the one below is going to be shown to you: ![](Build%20Your%20First%20Microservice/apigateway.configuration.png)



#### Step 4 - Configuring the method request settings



1.  Click on `Method Request`, or scroll down to its respective tab.
2.  Click on the `Edit` button. You are going to be taken to the `Edit method request` page.
3.  Find the section `URL Query String Parameters`, expand it, and add a new query string:
    1.  Click on `Add query string`.
    2.  For the field `Name`, input `sessionId`.
    3.  Mark the `Required` checkbox.
    4.  Scroll down and click on `Save`.



#### Step 5 - Configuring the integration request



1.  Click on `Integration Request`, or scroll down to its respective tab.
2.  Click on the `Edit` button. You are going to be taken to the `Edit integration request` page.
3.  For `Request body passthrough` select `When there are no templates defined (recommended)`.
4.  Expand the `URL Query String Parameters` section, and add a new parameter:
    1.  For `Name` input `sessionId`.
    2.  For `Mapped from` input `method.request.querystring.sessionId`.
5.  Expand the `Mapping templates` section:
    1.  Click on the `Add mapping template` button.
    2.  For `Content type` input `application/json` (you need to type it)
    3.  On the `Generate template` drop-down list, select `Method Request passthrough`. API Gateway will add, automatically, a mapping template that forwards the request to the Lambda function in a structured way.
6.  Scroll down and click on the `Save` button.



#### Step 6 - Testing your API



Here we are going to test the API that we just defined. We are going to execute 3 tests.



##### Step 6.1 - Testing with empty data



1.  Making sure that you have selected the `GET` method under `/topxstatistics`, go to the `Test` tab (it is the last one)
2.  You will see a page where you can configure the query string that you have defined. ![](Build%20Your%20First%20Microservice/apigateway.configuration.testpage.png)
3.  Leave it blank, and click on the `Test` button. Check the result
4.  Click on the `Test` button with the lightning symbol, down on the page. As we have provided no sessionId field yet, your Lambda function must respond accordingly. You are going to see that the status is 200, but your Lambda function is responding with a 404 statusCode in the response body.

![](Build%20Your%20First%20Microservice/apigateway.test.400.png)

5.  In a different tab of your browser, open the [AWS Lambda page](https://console.aws.amazon.com/lambda/home) , visit your Lambda function, then select the `Monitor` tab, and then click on `View logs in CloudWatch`. Select and explore the log stream. Compare it with the results of your previous tests.



##### Step 6.2 - Testing with a invalid sessionId



1.  Still at the testing page, on the Query Strings section, input the following value to the field `{topxstatistics}  ` : `sessionId=WRONG`
2.  Click on `Test`.
3.  You should get a 200 response, but see that your Lambda function is responding accordingly with 404
    
    ```
    {
      "isBase64Encoded": false,
      "statusCode": 404,
      "body": "Inexistent session",
      "headers": {
        "Content-Type": "text/plain"
      }
    }
    ```
    



##### Step 6.3 - Testing with a valid session



1.  Still at the testing page, on the `Query Strings` section, input the following value to the field `{topxstatistics}`: `sessionId=TheTestSession`
    
    This is the same value that we have used to insert data on DynamoDB. Check on that previous step.
    
2.  Click on `Test`.
    
3.  Now you should receive a response similar to the following one for the response body:
    
    ```
    <span>{"isBase64Encoded":false,"statusCode":200,"body":"[{\"Nickname\":\"alanis\",\"Position\":1,\"Performance\":4.032258064516129},{\"Nickname\":\"blanka\",\"Position\":2,\"Performance\":3.3707865168539324},{\"Nickname\":\"bugsbunny\",\"Position\":3,\"Performance\":2.272727272727273},{\"Nickname\":\"billythekid\",\"Position\":4,\"Performance\":4.239130434782608},{\"Nickname\":\"nobodyknows\",\"Position\":5,\"Performance\":3.823529411764706},{\"Nickname\":\"mordorlord\",\"Position\":6,\"Performance\":5},{\"Nickname\":\"naruto\",\"Position\":7,\"Performance\":0},{\"Nickname\":\"ramon\",\"Position\":8,\"Performance\":0},{\"Nickname\":\"bruceb\",\"Position\":9,\"Performance\":0},{\"Nickname\":\"hackz\",\"Position\":10,\"Performance\":-1}]","headers":{"Content-Type":"application/json"}}</span>
    ```
    

Check that the status code is 200, and that the body contains the expected result.

Let's now improve the way this information is returned to the requester.



#### Step 7 - Fixing the Integration Response

1.  Click on `Integration Response`. You are going to see that there is a `Default - Response` configured for the `Method response status code` = 200. Let's edit it.
2.  Click on `Edit`.
3.  Scroll down and expand the section `Mapping Templates`, and expand it. We are going to configure a Mapping Template that converts the response from String to a JSON.
4.  If there is a record labeled `application/json` under the section `Content-Type`, click on it. **ONLY** If **there is NOT that record**, add it by following these stes:
    1.  Scroll down and click on `Add mapping template`.
    2.  For `Content-Type`, input `application/json`.
5.  On the `Template body` filed, innput the following content:
    
    ```
    #set($inputRoot = $input.path('$'))
    ## The next line changes the HTTP response code with the one provided by the Lambda Function
    #set($context.responseOverride.status = $inputRoot.statusCode)
    ## Decoding base64 (this could have been left to the application)
    #if( $inputRoot.isBase64Encoded == true )
    $util.base64Decode($inputRoot.body)
    #else
    $inputRoot.body
    #end
    ```
    
6.  Scroll down and click on the `Save` button
7.  Test the API (run again the tests on Step 6.2)

Hopefully you are going to see that the responses from your Lambda Function were properly mapped to the corresponding fields of the response.



#### Step 8 - Deploy the API



Your need to deploy the API so you can have access to it, externally.

1.  On the top of the page, click on `Deploy API`. The Deploy API pop-up window will be shown to you.
2.  On the Deploy API window:
    1.  For Deployment Stage, select `[New Stage]`.
    2.  For `Stage name*`, input the value `prod`.
    3.  For `Deployment description`, input `first deployment`.
    4.  Click on the button `Deploy`. You are going to be forwarded to the Stage page of the console, where the your deployment URL is shown. It will be something similar to the following:
        
        ```
        <span>Invoke URL: https://&lt;API-id&gt;.execute-api.&lt;region&gt;.amazonaws.com/prod</span>
        ```
    
3.  At the left, you will see the section Stages, and prod under it. Click on `prod` to expand the section. You will see the `topxstatistics` resource, with GET and OPTIONS under it.



#### Step 9 - Test the API from outside

1.  Click on the `GET` method under the `topxstatistics` resource. Copy the Invoke URL for the resource
2.  Open a new tab or window on your browser.
3.  Using the invoke URL for topxstatistics that you have just copied (let's say it is `https://xyz.execute-api.region1.amazonaws.com/prod/topxstatistics`), use each one of the querystrings shown below to test the results. **IMPORTANT**: Depending if you are creating a new API or just creating a new resource in an existing API, you may need to fix the path to the resource. Visit you API deployment on the API Gateway console, and check the path to the resources topxstatistics.

-   ... topxstatistics?sessionId=
-   ... topxstatistics?sessionId=INVALID
-   ... topxstatistics?sessionId=TheTestSession

Check the results of the Network log of your browser. Check that the HTTP return codes are appropriate.



#### Step 9 - Adding security to your API

One problem remains to your API: it is publicly exposed. There are [many options](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-control-access-to-api.html)  to implement security for your API. In this lab, we are going to implement access control via [API KEY](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-usage-plans.html) . This kind of access is recommended for Business-to-Business scenarios, when you have a key shared exclusively with the business who wants to consume your API. The key acts as the identity token for the requester, and you **must** have in addition to it an IAM-based authorization or a [Lambda Authorizer](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html)  configured for each resource of your API. Get back to your API page and let's work on this.

1.  Enabling the security via API Key
    
    1.  On the menu on the left of the API page, under the name of your API, click the option `Resources`.
    2.  Click on the `GET` method for /topxstatistics.
    3.  Click on `Method Request`, and then on `Edit`.
    4.  Mark the checkbox `API Key Required`.
    5.  Scroll down and click on `Save`. ![](Build%20Your%20First%20Microservice/apigateway.apikey.true.png)
2.  Creating an API Key
    
    1.  Now, click on `API Keys` on the menu at the left-hand side of the console (this is NOT under your API. It is a "standalone" menu option).
    2.  Click on `Create API key`.
        1.  For `Name`, input your own name.
        2.  For the `Description` field, input `This is the key that I've created for myself`.
        3.  For the field `API key`, select `Auto Generate`.
        4.  Click on `Save`. You are going to be presented to a page where you have the details about the key.
    3.  Click on the `Show` label which is at the right hand side of `API key`
    4.  Copy the value to a helper text file. We will need it later to test the API.
3.  Creating the [Usage Plan](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-usage-plans.html)  to be used by the API.  
    Think of a Usage Plan as a mechanism to control the way a certain authorized entity (represented by the API Key) can access certain resources of your API.
    
    1.  On the menu on the left, click on `Usage Plans` (it is right above the API Keys option).
    2.  Click on the button `Create usage plan`. You are going to be taken to the `Create Usage Plan` page
        1.  For `Name`, input Usage plan for `<your name> usage plan`. (replace `<your name>` with.... your name!)
        2.  For `Description`, input `This usage plan was created to learn about API Keys and usage plans`.
        3.  Be sure that `Throttling` is unchecked (we are not testing throttling on this lab).
        4.  Be sure that `Quota` is unchecked (we are not testing quota validation on this lab).
        5.  Click on the `Create usage plan` button.
    3.  Now click on the name of the usage plan that you have just created.
    4.  Now you must be on the page `Associate stages`:
        1.  Click on `Add stage`.
        2.  For the field `API`, select your API. With this you see that one Usage Plan can be associated with multiple APIs and stages.
        3.  For `Stage`, select `prod`.
        4.  Click on `Add to usage plan`.
    5.  Select the tab `Associated API keys`.
        1.  Click on `Add API key`.
        2.  Select `Add existing key`. Yes, you are going to see that we could have created the API key here. Yes, that's true.
        3.  For `API key` select the API key that you have created.
        4.  Click on the button `Add API key`.
    
    Now your API Key is created, and associated to a Usage Plan, what makes it available to be used. Let's test it.
    
4.  Testing your API
    
    **First of all, make sure that you have re-deployed your API**. Give it some time to update (1 or 2 minutes max), then test it on the browser. You should get `{"message":"Forbidden"}`.
    
    To test the publicly exposed API with the API key we are going to use CURL. CURL is available on Cloud9 (see how to create an environment [here](https://docs.aws.amazon.com/cloud9/latest/user-guide/tutorial-create-environment.html) ) or on CloudShell (click on the shell symbol at the right of the search field on your AWS console page) if you are using any of them. If using your own machine and you don't have CURL, you can download and install it from [here](https://curl.haxx.se/) .
    
    After installing it, run the following commands on your terminal (or console prompt, depending on how you name it).
    
    While running the tests below, be sure of:
    
    -   Having copied the API key in full. If a character is left over, it will not work
        
    -   Be sure of using the full path for your API. topxstatistics must be at the end (like [https://xyz.execute-api.region1.amazonaws.com/prod/topxstatistics](https://xyz.execute-api.region1.amazonaws.com/prod/topxstatistics) )
        
    -   If you get an error message like _"no matches found: "_, try to fix it by putting the URL between single quotes.
        
    
    1.  Testing without any parameter (resulting in HTTP 404, "Inexistent session")
        
        ```shell
        curl --verbose --header "x-api-key: <put your API key here>" https://<API URL>
        ```
        
    2.  Testing with an empty session (resulting in HTTP 404, "Inexistent session")
        
        ```shell
        curl --verbose --header "x-api-key: <put your API key here>" https://<API URL>?sessionId=
        ```
        
    3.  Testing with a non-existing session (resulting in HTTP 404, "Inexistent session")
        
        ```shell
        curl --verbose --header "x-api-key: <put your API key here>" https://<API URL>?sessionId=INEXISTENT
        ```
        
    4.  Testing with the session that we have inserted on DynamoDB (resulting in HTTP 200)
        
        ```shell
        curl --verbose --header "x-api-key: <put your API key here>" https://<API URL>?sessionId=TheTestSession
        ```
        



### Can we simplify the things?

Get back to the Lambda integration, and make it a proxy integration. Test it again. What do you see? Why?



### Finishing the lab

You have finished the lab.

If you are using an account provided by your AWS or Partner instructor, you don't need to do anything else.

If you are using your own account, you can clear it by:

-   Deleting the DynamoDB table.
-   Deleting the IAM role associated to the Lambda function.
-   Deleting the Lambda function.
-   Deleting the Usage Plan.
-   Deleting the API Key.
-   Deleting the API Gateway.

___

**Lab Author:** Fabian Da Silva ([fabianmartins](https://github.com/fabianmartins) )

**Adaptation**:  [jmanteau](https://github.com/jmanteau/)
