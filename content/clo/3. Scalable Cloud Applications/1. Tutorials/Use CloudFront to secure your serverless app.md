+++
title = "Serverless: Use CloudFront to Secure Your Serverless App"
weight = 6
date = 2025-11-08
draft = true
+++

# Use CloudFront to Secure Your Serverless App

## Introduction

In this guide, we will enhance the security of your serverless web application by integrating Amazon CloudFront as a Content Delivery Network (CDN). This step ensures your app is protected against common web threats and delivers content with low latency. The serverless app in question includes AWS S3 for front-end hosting, AWS Lambda and API Gateway for backend processing, and DynamoDB for data storage. We’ll secure and optimize it further with CloudFront, adding SSL/TLS encryption and a Web Application Firewall (WAF).

## Prerequisites

Before we begin, ensure you have the following components set up:

- An AWS S3 bucket hosting your static site
- Lambda functions and an API Gateway configured for backend processing
- DynamoDB table for storing user data
- AWS CLI configured on your system
- Basic familiarity with AWS management console and CLI commands

## Step 1: Set Up Amazon CloudFront Distribution

### 1.1 Create a New Distribution
1. Navigate to the **CloudFront** console in your AWS Management Console.
2. Click on **Create Distribution** and choose **Web**.
3. Under **Origin Settings**:
   - **Origin Domain**: Enter your S3 bucket URL or the endpoint of your API Gateway.
   - **Origin Protocol Policy**: Set this to **HTTPS Only** to enforce encrypted communication.
4. Configure the **Viewer Protocol Policy** as **Redirect HTTP to HTTPS** to ensure secure traffic.

### 1.2 Enable Caching and Custom Behaviors
1. Under **Default Cache Behavior Settings**, ensure the following:
   - **Allowed HTTP Methods**: Choose GET, HEAD, OPTIONS for static content and POST if your app handles form submissions.
   - **Cache Based on Selected Request Headers**: Whitelist necessary headers such as `Authorization` for secure data transfers.
2. Set **Object Caching** to control how frequently CloudFront fetches updates from your origin. Use `Use Origin Cache Headers` for dynamic content.

### 1.3 Attach a Web Application Firewall (WAF)
1. Navigate to **AWS WAF** and create a new WebACL.
2. Add rules to protect against common threats, such as SQL injection and XSS (Cross-Site Scripting).
3. Attach this WebACL to your CloudFront distribution.

## Step 2: Configure SSL/TLS Certificates

1. Navigate to **AWS Certificate Manager (ACM)**.
2. Request a public certificate for your domain and verify ownership.
3. Link the certificate with your CloudFront distribution under **Custom SSL Certificate**.

## Step 3: Update S3 Bucket and API Gateway Security

### 3.1 Secure S3 Bucket
- Ensure your S3 bucket's **Block Public Access** settings are enabled.
- Use **Bucket Policy** to restrict access so only CloudFront can access the content:
  ```json
  {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Effect": "Allow",
              "Principal": {
                  "Service": "cloudfront.amazonaws.com"
              },
              "Action": "s3:GetObject",
              "Resource": "arn:aws:s3:::your-bucket-name/*",
              "Condition": {
                  "StringEquals": {
                      "AWS:SourceArn": "arn:aws:cloudfront::your-account-id:distribution/your-distribution-id"
                  }
              }
          }
      ]
  }
  ```

### 3.2 Secure API Gateway
- Use **Resource Policies** in API Gateway to allow requests only from CloudFront.

## Step 4: Test and Verify

1. Deploy the changes and test your web app's performance and security.
2. Ensure that HTTP requests are redirected to HTTPS.
3. Verify that the WAF is actively filtering malicious traffic.

## Conclusion

Integrating CloudFront not only secures your serverless app but also improves its global performance and scalability. With these changes, your application benefits from encrypted communication, reduced latency, and protection against web vulnerabilities.