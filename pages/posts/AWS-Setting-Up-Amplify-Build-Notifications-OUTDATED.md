---
title: "AWS: Setting up Amplify Deployment Notifications via SNS (OUTDATED)"
date: 2025/3/10
description: How to Set Up Email Notifications for AWS Amplify Deployments Using Amazon SNS
tag: AWS, Amplify, SNS 
author: Matthew Cordaro
---

# AWS: Setting up Amplify Deployment Notifications via SNS (OUTDATED)

## THIS IS NOW OUTDATED AS THIS USES OUTDATED AMPLIFY v1 AND AWS-CLI v2

**DO NOT USE THESE DIRECTIONS!  MOVE TO V2 & V3 respectively for custom hooks!**


_How to Set Up Email Notifications for AWS Amplify Deployments Using Amazon SNS_

**AWS Amplify** simplifies the process of building, deploying, and hosting web applications. But wouldn’t it be 
great if you could get real-time email notifications for successful or failed deployments? In this guide, I’ll show 
you how to integrate **Amazon Simple Notification Service (SNS)** with **Amplify** to receive email alerts—while keeping sensitive data secure by using environment variables.

_Note: This guide assumes you have `aws-sdk` installed._

## Create an SNS Topic

Start by creating an Amazon SNS topic that will send deployment notifications.

1. Go to the _Amazon SNS_ Console
2. Click _Create topic_ and choose _Standard_ as the type
3. Give your topic a _Name_, like `AmplifyDeploymentNotifications`
4. Click _Create topic_
5. Note the _ARN_, you will need it for later.

## Subscribe Your Email Address

To receive notifications, you need to subscribe your email address to the SNS topic.

1. Navigate to your SNS Topic in the console
2. Click _Create_ subscription
3. Set the _Protocol_ to Email
4. Under _Endpoint_ enter your email address
5. Click _Create subscription_
6. Click the provided link, in the confirmation email that was sent, to confirm the subscription

## Add / Modify IAM Role for Amplify

To enable Amplify access to push notifications to the SNS topic, you will need to create a service role.

1. Navigate to **IAM Roles** and click on _Create role_
2. For _Step 1_, set _Tursted entity type_ to _AWS service_ and for _Use case_ select _Amplify_
3. Click _Next_ twice to get to _Step 3_
4. Give it a name like `AmplifyServiceRole` click _Create role_
5. Select the role you just created in the list
6. Under _Permission policies_ select the dropdown menu that says _Add permission_ and select _Create inline policy_
7. Set the editor to JSON and paste in the following
   ```json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Effect": "Allow",
               "Action": "sns:Publish",
               "Resource": "<your-arn>"
           }
       ]
   }
   ```
8. Replace `<your-arn>` with the arn from the SNS Topic you created before
9. Click _Next_
10. Navigate to your Amplify app in the AWS Amplify Console 
11. In the menu, go to _App settings > IAM Roles_
12. Set the _Service role_ to the role you just created like `AmplifyServiceRole`
13. Click _Save_

## Add an SNS Topic ARN Environment Variable to Amplify

To prevent your SNS topic ARN, from appearing in your source code on GitHub,
store it as an environment variable in Amplify.

1. Navigate to your Amplify app in the AWS Amplify Console
2. In the menu, go to _Hosting > Environment_ variables
3. Click _Manage variables_
4. Click _Add new_
5. Set the _Variable_ to `SNS_TOPIC_ARN` and _Value_ to the ARN of your SNS topic
6. Click _Save_ the environment variable.

## Create a `post-build` Hook in Amplify

AWS Amplify supports lifecycle hooks that run custom scripts after deployments. Use the post-build hook to publish messages to your SNS topic.

1. Create and navigate to the `amplify/hooks` folder in your project.
2. Create or update the `post-build.js` file with the following code:
    ```javascript
    const AWS = require('aws-sdk');
    const sns = new AWS.SNS();
    
    const run = async () => {
    const topicArn = process.env.SNS_TOPIC_ARN;
    const branchName = process.env.GIT_BRANCH;
    const buildStatus = process.env.AWS_AMPLIFY_BUILD_STATUS;
    const buildLogs = process.env.AWS_AMPLIFY_BUILD_LOGS;
    const notificationBranches = ['main', 'dev'];
    
        const logInfo = {
            timestamp: new Date().toISOString(),
            topicArn,
            branchName,
            buildStatus,
            isNotificationBranch: notificationBranches.includes(branchName)
        };
    
        if (notificationBranches.includes(branchName)) {
            let message;
            if (buildStatus === 'SUCCEED') {
                message = `✅ Successful deployment to ${branchName}!\n` +
                    `🚀 The build and deployment completed successfully.`;
            } else if (buildStatus === 'FAILED') {
                message = `❌ Failed deployment to ${branchName}!\n` +
                    `⚠️ The build or deployment encountered issues.\n\n` +
                    `Build Logs:\n${buildLogs || 'No logs available'}`;
            } else {
                console.log(JSON.stringify({
                    ...logInfo,
                    event: 'skip_notification',
                    reason: 'build_in_progress'
                }, null, 2));
                return;
            }
    
            const params = {
                Message: message,
                TopicArn: topicArn
            };
    
            try {
                await sns.publish(params).promise();
                console.log(JSON.stringify({
                    ...logInfo,
                    event: 'notification_sent',
                    message,
                    status: 'success'
                }, null, 2));
            } catch (error) {
                console.error(JSON.stringify({
                    ...logInfo,
                    event: 'notification_error',
                    error: {
                        message: error.message,
                        code: error.code,
                        statusCode: error.statusCode
                    },
                    status: 'error'
                }, null, 2));
            }
        } else {
            console.log(JSON.stringify({
                ...logInfo,
                event: 'skip_notification',
                reason: 'non_notification_branch'
            }, null, 2));
        }
    };
    
    run();
    ```
3. Note `const notificationBranches = ['main', 'dev'];`  Feel free to change this to the branches you want to get 
deploy notifications for.
4. Make sure to update your `dependencies` in `package.json` to include latest `aws-sdk`.
5. Commit and push your changes to the branch you want to deploy.

## Test

Amplify should now be executing the `post-build.js` script after any deployment is complete. Check your email inbox 
for notifications when deploying the branches you set. If you don't receive an email, double-check the SNS topic 
subscription, the SNS_TOPIC_ARN environment variable, and the script for any errors.

### Known Issues

#### Resolve Auto unsubscribe issue (Gmail)

If you're finding that you're automatically unsubscribing but don't know why, you should enable authentication to
unsubscribe. [See this AWS Post on how to do this.](https://repost.aws/knowledge-center/prevent-unsubscribe-all-sns-topic)

#### 