AWS CloudFormation Notifications
================================

CloudFormation template to automate SNS notifications of specific CloudFormation events.

* [Overview](#overview)
* [How it works](#how-it-works)
* [Usage](#usage)
* [Changing the default parameters](#changing-the-default-parameters)
* [Links](#links)

## Overview

Instead of publishing your CloudFormation stack events directly to an email SNS topic, with the unwanted side-effect of getting irrelevant notifications in your inbox, you can instead use a Lambda function as an intermediary. This way, your CloudFormation stacks communicate with the Lambda function and, from there, you can do pretty much whatever you want - filter the notification messages, send emails through SNS, talk to an external service etc.

This CloudFormation template allows you to send SNS notifications (email messages) to users during CloudFormation stack creation, update, deletion etc. Only certain events (like `CREATE_COMPLETE`) are notified.

**Note about the Python file:** because the Lambda function is defined in the template itself (inline Python), there's no need to host the code anywhere. The [Python file](./cloudformation-notifications.py) is kept here just for reference - it is **not** directly used in the CloudFormation stack.

## How it works

These are the main components of the CloudFormation stack created by this template:

* `SNSTopicCloudFormation`: an SNS topic that _listens_ to CloudFormation events and forwards them to a Lambda function
* `LambdaFunction`: a Lambda function capable of sending email messages through SNS. This function works like an input filter, as described below
* `SNSTopicEmail`: an SNS topic that sends email messages to users - _called_ from the Lambda function above

Because we generally don't want to get email notifications about all possible CloudFormation events, we can tell the Lambda function to only keep the types of notifications that interest us - it filters out unwanted noise. (See `NOTIFICATION_TYPES` [below](#changing-the-default-parameters))

When you create this CloudFormation stack, it outputs the ARN associated with `SNSTopicCloudFormation`. Later on, whenever you create other CloudFormation stacks, you can use that ARN with the `--notification-arns` option in order to let that topic _listen_ to events coming from those new stacks.

## Usage

Start by opening the CloudFormation template and replacing the dummy email address. You can also add more email addresses if you want:

```YAML
Resources:
  # SNS topic to send emails to users (used inside Lambda function)
  SNSTopicEmail:
    Type: "AWS::SNS::Topic"
    Properties:
      Subscription:
        - Endpoint: "my-real-email-address@example.com"
          Protocol: "email"
        - Endpoint: "sivuca@jazz.com.br"
          Protocol: "email"
```

### Create stack

Create the CloudFormation stack with the following command:

```SHELL
$ aws cloudformation create-stack \
--stack-name cloudformation-notifications \
--template-body file://cloudformation-notifications.yaml \
--capabilities CAPABILITY_IAM
```

If you are not familiar with the `--capabilities` parameter, you can find more information about it [here](https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_CreateStack.html#API_CreateStack_RequestParameters).

After creating the stack, you should get an email message asking you to confirm your subscription to the email SNS topic (`SNSTopicEmail`). Confirm your subscription to the SNS topic before proceeding to the next step.

### Get SNS topic's ARN

As said above, the main purpose of this stack is to create the SNS topic (`SNSTopicCloudFormation`) to be used at later stages of CloudFormation deployments. To get the ARN generated for that SNS topic, you can use this:

```SHELL
$ aws cloudformation describe-stacks \
--stack-name cloudformation-notifications \
--output text \
--query "Stacks[0].Outputs[?OutputKey == 'SNSTopicCloudFormation'].OutputValue"
```

Take note of this ARN - you will need it when creating new CloudFormation stacks. (Note that the `--query` parameter is written in [JMESPath](http://jmespath.org/))

### Testing

Now that the main stack was created, you can run the tests to see the end result. Replace `${SNS_TOPIC_ARN}` below with the ARN you saved earlier.

```SHELL
$ aws cloudformation create-stack \
--stack-name cloudformation-notifications-test-failure \
--template-body file://tests/failure.yaml \
--notification-arns ${SNS_TOPIC_ARN}
```

This test should fail and you should get an email notifying you of the `ROLLBACK_IN_PROGRESS` event.

```SHELL
$ aws cloudformation create-stack \
--stack-name cloudformation-notifications-test-success \
--template-body file://tests/success.yaml \
--parameters "ParameterKey=BucketName,ParameterValue=testing-cloud-formation-sns-$(date +%s)" \
--notification-arns ${SNS_TOPIC_ARN}
```

This test should succceed and you should get an email notifying you of the `CREATE_COMPLETE` event.

Note that no other events are sent to your inbox when you run the tests.

You can delete these two stacks after running the tests.

## Changing the default parameters

Update the `NOTIFICATION_TYPES` according to your needs (use a comma-separated value). For example:

* `ROLLBACK_IN_PROGRESS`
* `CREATE_COMPLETE,UPDATE_COMPLETE`

## Links

* For a similar solution, but without CloudFormation, check [this tutorial](https://aws.amazon.com/premiumsupport/knowledge-center/cloudformation-rollback-email)
