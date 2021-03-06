# Lambda Mode

## Vocabulary

This is a quick primer on AWS Lambda terminology:

* Lambda Function - A **named** piece of source code (Java, Node.js or Python
for now) that runs essentially in a vacuum. This has absolutely no relation to the
common programming pattern of "lambda functions". Amazon just thought the name
would be cute to use.

* Handler - A function in the above code that is the entry point to the Lambda
Function.

* Event Source - A way to trigger a single run of the Lambda Function. Current
sources include creating/deleting an object in S3, receiving a message on an
SNS topic, or (the one we're concerned with) a regular schedule provided by
CloudWatch.

## Configuration

This assumes a basic familiarity with the AWS web console. It can also be set
up using the AWS CLI if you're so inclined.

To set up aws-snapper as a Lambda Function, you will have to:
* Create a Lambda Function and provide it the aws-snapper code
* Authorize that Lambda Function to perform AWS operations on your account
* Create an event (or several) that trigger the Lambda Function

### Notification (Optional)

Create an AWS SNS Topic to allow the script to send you a status update
when it completes.

You will need to subscribe to this topic using some notification method
(probably email). For more information,
[see the AWS SNS documentation](http://docs.aws.amazon.com/sns/latest/dg/CreateTopic.html).

### Permissions

Create an AWS IAM Role that allows the AWS Lambda service to call AWS services.

(Note that an existing IAM Role for running the script on a server with
an EC2 Instance Profile will not work for AWS Lambda execution.)

**If you do not already have a Customer Managed Policy** for aws-snapper:

1. Go to the AWS IAM console.

2. Select "Policies".

3. Click "Create Policy".

4. Click "Select" next to "Create Your Own Policy".

5. Enter a Policy Name and Description (anything you like) and for the Policy
Document field, paste the contents of [iam.policy.json](iam.policy.sample). Be
sure to set your account number, EC2 regions, and SNS ARN in place of the bogus
values in the sample policy.

To create the Lambda Role:

1. Go to the AWS IAM console.

2. Select "Roles".

3. Click "Create New Role".

4. Name the Role (it can be anything, but something like "lambda-aws-snapshot"
might make it easy to identify later). Click "Next Step".

5. On the "Select Role Type" page, select "AWS Lambda" and click "Next Step".

6. On the "Attach Policy" page, select the checkbox next to the Customer
Managed Policy document you created earlier. If you would like to use
CloudWatch Logs to monitor the script, also select the checkbox next t
the AWS Managed Policy "AWSLambdaBasicExecutionRole". Click "Next Step"
7. Review the options and click "Create Role".

(Remember the Role name for later.)

### Lambda Function

Create an AWS Lambda Function that contains the aws-snapper code.

1. Go to the AWS Lambda console. (Note that this is not available in all regions
but there's no reason you have to run the script in the region it will be
backing up.)

2. Click "Create A Lambda Function" (or "Get Started Now" in the likely event
that you don't already have a Lambda Function created.)

3. On the "Select blueprint" screen, choose "Blank Function".

4. Bypass the "Configure triggers" step for now.

5. On the "Configure function" screen:

    1. Enter anything you like for the Name and Description fields.
    2. Select "Python 2.7" for the Runtime field.
    3. Paste the source code of [aws-snapper.py](aws-snapper.py) into the code
entry field.
    4. For the Handler field enter `YourFunctionName.lambda_handler` where
        YourFunctionName is the name you gave the Lambda Function.
    5. Select "Choose an existing role" for the Role field.
    6. Select the IAM Role you created above for the Existing role field.
    7. Set Memory to the minimum (128MB). Set Timeout to 30 sec. (This
        can be tuned later using execution stats from CloudWatch)
    8. The "VPC" field should be left as "No VPC," since the script will
        interact with your EBS volumes through the AWS API and not via
        the EC2 instances themselves.

6. Click "Next".

7. Review the settings and click "Create function".

You now have a function that can be triggered by other sources. The triggers
will be configured in the next step.

### Test (optional)

At this point you can run the script manually by clicking "Test". A dialog will
appear the first time you try to test the script asking for input values.

Paste a JSON document that contains the same configuration values you would
otherwise provide to the command line. A sample JSON config is
[saved in the repository](config.json). As in the Policy Document above, be
sure to specify your own values for SNS ARN and regions as appropriate.

Then click "Save and test" to run the script once. If you later need to change
the configuration for the test event (it will default to re-using the JSON you
entered the first time), click "Actions" and then "Configure test event".

### Schedule

Amazon's CloudWatch service (used primarily for monitoring AWS services)
generates regularly scheduled events using either frequency (e.g. "run this
every 45 minutes") or a cron-style schedule (e.g. "run this every Monday at
2:04 PM").

To configure an event trigger:

1. Go to the AWS Lambda console.

2. Select your Lambda Function from the list.

3. Click on the "Event sources" tab.

4. Click "Add event source".

5. Select "CloudWatch Events - Schedule" from the "Event source type".

6. Provide a name for the rule and optionally a description.

7. For the Schedule expression, enter a schedule using their syntax. A
peculiarity of the cron syntax in this system is that the day of the week field
has to be ? instead of *.  For example "cron(05 00 * * ? *)" will run at 00:05
UTC every day.

8. Choose "Enable now" or "Enable later" as suits your situation. (To toggle
this setting later, click "Enabled" or "Disabled" on the event list.)

9. Click "Submit".

Note that this will simply cause your script to be run -- it will not provide
any particular input.

### Configuration

> **NOTE**: This section is slightly deprecated. It's probably best to edit
> the DEFAULTS at the top of the script when you paste it into Lambda. The
> script will still parse a JSON event object but it's easier to change
> schedules if it just receives the default CloudWatch event.

Now that the aws-snapper script is running on a schedule, you need to specify
the inputs to the script when it is run. Confusingly, this is set in the
CloudWatch console for now.

If you are still on the Event Sources tab, you can click the "Scheduled
Event:..." entry to skip to step 4 of this section.

1. Go to the AWS CloudWatch console. (Make sure your selected region is the
same one where you created your AWS Lambda Function.)

2. Click on "Rules" under the "Events" heading.

3. Select the Rule you created in step 7 of the previous section.

4. Click "Actions" (it's in the top left) and then "Edit".

5. If the schedule you created in the previous section is Enabled, there will
be a Lambda Function listed under the Targets heading. If you set the schedule
to Disabled, you will need to select "Add target" then "Lambda function" then
select the Function name from the list.

6. Expand the "Configure input" menu.

7. Select "Constant (JSON text)" and enter your JSON configuration (on a single
line) into the provided field.

8. Click "Configure details".

9. Review the settings and click "Update rule".

Note that you can create multiple schedules (or one schedule with multiple
targets) that re-use the same Lambda Function with different configuration
values to perform different snapshot behaviors. By using unique `prefix`
settings, snapshot run will ignore the others.

### Tags

The script scans your EC2 instances and EBS volumes for specific Tags to
determine which volumes to snapshot and how to delete old snapshots.

More information about configuring tags is [in a separate document](TAGS.md).
