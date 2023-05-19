# AWS Health Events Insight

AWS Health serves as the primary means to inform users of service degradation, planned modifications, and issues that may impact their resources. Teams or engineers can go to the aws health dashboard to find open events, historical events, or scheduled changes. However, they may not always have access to the AWS Health console, and managing communications across multiple accounts and regions can be difficult. Moreover, essential notifications may get lost amid a large volume of data. This solution creates a QuickSight dashboard that helps teams to visualize open events, historical events and upcoming events across all accounts,regions, and even different organizations.  

# Overview

This solution offers a centralized approach to store and analyze AWS Health events. The automation of capturing and storing events using EventBridge and Lambda reduces the effort required to access different accounts, regions and organizations. Furthermore, the use of QuickSight to visualize the consolidated AWS Health events stored in DynamoDB provides engineers with a robust tool to monitor the health of their AWS resources. 

![ALT](img/sampleHistorical.jpeg)

# Architecture

AWS Health events are generated by the AWS Health service in each account and region. These events include issues, scheduled maintenances, and account notifications. The AWS Health events are sent to the default event bus in the account and region where they are generated. An EventBridge rule is set up on the default event bus to forward the events to an organizational event health bus. This helps to centralize the events and simplify management. Another EventBridge rule is set up on the organizational event health bus to receive the events and trigger a Lambda function. The Lambda function receives the AWS Health events and stores them in DynamoDB. DynamoDB provides a scalable and reliable NoSQL database for storing the events. Amazon QuickSight is used to consume the consolidated events stored in DynamoDB and display them on a customizable dashboard. QuickSight provides a range of visualization options, such as charts, tables, and graphs, to help users analyze and monitor the AWS Health events.

 ![ALT](img/awshealtheventQS-archDiag.jpg)

# Prerequisites

1. Before you use the AWS Health APIs, you need to have a business or enterprise support plan from AWS Support.
2. Sign up for QuickSight if you have never used QuickSight in this account before. To use the forecast capability in QuickSight, sign up for the Enterprise Edition.
3. Verify Quicksight access to Amazon Athena Service. To enable, go to security and permissions under managed quicksight
4. we will use SPICE to hold data. Go to spice capacity under manage quicksight and verify you have 1GB spice.

# Deploying the solution

In this section, we will go through the steps to set up permissions for StackSets in both the central and member accounts.

1. **Control Account Setup:** The setup script provided in this repo will set up all the necessary components required to receive AWS health events from other accounts. This can be payer or any other regular account which would receive AWS Health data from all other accounts and regions. 

    1. To start, clone aws-health-events-insight repo

    `git clone https://github.com/aws-samples/aws-health-events-insight.git`

    2. Go to aws-health-events-insight directory and run ControlAccountSetup.py and provide account specific inputs.

    `cd aws-health-events-insight`

    `python3 ControlAccountSetup.py`

    **Note:** if you're running this script from your local machine, ensure that you have set up AWS credentials properly and have `boto3` , `subprocess` and `botocore.exceptions`  modules. Alternatively, you can use CloudShell, which automatically inherits the login role and have all the required modules with python3.

    3. Go to cloudformation and wait until Status changes to CREATE_COMPLETE (roughly 5-10 minutes). Once status is CREATE_COMPLETE, go to Amazon QuickSight dashboard and verify analysis. At this point, you must have atleast one event in analysis under historical tab.

    **TIP**: If cloudformation failed due to wrong parameters(such as wrong Amazon QuickSight principal etc), rerun step 2 with correct parameters, This would update failed stack.

2. **Child Account Setup:** Cloudformation template in ![src/ChildAccountStack](https://github.com/aws-samples/aws-health-events-insight/blob/main/src/ChildAccountStack) will set up all the necessary components required to send health events to management account. You can also use stacksets to deploy to multiple accounts and regions.

    1. In CloudFormation Console create a stack with new resources from the template file ![Childaccount-Stack.yaml](https://github.com/aws-samples/aws-health-events-insight/blob/main/src/ChildAccountStack/childaccount-stack.yaml) .
    2. Input the HealthBus ARN. Go to the AWS CloudFormation console and get this information from output of the stack(HealthEventDashboardStack).
    3. Launch the stack.

3. **Setup QS data refresh interval** By default Amazon QuickSight dataset will refresh every hour. you can edit this schedule to meet your need.

    1. Go to Datasets in Amazon QuickSight dashboard and add new schedule.
    2. Create/Edit refresh schedule and frequency based on your need.

4. **Testing:** Send Mockevent to test eventpipeline

    1. Go to Amazon EventBridge console and chose default event bus. (You can chose any account or region) and select send events.
    2. **Important** Put the event source and Detail Type as "awshealthtest" , otherwise the EB rule will discard mock event.
    3. Copy the json from ![MockEvent.json](https://github.com/aws-samples/aws-health-events-insight/blob/main/src/MockEvent.json) and paste it in the events field, hit send
    4. You will see the event in DynamoDB. For event to reflect in AWS analysis, make sure you refresh the Amazon QuickSight dataset.

# Performance Test

This solution has two lambds. One to put event in DynamoDB and another as part of fedrated query package. We stress test against lambda for fedrated query. This lambda must read dynamodb and retrun results to Amazon QuickSight in less than 60 secs.

Event count: 108,666 TIme to refresh: 47secs.
