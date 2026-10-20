# Creating the Lambda Function for Fetching Terms

This guide creates the Lambda function that receives a search term from API Gateway, finds it in DynamoDB and returns its definition.

**Steps:**
1. [Create an IAM role for Lambda](#1-create-an-iam-role-for-lambda)
2. [Create the function and add the code](#2-create-the-function-and-add-the-code)
3. [Test the Lambda function](#3-test-the-lambda-function)

**Before you start:** create the DynamoDB table and load its data. See [setup-dynamodb.md](setup-dynamodb.md).

## 1. Create an IAM role for Lambda

The function needs permission to read from DynamoDB and to write logs to CloudWatch.

1. Open the **IAM** console and select **Roles**.
2. Click **Create role**.
3. Under **Trusted entity type**, choose **AWS service**. Under **Use case**, choose **Lambda**, which means a Lambda function will assume this role. Click **Next**.
4. Attach these two policies:
   - **`AmazonDynamoDBReadOnlyAccess`** lets the function read DynamoDB tables (`GetItem`, `Query`, `Scan`, ...).
   - **`AWSLambdaBasicExecutionRole`** lets the function write logs to CloudWatch.
5. Click **Next**, name the role (for example, `LambdaDynamoDBAccessRole`), and click **Create role**.

> **Least privilege (optional):** `AmazonDynamoDBReadOnlyAccess` allows reading **every** table in the account. In production, replace it with an inline policy limited to this table:
> ```json
> {
>   "Version": "2012-10-17",
>   "Statement": [{
>     "Effect": "Allow",
>     "Action": ["dynamodb:Scan", "dynamodb:GetItem"],
>     "Resource": "arn:aws:dynamodb:<region>:<account-id>:table/CloudDefinitions"
>   }]
> }
> ```

## 2. Create the function and add the code

### Create the function

1. Open the **AWS Lambda** console and click **Create function**.
2. Choose **Author from scratch** and fill in:
   - **Function name:** for example, `FetchTermFromDynamoDB`
   - **Runtime:** **Python 3.12** (or newer)
3. Under **Permissions → Change default execution role**, choose **Use an existing role** and select `LambdaDynamoDBAccessRole`.
4. Click **Create function**.

### Set the table name

The code reads the table name from an environment variable, so you don't need to hardcode it.

1. Go to **Configuration → Environment variables → Edit**.
2. Click **Add environment variable**:

   | Key | Value |
   |---|---|
   | `TABLE_NAME` | `CloudDefinitions` (your table name) |

3. Click **Save**.

Or set it with the CLI:

```bash
aws lambda update-function-configuration \
  --function-name FetchTermFromDynamoDB \
  --environment "Variables={TABLE_NAME=CloudDefinitions}"
```

### Add the code

1. Open the **Code** tab.
2. Replace the contents of `lambda_function.py` with the code from [`lambda/lambda_function.py`](../lambda/lambda_function.py) in this repository.
3. Click **Deploy**.

### How the code works

**Imports and setup**

```python
import json
import os
import boto3

dynamodb = boto3.client('dynamodb')
table_name = os.environ['TABLE_NAME']
```

- `json` builds the JSON response, and `boto3` is the AWS SDK for Python.
- `boto3.client('dynamodb')` creates the DynamoDB client. It's created once, outside the handler, so warm invocations can reuse it.
- `table_name` is read from the `TABLE_NAME` environment variable. If the variable is missing, the function fails at startup with a `KeyError`, which makes the configuration problem obvious.

**CORS headers**

```python
HEADERS = {
    'Content-Type': 'application/json',
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'OPTIONS,GET',
    'Access-Control-Allow-Headers': 'Content-Type',
}
```

Every response includes these headers so the browser lets the React app, served from the Amplify domain, read the API response.

**Case-insensitive lookup**

```python
def normalize(text):
    return ' '.join(text.split()).lower()

def find_term(search):
    target = normalize(search)
    paginator = dynamodb.get_paginator('scan')
    for page in paginator.paginate(TableName=table_name):
        for item in page['Items']:
            if normalize(item['term']['S']) == target:
                return item
    return None
```

- `normalize()` trims the text, collapses repeated spaces and lowercases it. `"  AWS   KMS "` becomes `"aws kms"`.
- `find_term()` scans the table and returns the first item whose normalized `term` matches the normalized search.
- **Why `scan` and not `get_item`?** `get_item` is an exact, case-sensitive key lookup, so `aws kms` wouldn't find `AWS KMS`. Scanning lets the function compare in lowercase. The paginator handles tables larger than DynamoDB's 1 MB-per-scan limit.
- **Trade-off:** a scan reads the whole table on every request. That's fine for this dataset of 100 terms. For thousands of terms, store a lowercase key such as `term_lower`, add a global secondary index on it, and query the index instead.

**Handler**

```python
def lambda_handler(event, context):
    params = event.get('queryStringParameters') or {}
    term = params.get('term', '')
```

- `lambda_handler` is the entry point Lambda calls on every request.
- The handler reads `term` from the query string. For `?term=AWS%20KMS`, it gets `"AWS KMS"`.
- `queryStringParameters` is `None` when the URL has no query string, so `or {}` prevents a crash.

**Responses**

| Case | Status | Body |
|---|---|---|
| `term` missing or empty | `400` | `{"message": "term is required"}` |
| Term found | `200` | `{"term": "AWS KMS", "definition": "..."}` (the stored spelling) |
| Term not found | `404` | `{"message": "Term not found"}` |

## 3. Test the Lambda function

1. Make sure you clicked **Deploy** after pasting the code.
2. Click **Test**. The first time, it opens **Configure test event**.
3. Create an event named, for example, `SearchAwsKms`, with this JSON:

   ```json
   {
     "queryStringParameters": {
       "term": "aws kms"
     }
   }
   ```

4. Click **Save**, then **Test**.
5. The **Execution result** should show:

   ```json
   {
     "statusCode": 200,
     "headers": { "...": "..." },
     "body": "{\"term\": \"AWS KMS\", \"definition\": \"AWS Key Management Service (KMS) allows you to create and control the encryption keys used to encrypt your data.\"}"
   }
   ```

The test uses lowercase `aws kms` on purpose: getting the `AWS KMS` item back confirms the search ignores case.

**More test cases:**

| Event | Expected status |
|---|---|
| `{"queryStringParameters": {"term": "AWS Cognito"}}` | `200` |
| `{"queryStringParameters": {"term": "not a real term"}}` | `404` |
| `{"queryStringParameters": null}` | `400` |

Next, expose the function through API Gateway as described in the main [README](../README.md).

## Troubleshooting

| Error in the test result or CloudWatch logs | Cause / fix |
|---|---|
| `KeyError: 'TABLE_NAME'` | The environment variable isn't set. Add `TABLE_NAME` under **Configuration → Environment variables**. |
| `ResourceNotFoundException` | `TABLE_NAME` doesn't match the table name, or the table is in a different region from the function. |
| `AccessDeniedException ... dynamodb:Scan` | The execution role is missing DynamoDB read permissions. Attach `AmazonDynamoDBReadOnlyAccess`. |
| `Task timed out after 3.00 seconds` | Rare with 100 items. If it happens, raise the timeout under **Configuration → General configuration**. |
| Code changes have no effect | Click **Deploy** after editing. Changes in the console editor aren't live until you do. |
