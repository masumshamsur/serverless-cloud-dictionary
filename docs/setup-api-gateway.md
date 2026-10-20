# Setting Up API Gateway

This guide exposes the Lambda function as a public HTTP endpoint, `GET /get-definition?term=...`, and connects it to the React frontend.

**Steps:**
1. [Create a REST API](#1-create-a-rest-api)
2. [Integrate Lambda with Lambda proxy integration](#2-integrate-lambda-with-lambda-proxy-integration)
3. [Enable CORS](#3-enable-cors)
4. [Deploy the API](#4-deploy-the-api)
5. [Connect the React app and test](#5-connect-the-react-app-and-test)

**Before you start:** the Lambda function has to exist and pass its console test. See [setup-lambda.md](setup-lambda.md).

## 1. Create a REST API

1. Open the **API Gateway** console.
2. Click **Create API**.
3. Find **REST API** (not *REST API Private* or *HTTP API*) and click **Build**.
4. Choose **New API** and enter a name, for example `CloudDictionaryAPIGateway`. Leave **API endpoint type** as **Regional**.
5. Click **Create API**.

## 2. Integrate Lambda with Lambda proxy integration

### Create the resource

1. With the root resource `/` selected, click **Create resource**.
2. Enter `get-definition` as the **Resource name**. The resource path becomes `/get-definition`.
3. Click **Create resource**.

> 📌 The path must be exactly **`/get-definition`**, because `src/App.js` calls `${apiUrl}/get-definition`.

### Create the GET method

1. Select the `/get-definition` resource and click **Create method**.
2. Configure it:
   - **Method type:** `GET`
   - **Integration type:** **Lambda function**
   - **Lambda proxy integration:** **on**. API Gateway then passes the whole request to Lambda, including headers, query string parameters and body, and returns the Lambda's response as is. The function relies on this to read `event['queryStringParameters']` and set its own status code and headers.
   - **Lambda function:** choose the region where you created the function, then select it (for example, `FetchTermFromDynamoDB`).
3. Click **Create method**.

API Gateway automatically adds a resource-based permission that allows it to invoke the function.

## 3. Enable CORS

The React app runs on an Amplify domain, which is a different origin from the API. Browsers block such cross-origin requests unless the API allows them.

1. Select the `/get-definition` resource and click **Enable CORS**.
2. Check:
   - **Gateway responses:** `Default 4XX` and `Default 5XX`. Then even error responses from API Gateway itself include CORS headers, so the browser shows the real error instead of a misleading CORS error.
   - **Access-Control-Allow-Methods:** `GET` and `OPTIONS`.
3. Leave **Access-Control-Allow-Origin** as `*` and click **Save**.

This creates an `OPTIONS` method that answers the browser's preflight request.

> **Note:** with proxy integration, API Gateway doesn't add CORS headers to the `GET` response. The Lambda function returns them itself (the `HEADERS` dict in `lambda_function.py`). Both parts are needed.

## 4. Deploy the API

API changes don't go live until you deploy them.

1. Click **Deploy API**.
2. Under **Stage**, choose **\*New stage\*** and enter a stage name, such as `dev`.
3. Click **Deploy**.
4. Copy the **Invoke URL** shown on the stage page. For example:
   ```
   https://abc123xyz.execute-api.us-east-1.amazonaws.com/dev
   ```

Test it from the terminal:

```bash
curl "https://<api-id>.execute-api.<region>.amazonaws.com/dev/get-definition?term=aws%20kms"
```

Expected response:

```json
{"term": "AWS KMS", "definition": "AWS Key Management Service (KMS) allows you to create and control the encryption keys used to encrypt your data."}
```

> **Redeploy after every change.** After you change a method, an integration or CORS, click **Deploy API** again and pick the same stage. Otherwise the change has no effect on the Invoke URL.

## 5. Connect the React app and test

### Update the API URL

Open `src/App.js` and set `apiUrl` to your Invoke URL, including the stage name and without a trailing slash:

```javascript
const apiUrl = 'https://<api-id>.execute-api.<region>.amazonaws.com/dev';  // Replace with your API Gateway URL
```

### Push the change to GitHub

```bash
git add src/App.js
git commit -m "Integrate API Gateway with React frontend"
git push origin main
```

### Wait for Amplify to redeploy

1. Amplify detects the push and starts a new build automatically. This takes a few minutes.
2. Follow its progress in the **Amplify console** under your app's **Deployments**.

### Test the app

1. Open your Amplify URL, for example `https://main.<app-id>.amplifyapp.com`.
2. Search for a term from the dataset, such as `AWS KMS`.
3. The definition card should appear.

Search ignores case and extra spaces, so `aws kms`, `Aws Kms` and `  AWS   KMS ` all return the same result. You can only find terms that exist in the DynamoDB table, which means the ones loaded from `records/`.

🎉 **The Serverless Cloud Dictionary is complete.**

### Ideas for next steps

- Grow the dictionary with more records.
- Add partial-match or prefix search, such as suggestions as the user types.
- Show a "Term not found" message in the UI. Right now, `App.js` only logs errors to the browser console.
- For a much larger dataset, replace the table scan with a `term_lower` global secondary index.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Browser console: `blocked by CORS policy` | CORS isn't enabled on `/get-definition`, the API wasn't redeployed after enabling it, or the Lambda response is missing the `Access-Control-Allow-Origin` header. |
| `{"message":"Missing Authentication Token"}` | Wrong path, a typo in `/get-definition`, or the stage name is missing from the URL. |
| `{"message":"Internal server error"}` (502) | The Lambda raised an error or returned an invalid response. Check the function's CloudWatch logs. A common cause is a missing `TABLE_NAME` variable. |
| `event['queryStringParameters']` is empty or missing in Lambda | **Lambda proxy integration** is off. Turn it on in the method's integration request and redeploy. |
| Clicking **Search** does nothing | Open the browser console. Check that `apiUrl` in `App.js` is correct and includes the stage, such as `/dev`. A `404` means the term isn't in the table. |
| Changes to the API have no effect | Click **Deploy API** again for the same stage. |
