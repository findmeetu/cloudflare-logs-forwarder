# cloudflare-logs-forwarder

This project contains an AWS Lambda function that:

 - listens to S3 events with uploaded Cloudflare logs using Cloudflare's [logpush](https://developers.cloudflare.com/logs/logpush) service
 - compacts the json logs by cherry-picking fields of interest and producing space-delimited log lines
 - forwards the logs via HTTP to an arbitrary log aggregator
 
## Context

Cloudflare can be configured to [push the logs to an s3 bucket](https://developers.cloudflare.com/logs/logpush).
It does so every 5 minutes by uploading a gzipped log file where each line is a log entry in [json](https://developers.cloudflare.com/logs/log-fields) format.
Depending on the amount of traffic that goes via Cloudflare, the logs can be quite massive.
While storing them in S3 or S3 Glacier can be cheap, companies often want to extract some valuable data from those logs, 
build some analytics, correlate them with logs from other services, etc.
Both analytics and log centralization/aggregation is usually provided by [third party services](https://developers.cloudflare.com/logs/analytics-integrations)
that often charge for the amount of data ingested.
Even if the log aggregation is a self-hosted Elastic stack, the large amounts of data need to be stored somewhere.

This project can reduce the size of logs ingested by an aggregator by selecting only the data fields of interest 
and by stripping the json field names that add a lot of overhead given large amounts of log lines.
This can reduce the amount of ingested data by a factor of <X>.

An overhead is that the aggregator needs to know how to parse the new format. 
If Elastic stack is used, this can be done by using a simple GROK filter <add example>.
Proprietary services can also be configured to parse the custom format of incoming logs.

## Structure

This project contains source code and supporting files for a serverless application that you can deploy with the [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html).

- cloudflare-logs-forwarder-function/src/main - Code for the application's Lambda function.
- cloudflare-logs-forwarder-function/src/test - Tests for the application code. 
- template.yaml - A SAM template that defines the application's AWS resources.

The application itself is written in Java11 and built using Gradle.
The deployment uses several AWS resources, including Cloudformation, Lambda functions and an S3 bucket. 

## Prerequisites for local development and deployment

The Serverless Application Model Command Line Interface (SAM CLI) is an extension of the AWS CLI that adds functionality for building and testing Lambda applications. It uses Docker to run your functions in an Amazon Linux environment that matches Lambda. It can also emulate your application's build environment and API.

To use the SAM CLI, you need the following tools.

* SAM CLI - [Install the SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html)
* Java11 - [Install the Java 11](https://docs.aws.amazon.com/corretto/latest/corretto-11-ug/downloads-list.html)
* Docker - [Install Docker community edition](https://hub.docker.com/search/?type=edition&offering=community)

## Build and deploy

To build and deploy your application for the first time, run the following in your shell:

```bash
sam build
sam deploy --guided
```

The first command will build the source of your application. The second command will package and deploy your application to AWS, with a series of prompts:

* **Stack Name**: The name of the stack to deploy to CloudFormation. This should be unique to your account and region, and a good starting point would be something matching your project name.
* **AWS Region**: The AWS region you want to deploy your app to.
* **Confirm changes before deploy**: If set to yes, any change sets will be shown to you before execution for manual review. If set to no, the AWS SAM CLI will automatically deploy application changes.
* **Allow SAM CLI IAM role creation**: Many AWS SAM templates, including this example, create AWS IAM roles required for the AWS Lambda function(s) included to access AWS services. By default, these are scoped down to minimum required permissions. To deploy an AWS CloudFormation stack which creates or modified IAM roles, the `CAPABILITY_IAM` value for `capabilities` must be provided. If permission isn't provided through this prompt, to deploy this example you must explicitly pass `--capabilities CAPABILITY_IAM` to the `sam deploy` command.
* **Save arguments to samconfig.toml**: If set to yes, your choices will be saved to a configuration file inside the project, so that in the future you can just re-run `sam deploy` without parameters to deploy changes to your application.

You can find your API Gateway Endpoint URL in the output values displayed after deployment.

## Use the SAM CLI to build and test locally

Build your application with the `sam build` command.

```bash
sam build
```

The SAM CLI installs dependencies defined in `cloudflare-logs-lambda-function/build.gradle`, creates a deployment package, 
and saves it in the `.aws-sam/build` folder.

Test a single function by invoking it directly with a test event. An event is a JSON document that represents the input that the function receives from the event source. Test events are included in the `events` folder in this project.

Run functions locally and invoke them with the `sam local invoke` command.

```bash
cloudflare-logs-forwarder$ sam local invoke HelloWorldFunction --event events/event.json
```

## Add a resource to your application
The application template uses AWS Serverless Application Model (AWS SAM) to define application resources. 
AWS SAM is an extension of AWS CloudFormation with a simpler syntax for configuring common serverless application resources such as functions, triggers, and APIs. 
For resources not included in [the SAM specification](https://github.com/awslabs/serverless-application-model/blob/master/versions/2016-10-31.md), 
you can use standard [AWS CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html) resource types.

## Fetch, tail, and filter Lambda function logs

To simplify troubleshooting, SAM CLI has a command called `sam logs`. `sam logs` lets you fetch logs generated by your deployed Lambda function from the command line. In addition to printing the logs on the terminal, this command has several nifty features to help you quickly find the bug.

```bash
cloudflare-logs-forwarder$ sam logs -n cloudflare-logs-forwarder-function --stack-name cloudflare-logs-forwarder --tail
```

You can find more information and examples about filtering Lambda function logs in the [SAM CLI Documentation](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-logging.html).

## Cleanup

To delete the sample application that you created, use the AWS CLI. Assuming you used your project name for the stack name, you can run the following:

```bash
aws cloudformation delete-stack --stack-name cloudflare-logs-forwarder
```