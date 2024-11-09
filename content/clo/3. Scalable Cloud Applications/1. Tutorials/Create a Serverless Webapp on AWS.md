+++
title = "Create a Serverless Webapp on AWS"
weight = 3
date = 2024-05-27
draft = false
+++

## Introduction

This tutorial guides you through creating a serverless web application on AWS that displays "Hello World" greetings in different languages. The application uses AWS S3 for hosting the front-end, AWS Lambda and API Gateway for backend processing and DynamoDB to store the greetings. We'll walk through each step, ensuring you verify the functionality at each stage.

## Method

1. Develop a simple web page.
2. Set up S3 for static website hosting.
3. Create a basic Lambda function.
4. Create API Gateway as a trigger.
5. Develop the web page to show the Lambda response.
    - Enable CORS
6. Add a DynamoDB table.
7. Update the Lambda function to read from the DynamoDB table.
    - Add IAM permissions

## Prerequisites

- An AWS account. If you don't have one, sign up [here](https://aws.amazon.com/).
- Basic familiarity with the AWS Management Console, AWS CLI, Python, HTML, and JavaScript.

## Step 1: Develop a Simple Web Page

1. **Create an `index.html` file with the following content:**

    > `index.html`

    ```html
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>Hello World!</title>
    </head>
    <body>
        <h1>Hello, World!</h1>
    </body>
    </html>
    ```

2.  **Verify**

    Use your web browser and go to `file://<path>/index.html`


## Step 2: Set Up S3 for Static Website Hosting

1. **Create an S3 Bucket**
    - Navigate to the S3 service in the AWS Management Console.
    - Click "Create bucket"
        - Bucket name: `greetings<date><time>` (The bucket name must be unique)
        - Click "Create bucket"

2. **Upload the Web Page**
    - Upload the `index.html` file to your S3 bucket.


3. **Enable Static Website Hosting**
    - Go to the properties tab
        - Enable "Static website hosting" in the bucket properties.
        - Index document: `index.html`
        - Press "Save changes"
    - Go to the Permissions tab
        - Uncheck "Block public access" in the bucket permissions.
        - Press "Save changes"
        - Add Bucket Policy (Change the bucket name!)
            ```json
            {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Sid": "PublicAccessGetObject",
                        "Principal": "*",
                        "Effect": "Allow",
                        "Action": [
                            "s3:GetObject"
                        ],
                        "Resource": [
                            "arn:aws:s3:::<bucket_name>/*"
                        ]
                    }
                ]
            }
            ```
        - Press "Save changes"

3. **Verify**
    - Access the bucket URL (provided in the static website hosting settings) to ensure your web page loads.

## Step 3: Create a Basic Lambda Function

1. **Create a Lambda Function**
    - Navigate to the Lambda service in the AWS Management Console.
    - Click "Create function" and choose "Author from scratch".
        - Function name: Greetings
        - Runtime: Python
        - Expand _Change default execution role_ and note the role created (`Greetings-role-<number>`)
    - The Python code for your Lambda function should look like this:

    > `lambda_function.py`

    ```python
    import json

    def lambda_handler(event, context):
        # TODO implement
        return {
            'statusCode': 200,
            'body': json.dumps('Hello from Lambda!')
        }
    ```

2. **Verify**
    - Use the "Test" feature in the Lambda console to ensure it returns the correct response.
        - Press "Test"
            - Test event action: Create new event
            - Event name: Test
            - Press "Save"
        - Press "Test" (again)
            - Verify the response
                ```
                Response
                {
                "statusCode": 200,
                "body": "\"Hello from Lambda!\""
                }
                ```

## Step 4: Create API Gateway as a Trigger

1. **Create an API Gateway**
    - Press "+ Add trigger" in the Function Overview Diagram
    - Choose the API Gateway service.
        - Create a new REST API.
        - Security: Open
        - Press "Add"

2. **Verify**
    1.  Browse to the API Gateway URL to ensure it triggers your Lambda function and returns the greeting.
    2. Navigate to the API Gateway service and select the newly created API
        - Go to the Test tab
            - Method: GET
            - Press "Test"
        - Check the Status and Response body

## Step 5: Develop the Web Page to Show the Lambda Response

1. **Update the `index.html` to use the API Gateway URL**

    > `index.html`

    ```html
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>Hello World!</title>
    </head>
    <body>
        <h1>Hello, World!</h1>

        <p id="greeting">Loading...</p>
        <script>
            async function fetchGreeting() {
                const response = await fetch('https://<API_URL>');
                const responseBody = await response.text();
                document.getElementById('greeting').innerText = responseBody;
            }
            fetchGreeting();
        </script>
        
    </body>
    </html>
    ```

