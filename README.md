# AWS Serverless Web Application Deployment

A complete guide for deploying a serverless student data management application using AWS services.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Deployment Guide](#deployment-guide)
  - [Part 1: DynamoDB and Lambda Setup](#part-1-dynamodb-and-lambda-setup)
  - [Part 2: API Gateway and S3 Static Website](#part-2-api-gateway-and-s3-static-website)
  - [Part 3: CloudFront Distribution](#part-3-cloudfront-distribution)
- [Testing the Application](#testing-the-application)

## Architecture Overview

```
Internet
   ↓
CloudFront (HTTPS)
   ↓
S3 (Static Website)
   ↓
API Gateway
   ├── Lambda: GET <──────  DynamoDB
   └── Lambda: POST ─────>  DynamoDB
                           
``` 

**Components:**
- **CloudFront**: CDN for secure HTTPS delivery
- **S3**: Static website hosting for HTML/CSS/JS
- **API Gateway**: REST API endpoints
- **Lambda Functions**: Serverless compute for business logic
- **DynamoDB**: NoSQL database for student data

## Prerequisites

- AWS Account with appropriate permissions
- Basic understanding of AWS services
- Files included in this project:
  - `functions/getStudentsData.py`
  - `functions/insertStudentData.py`
  - `static/index.html`
  - `static/js/scripts.js`

## Project Structure

```
.
├── functions/
│   ├── getStudentsData.py      # Lambda function to retrieve students
│   └── insertStudentData.py    # Lambda function to save students
└── static/
    ├── index.html              # Frontend UI
    └── js/
        └── scripts.js          # Frontend JavaScript
```

## Deployment Guide

### Part 1: DynamoDB and Lambda Setup

#### Step 1: Create DynamoDB Table

1. Navigate to **DynamoDB** in AWS Console
2. Click **Create table**
3. Configure the table:
   - **Table name**: `studentData`
   - **Partition key**: `studentid` (String)
   - **Sort key**: _(Optional - leave empty)_
   - **Table settings**: Default settings
4. Click **Create table**

#### Step 2: Create Lambda Function for GET Operation

1. Navigate to **Lambda** in AWS Console
2. Click **Create function**
3. Configure the function:
   - **Function name**: `getStudentData`
   - **Runtime**: Python 3.12
   - **Architecture**: x86_64
4. Click **Create function**
5. In the **Code** tab, paste the contents from `functions/getStudentsData.py`
6. Update the `region_name` in the code if needed (default is `us-east-1`)
7. Click **Deploy**

**Important**: Add IAM permissions for DynamoDB access:
- Go to **Configuration** → **Permissions**
- Click on the execution role
- Add policy: `AmazonDynamoDBFullAccess` or create a custom policy

#### Step 3: Create Lambda Function for POST Operation

1. Repeat the same process to create another Lambda function
2. Configure the function:
   - **Function name**: `insertStudentData`
   - **Runtime**: Python 3.12
   - **Architecture**: x86_64
3. Paste the contents from `functions/insertStudentData.py`
4. Click **Deploy**
5. Add the same IAM permissions for DynamoDB access

### Part 2: API Gateway and S3 Static Website

#### Step 4: Create API Gateway

1. Navigate to **API Gateway** in AWS Console
2. Click **Create API** → Select **REST API**
3. Configure the API:
   - **API name**: `student-api`
   - **API endpoint type**: Edge-optimized _(allows users from around the world)_
4. Click **Create API**

#### Step 5: Create API Methods

**Create GET Method:**

1. Select **Resources** in your API
2. Click **Create Method**
3. Select **GET** from dropdown
4. Configure integration:
   - **Integration type**: Lambda function
   - **Lambda function**: Select your region and `getStudentData` function
5. Click **Create Method**

**Optional**: Test the GET method by clicking **Test** tab

**Create POST Method:**

1. Click **Create Method** again
2. Select **POST** from dropdown
3. Configure integration:
   - **Integration type**: Lambda function
   - **Lambda function**: Select your region and `insertStudentData` function
4. Click **Create Method**

#### Step 6: Deploy the API

1. Click **Deploy API** button
2. Configure deployment:
   - **Stage**: *New stage*
   - **Stage name**: `dev`
3. Click **Deploy**
4. **Copy the Invoke URL** - you'll need this for the frontend

#### Step 7: Enable CORS

1. Select **Resources** in your API
2. Click **Enable CORS**
3. Configure CORS:
   - **Access-Control-Allow-Methods**: Select **GET** and **POST**
4. Click **Save**

#### Step 8: Update Frontend with API Endpoint

1. Open `static/js/scripts.js`
2. Replace `API_ENDPOIND_PASTE_HERE` with your API Gateway Invoke URL
3. Save the file

#### Step 9: Create S3 Bucket

1. Navigate to **S3** in AWS Console
2. Click **Create bucket**
3. Configure bucket:
   - **Bucket name**: `serverless-student-app` _(must be globally unique)_
   - Keep other settings as default
4. Click **Create bucket**

#### Step 10: Upload Files to S3

1. Open your bucket
2. Click **Upload**
3. Upload the following files:
   - `static/index.html`
   - `static/js/scripts.js`
4. Click **Upload**

**Note**: Maintain the folder structure: `js/scripts.js`

#### Step 11: Enable Static Website Hosting

1. Go to **Properties** tab in your bucket
2. Scroll to **Static website hosting**
3. Click **Edit**
4. Configure:
   - **Static website hosting**: Enable
   - **Index document**: `index.html`
5. Click **Save changes**
6. Note the **Bucket website endpoint** URL

#### Step 12: Make S3 Bucket Public (Temporary)

1. Go to **Permissions** tab
2. Click **Edit** on **Block public access**
3. Uncheck **Block all public access**
4. Click **Save changes** and confirm

#### Step 13: Add S3 Bucket Policy

1. Still in **Permissions** tab, scroll to **Bucket policy**
2. Click **Edit** and add this policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
    }
  ]
}
```

**Important**: Replace `YOUR-BUCKET-NAME` with your actual bucket name

3. Click **Save changes**

Now you can access your website via HTTP, but it's not secure yet.

### Part 3: CloudFront Distribution

#### Step 14: Create CloudFront Distribution

1. Navigate to **CloudFront** in AWS Console
2. Click **Create distribution**
3. Configure **Origin** settings:
   - **Origin domain**: Select your S3 bucket from dropdown
   - **Origin access**: Origin access control settings (recommended)
   - **Origin access control**: Click **Create new OAC**
   - Accept the default OAC settings
4. Configure **Web Application Firewall (WAF)**:
   - Select: **Do not enable security protections**
5. Configure **Settings**:
   - **Default root object**: `index.html`
6. Click **Create distribution**

#### Step 15: Update S3 Bucket Policy for CloudFront

After creating the distribution, you'll see a notice that the S3 bucket policy needs to be updated.

1. **Copy the policy** provided by CloudFront
2. Go back to **S3** → Your bucket → **Permissions**
3. **Re-enable Block public access** (make bucket private again)
4. Edit **Bucket policy** and replace it with:

```json
{
  "Version": "2008-10-17",
  "Id": "PolicyForCloudFrontPrivateContent",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::YOUR-ACCOUNT-ID:distribution/YOUR-DISTRIBUTION-ID"
        }
      }
    }
  ]
}
```

**Important**: Replace the following:
- `YOUR-BUCKET-NAME` with your S3 bucket name
- `YOUR-ACCOUNT-ID` with your AWS account ID
- `YOUR-DISTRIBUTION-ID` with your CloudFront distribution ID

5. Click **Save changes**

#### Step 16: Access Your Application

1. Go back to **CloudFront** → **Distributions**
2. Wait for the distribution status to change from **Deploying** to **Enabled** (this may take 5-10 minutes)
3. Copy the **Distribution domain name** (e.g., `d1234abcd.cloudfront.net`)
4. Open it in your browser

Your application is now live with HTTPS security!

## Testing the Application

1. Open your CloudFront distribution URL in a browser
2. Fill in the student information:
   - Student ID
   - Name
   - Class
   - Age
3. Click **Save Student Data** to add a record
4. Click **View all Students** to retrieve and display all records

## Troubleshooting

### 403 Forbidden Error
- Ensure S3 bucket policy is correctly configured
- Verify CloudFront has proper access to S3 bucket
- Check that files are uploaded with correct paths

### CORS Errors
- Ensure CORS is enabled in API Gateway
- Deploy the API after making CORS changes
- Clear browser cache and retry

### Lambda Function Errors
- Verify IAM roles have DynamoDB permissions
- Check CloudWatch Logs for detailed error messages
- Ensure DynamoDB table name matches in Lambda code

### API Not Working
- Verify the API endpoint URL is correctly pasted in `scripts.js`
- Ensure the API is deployed to the `prod` stage
- Test Lambda functions independently in AWS Console

## Cost Considerations

This architecture uses AWS Free Tier eligible services:
- **Lambda**: 1M free requests per month
- **DynamoDB**: 25GB storage + 25 RCU/WCU
- **API Gateway**: 1M API calls per month
- **CloudFront**: 1TB data transfer out per month
- **S3**: 5GB storage + 20,000 GET requests

Monitor your usage to stay within free tier limits.

## Security Best Practices

For production deployment, consider:
- Enable WAF on CloudFront
- Add authentication to API Gateway
- Implement input validation in Lambda functions
- Use DynamoDB encryption at rest
- Enable CloudTrail for audit logging
- Use Secrets Manager for sensitive data
- Implement rate limiting on API Gateway