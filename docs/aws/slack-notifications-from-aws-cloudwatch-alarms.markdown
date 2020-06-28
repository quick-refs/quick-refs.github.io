---
layout: post
title: Slack notifications from AWS CloudWatch alarms
category: [aws, cloudwatch, slack]
parent: Amazon Web Services (AWS)
nav_order: 1
youtubeId: QgfMCDkVRPA
permalink: aws/slack-notifications-from-aws-cloudwatch-alarms
---

# Slack notifications from AWS CloudWatch alarms

In this tutorial, we will setup AWS CloudWatch alarms to receive a Slack notification whenever there is an error in AWS Lambda execution. Many other types of alarms can be set in CloudWatch using the available metrics for most of the AWS resources. This same process can be used to send a Slack notification for any alarm created in CloudWatch.

We will start by creating a simple Lambda function which will throw an exception when invoked, then we will create a CloudWatch Alarm which will be configured to send SNS Notification when that lambda fails. 

![slack-notification-from-aws-cloudwatch-alarms]({{ site.baseurl }}/docs/aws/images/slack-notification-from-aws-cloudwatch-alarms.png)

We will then create a slack notifier lambda function, which will subscribe to the same SNS topic and when invoked will send notification to a slack channel. 

Let's get started.

{% include youtube.html id=page.youtubeId %}

## Create Simple Lambda Function to test

First let's create simple lambda function that when invoked can throw an exception. We will use error metrics from this lambda function to setup CloudWatch alarm and send Slack notification when that happens.

Go to Lambda console in AWS

* Click "Create Function" 
* Select "Author from scratch" and give this function a name, we will call it "test-function"
* For Runtime we will use Python 3.6
* Select "Create new role with basic lambda permissions" 
* Click "Create Function"

Copy and paste this snippet to Function code in lambda_function.py and Save it.

```python 
import json
def lambda_handler(event, context):
    # TODO implement
    print(event);
    
    if "error" in event:
        raise Exception(event["error"])
    
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from Lambda!')
    }
```

This function basically, has an if condition where if the event input has "error" it will raise an Exception and lambda function execution will fail. 

On the top right corner of the page Click on the drop down for "Select a test event" and "Configure test events", update the payload to look like this:

``` python
{
  "error": "this is an error"
}
```

## Create a CloudWatch Alarm

Next, let's go ahead and create a CloudWatch Alarm to trigger when this lambda execution results in an error

Go to CloudWatch console in AWS

