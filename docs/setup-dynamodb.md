# Configuring DynamoDB to Store Cloud Definitions

This guide sets up the DynamoDB table that stores the dictionary's terms and definitions, then loads the dataset from the `records/` folder.

**Steps:**
1. [Create the DynamoDB table](#1-create-the-dynamodb-table)
2. [Prepare the records for the batch operation](#2-prepare-the-records-for-the-batch-operation)
3. [Upload the records with the AWS CLI](#3-upload-the-records-with-the-aws-cli)
4. [Verify the records in the table](#4-verify-the-records-in-the-table)

## 1. Create the DynamoDB table

DynamoDB is a fully managed NoSQL database from AWS with fast, predictable performance. It suits applications that need high availability, scalability and low latency, like this dictionary.

1. Log in to the **AWS Management Console**.
2. Type **DynamoDB** in the search bar and select it.
3. On the DynamoDB dashboard, click **Create table**.
4. Fill in the following:
   - **Table name:** `CloudDefinitions`
   - **Partition key:** `term`, type **String**. This key uniquely identifies each term, so the dictionary can't contain duplicates.
5. Leave the other options at their defaults (capacity mode **On-demand**) and click **Create table**.

Or create the table with the CLI:

```bash
aws dynamodb create-table \
  --table-name CloudDefinitions \
  --attribute-definitions AttributeName=term,AttributeType=S \
  --key-schema AttributeName=term,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

## 2. Prepare the records for the batch operation

The `records/` folder in the project contains four files: `records-1.json`, `records-2.json`, `records-3.json` and `records-4.json`.

### Why four files?

DynamoDB's `batch-write-item` API accepts at most **25 put or delete requests per call**. The dataset has 100 terms, so it's split into four files of 25 records each. That keeps every request within the limit.

### File format

Each file is a map whose key is the **table name** and whose value is the list of items to write. The files ship with a placeholder as the table name:

```json
{
  "<ADD_DYNAMODB_TABLE_NAME_HERE>": [
    {
      "PutRequest": {
        "Item": {
          "term": { "S": "AWS KMS" },
          "definition": { "S": "AWS Key Management Service (KMS) allows you to create and control the encryption keys used to encrypt your data." }
        }
      }
    }
  ]
}
```

- **`<ADD_DYNAMODB_TABLE_NAME_HERE>`:** replace it with your table name, such as `CloudDefinitions`.
- **`PutRequest`:** the operation, which inserts or overwrites the item.
- **`Item`:** the attributes to store. `"S"` marks a String value.

> **Important:** replace the placeholder on line 2 of **all four files** before uploading. Otherwise the upload fails with `ResourceNotFoundException`.

On macOS or Linux, you can replace it in all four files at once:

```bash
# macOS
sed -i '' 's/<ADD_DYNAMODB_TABLE_NAME_HERE>/CloudDefinitions/' records/*.json

# Linux
sed -i 's/<ADD_DYNAMODB_TABLE_NAME_HERE>/CloudDefinitions/' records/*.json
```

## 3. Upload the records with the AWS CLI

Your AWS CLI needs to be installed and configured with credentials that can write to DynamoDB. If it's already set up (`aws sts get-caller-identity` works), skip to [Run the upload](#run-the-upload).

### Install the AWS CLI

Run the command for your operating system:

```bash
# Windows (Command Prompt or PowerShell)
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi

# macOS (Homebrew)
brew install awscli

# Linux: official installer (installs AWS CLI v2)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

> `sudo apt-get install awscli` also works on Debian/Ubuntu, but it often installs the older **v1** CLI. Use the official installer above to get v2.

Check the installation:

```bash
aws --version
```

If you see a "command not found" error, restart your terminal and try again.

### Create access keys

The CLI authenticates with an **access key ID** and **secret access key** that belong to an IAM user.

1. In the AWS Management Console, search for **IAM** and open it.
2. Go to **Users** and click **Create user**.
3. Enter a user name and click **Next**.
4. Under **Permission options**, choose **Attach policies directly** and attach **AdministratorAccess**.
   > **Caution:** `AdministratorAccess` grants full access to every AWS service. Use it only on a personal learning account, and never share these keys. For a tighter setup, attach `AmazonDynamoDBFullAccess` instead, or a custom policy limited to the `CloudDefinitions` table.
5. Click **Next**, then **Create user**.
6. Open the new user, go to **Security credentials → Access keys**, and click **Create access key**.
7. Choose **Command Line Interface (CLI)** as the use case, tick the confirmation box and click **Next**.
8. Add a description, such as `cloud-dictionary-cli`, and click **Create access key**.
9. Copy the **Access key** and **Secret access key**. You can't view the secret again after you leave this page.

> Treat these keys like a password. Don't commit them to Git or paste them into code.

### Configure the CLI

```bash
aws configure
```

Enter the values when prompted:

```
AWS Access Key ID [None]: <your-access-key-id>
AWS Secret Access Key [None]: <your-secret-access-key>
Default region name [None]: us-east-1
Default output format [None]: json
```

Set the **default region** to the region where you created the DynamoDB table. If they don't match, the CLI can't find the table.

### Run the upload

From the **project root**, run:

```bash
aws dynamodb batch-write-item --request-items file://records/records-1.json
aws dynamodb batch-write-item --request-items file://records/records-2.json
aws dynamodb batch-write-item --request-items file://records/records-3.json
aws dynamodb batch-write-item --request-items file://records/records-4.json
```

Or upload all four in a loop (macOS/Linux):

```bash
for f in records/records-*.json; do
  aws dynamodb batch-write-item --request-items file://$f
done
```

### What happens here?

- Each command writes up to 25 records from its file to DynamoDB.
- A successful upload returns:
  ```json
  {
      "UnprocessedItems": {}
  }
  ```
- If `UnprocessedItems` isn't empty, DynamoDB couldn't write those items, usually because of throttling. Run the same command again for that file. `PutRequest` overwrites existing items, so re-running is safe.

## 4. Verify the records in the table

### In the console

1. Return to the **DynamoDB** dashboard.
2. Select **Tables** and open the **CloudDefinitions** table.
3. Click **Explore table items** to view the uploaded records.
4. Check that all terms and definitions are present and filled in correctly.

### With the CLI

Count the items. It should return **100**:

```bash
aws dynamodb scan --table-name CloudDefinitions --select COUNT
```

Look up a single term:

```bash
aws dynamodb get-item \
  --table-name CloudDefinitions \
  --key '{"term": {"S": "AWS KMS"}}'
```

DynamoDB configuration is complete. Next, set up the Lambda function and API Gateway as described in the main [README](../README.md).

## Troubleshooting

| Error | Cause / fix |
|---|---|
| `ResourceNotFoundException: Requested resource not found` | The table name in the JSON file doesn't match your table, or the CLI's default region is different from the table's region. |
| `Unable to load paramfile file://...: No such file or directory` | Wrong path or typo. Run the command from the project root and check the spelling (`records`, not `recordds`). Keep the whole command on one line. |
| `Unable to locate credentials` | The CLI isn't configured. Run `aws configure`. |
| `AccessDeniedException` | The IAM user's policies don't allow `dynamodb:BatchWriteItem` on the table. |
| `ValidationException: Too many items requested` | A file has more than 25 items. Split it into smaller files. |