2. **Verify**

    - Run index.html in your browser.
    - Inspect the request using the browser development tool. Note the error that occurs.

3. **Enable CORS**

    1. Navigate to the API Gateway -> Resources (/Greetings)
        - Click on "Enable CORS"
        - Press "Save"

    2. Add headers to the lambda function response (Don´t forget to Deploy the change)

        > `lambda_function.py`

        ```python
        import json

        def lambda_handler(event, context):
            # TODO implement
            return {
                'statusCode': 200,
                'headers': {
                    "Access-Control-Allow-Origin": "*",
                    "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE, OPTIONS",
                    "Access-Control-Allow-Headers": "Content-Type, X-Amz-Date, Authorization, X-Api-Key, X-Amz-Security-Token",
                },
                'body': json.dumps('Hello from Lambda!')
            }
        ```

4. **Verify (again)**

    - Run index.html in your browser.
    - Inspect the request using the browser development tool. Note the error is gone.
    - The response "Hello from Lambda!" should show up on the page.

## Step 6: Add a DynamoDB Table

1. **Create a DynamoDB Table**
    - Navigate to the DynamoDB service and click on "Create Table"
        - Table name: `Greetings`
        - Primary key: `greeting`
        - Click on "Create Table"

2. **Add Items**
    - Select the newly created table
        - Click on "Explore table items"
        - Click on "Create item"
        - Add two new records:
            - "Hello World"
            - "Hej Världen"

3. **Verify**
    - Ensure the items are added correctly in the DynamoDB console.

## Step 7: Update Lambda Function to Read from DynamoDB

1. **Update your Lambda function to fetch greetings from DynamoDB**

    > `lambda_function.py`

    ```python
    import json
    import boto3

    # Initialize the DynamoDB client
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('Greetings')  # Replace with your DynamoDB table name

    def lambda_handler(event, context):
        # Scan the DynamoDB table
        result = table.scan()
        items = result['Items']

        return {
            'statusCode': 200,
            'headers': {
                        "Access-Control-Allow-Origin": "*",
                        "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE, OPTIONS",
                        "Access-Control-Allow-Headers": "Content-Type, X-Amz-Date, Authorization, X-Api-Key, X-Amz-Security-Token",
                    },
            'body': json.dumps(items)
        }
    ```

    Deploy the updated Lambda function.

2. **Verify**

    - Run index.html in your browser.
    - Inspect the request using the browser development tool. Note the error that occurs.

3. **Add IAM permission to the role used by the lambda function**

    - In the Lambda service go to the Configuration tab
    - Select Permissions in the left hand menu
    - Press the link to the role used by the lambda service
        - Click "Add permissions -> Attach policies"
        - Check the "AmazonDynamoDBFullAccess" policy
        - Press "Add permissions"

4. **Verify (again)**

    - Run index.html in your browser.
    - Inspect the request using the browser development tool. Note the error is gone.
    - The response `[{"greeting": "Hello World"}, {"greeting": "Hej V\u00e4rlden"}]` should show up on the page.

## Step 8: Make a small adjustment to show the JSON response as a list

1. **Update the `index.html` to use the JSON response**

    > `index.html`

    ```html
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>Hello World!</title>
    </head>
    <body>
        <h1>Hello, World!</h1>

        <ul id="greetingsList">
            <li>Loading...</li>
        </ul>
        <script>
            async function fetchGreeting() {
                const response = await fetch('<API_URL>');
                const responseBody = await response.json();
                const greetingsList = document.getElementById('greetingsList');
                greetingsList.innerHTML = ''; // Clear the loading message
                responseBody.forEach(item => {
                    const listItem = document.createElement('li');
                    listItem.textContent = item.greeting;
                    greetingsList.appendChild(listItem);
                });
            }
            fetchGreeting();
        </script>
        
    </body>
    </html>
    ```

2. **Verify**

    - Run index.html in your browser.
    - Verify that a list of greetings is shown on the page
    - Add a record in the DynamoDB table and refresh the page to see the new record presented


## Step 9: Upload the index.html to S3

1. **Upload the finished Web Page**
    - Upload the `index.html` file to your S3 bucket.

2. **Verify**
    - Access the web page hosted on S3 and verify it displays the correct greeting.


## Final Thoughts

This tutorial demonstrates a step-by-step approach to creating a serverless web application using AWS services. By following these steps, you should have a functional web app that displays greetings sourced from DynamoDB.

## Cleanup

Remove resources you no longer need to avoid unnecessary costs.

- S3
- Lambda
- API Gateway
- DynamoDB
- IAM Role

# Happy Serverless Developing on AWS! 🚀