- Click "Alarms" at the left, and then Create Alarm.
- Click "Lambda Metrics".
- Look for your Lambda name in the listing, and click on the checkbox for the row where the metric name is "Errors". Click "Next".
- Enter a name and description for this alarm.
- Setup the alarm to be triggered whenever "Errors" is above 0, for 1 consecutive period(s).
- Select "Sum" as the Statistic and 5 minutes (or the amount of minutes that's reasonable for your use case) in the "Period" dropdown. Since we are testing keeping it at minimum and changing it back to more resonable number will be better.
- In the "Notification" box, click the Select a notification list dropdown and select your new SNS endpoint.
- Also add your email so that you get email when this sns is triggered, we will disable this later.
- Click "Create Alarm".
- Check you email and confirm SNS Subscription

Go Back to the earlier create lambda function

* Click on "Test" to run the lambda function multiple time to cause failures so that it will trigger the CloudWatch ALARM. 
* Wait for the period as you set earlier in CloudWatch alarm
* Once the CloudWatch alarm's status changes from "INSUFFICIENT" to "ALARM"  you must receive an email with the failure alert.

That's good but we wanted to receive a slack notification instead!

Let's get started on that now...

## Add incoming webhook app to Slack

Go to Administration from the dropdown in your slack workspace

* Click on "Manage apps"
*  Search for "Incoming Webhook" in App Directory

- Click on "Add to Slack"
- Choose a slack channel to post the message to or create a new one
- Scroll to the Example section and copy the curl command, then run it in your terminal window to test it.
- We will need the Webhook URL later, so save it. Also make sure you keep it secret because it contains credentials required to authenticate for posting into your slack channel.

## Create Slack Notifier Lambda Function

Similar to before, create a New Lambda functionand name it "slack-notifier". Copy and paste or type the following code in the lambda_function.py:

```python
import json
from urllib.parse import urlencode
from urllib.request import Request, urlopen


def lambda_handler(event, context):
    webhook_url = 'https://hooks.slack.com/services/TRARKKGHX/BRsadf8ATCAJW/ygOFNsadfzAMEX0SuTUmdtzCONwm' # Set destination URL here
    slack_data = {
        'channel': 'cloudwatch-alerts',
        'text': 'this is a simple test'
    }

    request = Request(
        webhook_url, 
        data=json.dumps(slack_data).encode(),
        headers={'Content-Type': 'application/json'}
        )
    response = urlopen(request)
    return {
        'statusCode': response.getcode(),
        'body': response.read().decode()
    }


if __name__ == "__main__":
    print(lambda_handler(None, None))

```
This Lambda Function now has a simple Python Script to send POST request to the webhook url we got from the slack incoming webhook app.

Update the webhook_url and slack channel, in ths function to match yours.

Click on "Test" to execute this function. You must have received a slack notification.


## Add SNS as lambda trigger 

So far, we have a CloudWatch alarm that is sending notifications to SNS and we also have a Lambda function which when invoked sends message to slack channel.

Our next task, is to invoke this slack-notifier lambda by SNS. Let's do that now.

In the slack-notifier lambda that we just created, go to the top of the page 

* Click on "+ Add trigger" and select the same SNS Topic that was created when creating the CloudWatch Alarm
* Now, if we go to SNS console in AWS, we must see that we have Email and a lambda subscribed to the SNS topic. 
* Let's run our "test-function" lambda function to create few failed invocations like before.
* After the set period it should now, send both email and slack notification to the channel.

## Slack message format

The Final step to to format our slack message so that we have enough details about the CloudWatch alarm.

We need to know what type of event payload does the lambda function get upon when invoked by SNS. The easiest way to do that is to:

Go to SNS console in AWS

* Select the SNS topic

- Add another subscription to the sns topic with Protocol set as Email-JSON.
- Confirm the subscription in email
- Again, run the "test-function" lambda to cause errors 
- Wait for the Alarm to send u an email.
- The email will have the JSON payload the lambda function would receive. Based on this JSON we will update the "slack-notifier" function to format the slack message.

First let's create a JSON payload to test the execution for "slack-notifier" lambda function

Go to Lambda console in AWS

* Select the "slack-notifier" lambda function that we created earlier
* Click on the dropdown for "Configure test events"
* In the Event template select "Amazon SNS Topic Notification". You must see the JSON payload for SNS notification
* Replace the "Sns" field with the JSON message we received in the email and save it.

Finally, Copy and paste this script, which will take care of formatting the slack message appropriately for a CloudWatch alarm. 

```python
import json
from urllib.parse import urlencode
from urllib.request import Request, urlopen
from datetime import datetime
import boto3

session = boto3.session.Session()


class CloudWatchAlarmParser:
    def __init__(self, msg):
        self.msg = msg
        self.timestamp_format = "%Y-%m-%dT%H:%M:%S.%f%z"
        self.trigger = msg["Trigger"]

        if self.msg['NewStateValue'] == "ALARM":
            self.color = "danger"
        elif self.msg['NewStateValue'] == "OK":
            self.color = "good"

    def __url(self):
        return ("https://console.aws.amazon.com/cloudwatch/home?"
                + urlencode({'region': session.region_name})
                + "#alarmsV2:alarm/"
                + self.msg["AlarmName"]
                )

    def slack_data(self):
        _message = {
            'text': '<!here|here>',  # add @here to message
            'attachments': [
                {
                    'title': ":aws: AWS CloudWatch Notification :alarm:",
                    'ts': datetime.strptime(
                            self.msg['StateChangeTime'],
                            self.timestamp_format
                          ).timestamp(),
                    'color': self.color,
                    'fields': [
                        {
                            "title": "Alarm Name",
                            "value": self.msg["AlarmName"],
                            "short": True
                        },
                        {
                            "title": "Alarm Description",
                            "value": self.msg["AlarmDescription"],
                            "short": False
                        },
                        {
                            "title": "Trigger",
                            "value": " ".join([
                                self.trigger["Statistic"],
                                self.trigger["MetricName"],
                                self.trigger["ComparisonOperator"],
                                str(self.trigger["Threshold"]),
                                "for",
                                str(self.trigger["EvaluationPeriods"]),
                                "period(s) of",
                                str(self.trigger["Period"]),
                                "seconds."
                            ]),
                            "short": False
                        },
                        {
                            'title': 'Old State',
                            'value': self.msg["OldStateValue"],
                            "short": True
                        },
                        {
                            'title': 'Current State',
                            'value': self.msg["NewStateValue"],
                            'short': True
                        },
                        {
                            'title': 'Link to Alarm',
                            'value': self.__url(),
                            'short': False
                        }
                    ]
                }
            ]
        }
        return _message



def lambda_handler(event, context):
    sns_message = json.loads(event['Records'][0]['Sns']['Message'])
    print(sns_message)

    webhook_url = 'https://hooks.slack.com/services/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' # Set destination URL here

    slack_data = CloudWatchAlarmParser(sns_message).slack_data()
    slack_data["channel"] = 'cloudwatch-alerts'


    request = Request(
        webhook_url, 
        data=json.dumps(slack_data).encode(),
        headers={'Content-Type': 'application/json'}
        )
    response = urlopen(request)
    return {
        'statusCode': response.getcode(),
        'body': response.read().decode()
    }


if __name__ == "__main__":
    print(lambda_handler(None, None))


```

Save it and Click on "Test" to run test. You should have received a nicely formatted slack message, using the test event.

For the actual event, 

Go back to the "test-function" lambda

* Click on "Test" to generate few errors again.

* Wait for slack notification to show up! That's it.

## What next?

To further optimize it, you can change the slack channel and webhook url to be passed as environment variable to the lambda function. I will leave that for you to explore. There is lambda blueprint for cloudwatch alarm to slack in python, which can be used as reference. 
