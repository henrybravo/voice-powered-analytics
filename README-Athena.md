# Voice Powered Analytics - Athena Lab

In this lab, we will work with Athena and Lambda. 
The goal of the lab is to use Lambda and Athena to create a solution to query data at rest in s3 and build answers for Alexa. 

## Step 1 - Double check you are running in the same region that the cloudformation was launched

In this section we will use Athena and Lambda. Please make sure as you switch between tabs, that you are still using the same region launched with the cloudformation


## Step 2 - Create a query to find the number of reinvent tweets 

We need to produce an integer for our Alexa skill. To do that we need to create a query that will return our desired count.

1. To find the last set of queries from Quicksight, go to the Athena AWS Console page, then select **History** on the top menu.
1. You can see the latest queries under the column **Query** (starting with the word 'SELECT').  You can copy these queries to a text editor to save later.  
1. We'll be running these queries in the **Query Editor**. Navigate there in the top Athena menu.  
1. Ensure that the **default** database is selected and you'll see your **tweets** table.  
1. The Athena syntax is widely compatible with Presto. You can learn more about it from our [Amazon Athena Getting Started](http://docs.aws.amazon.com/athena/latest/ug/getting-started.html) and the [Presto Docs](https://prestodb.io/docs/current/) web sites
1. Once you are happy with the value returned by your query you can move to **Step 3**, otherwise you can experiment with other query types. 
1. Let's write a new query. Hint: The Query text to find the number of #reinvent tweets is:  `SELECT COUNT(*) FROM tweets`
 
**Bonus: What other interesting SQL insights can you create in Athena?**
 
**Tweet @Henry_B_** with link to additional query insights that you captured from this data. It may be added below to our ***Voice Powered Analytics** Additional Queries Attendee Submissions*

<details>
<summary><strong>OPTIONAL - Try out a few additional queries (and Attendee Submissions).</strong></summary><p>

```SQL
--Total number of tweets
SELECT COUNT(*) FROM tweets

--Total number of tweets in last 3 hours
SELECT COUNT(*) FROM tweets WHERE created > now() - interval '3' hour

--Total number of tweets by user chadneal
SELECT COUNT(*) FROM tweets WHERE screen_name LIKE '%chadneal%'

--Total number of tweets that mention AWSreInvent
SELECT COUNT(*) FROM tweets WHERE text LIKE '%AWSreInvent%'

** Attendee Submissions **
-- Submitted by Patrick (@imapokesfan) 11/29/17:
SELECT screen_name as tweeters,text as the_tweet
from default.tweets
where text like '#imwithslalom%'
group by screen_name, text

-- Submitted by Cameron Pope (@theaboutbox) 11/29/17:
SELECT count(*) from tweets where text like '%excited%'

```
</details>

## Step 3 - Create a lambda to query Athena

In this step we will create a **Lambda function** that runs every 5 minutes. The lambda code is provided but please take the time to review the function.

1. Go to the [AWS Lambda console page](https://console.aws.amazon.com/lambda/home?region=eu-west-1#/functions)
1. Click **Create Function** 
1. We will skip using a blueprint to get started and author one from scratch. Click **Author one from scratch** 
1. Under name add **vpa_lambda_athena_poller**
1. For Runtime, select **Python 3.6**
1. Under Permissions open **Choose or create an execution role**
1. Select **Use an existing role** in Execution role
1. Under existing role, select **VPALambdaAthenaPollerRole**
1. Click **Create Function** 

<details>
<summary><strong>Watch how to create the function</strong></summary><p>

**Watch how to create the function**
![Watch how to create a function](https://github.com/awslabs/voice-powered-analytics/blob/master/media/images/Alexa_lab_lambda-create-function.gif)

</details>


### Function Code

1. For Handler, ensure that it is set to: **lambda_function.lambda_handler**
1. Select inline code and then use the code below

```Python
import boto3
import csv
import time
import os
import logging
from urllib.parse import urlparse

# Setup logger
logger = logging.getLogger()
logger.setLevel(logging.INFO)


# These ENV are expected to be defined on the lambda itself:
# vpa_athena_database, vpa_ddb_table, vpa_metric_name, vpa_athena_query, region, vpa_s3_output_location

# Responds to lambda event trigger
def lambda_handler(event, context):
    vpa_athena_query = os.environ['vpa_athena_query']
    athena_result = run_athena_query(vpa_athena_query, os.environ['vpa_athena_database'],
                                     os.environ['vpa_s3_output_location'])
    upsert_into_DDB(str.upper(os.environ['vpa_metric_name']), athena_result, context)
    logger.info("{0} reinvent tweets so far!".format(athena_result))
    return {'message': "{0} reinvent tweets so far!".format(athena_result)}


# Runs athena query, open results file at specific s3 location and returns result
def run_athena_query(query, database, s3_output_location):
    athena_client = boto3.client('athena', region_name=os.environ['region'])
    s3_client = boto3.client('s3', region_name=os.environ['region'])
    queryrunning = 0

    # Kickoff the Athena query
    response = athena_client.start_query_execution(
        QueryString=query,
        QueryExecutionContext={
            'Database': database
        },
        ResultConfiguration={
            'OutputLocation': s3_output_location
        }
    )

    # Log the query execution id
    logger.info('Execution ID: ' + response['QueryExecutionId'])

    # wait for query to finish.
    while queryrunning == 0:
        time.sleep(2)
        status = athena_client.get_query_execution(QueryExecutionId=response['QueryExecutionId'])
        results_file = status["QueryExecution"]["ResultConfiguration"]["OutputLocation"]
        if status["QueryExecution"]["Status"]["State"] != "RUNNING":
            queryrunning = 1

    # parse the s3 URL and find the bucket name and key name
    s3url = urlparse(results_file)
    s3_bucket = s3url.netloc
    s3_key = s3url.path

    # download the result from s3
    s3_client.download_file(s3_bucket, s3_key[1:], "/tmp/results.csv")

    # Parse file and update the data to DynamoDB
    # This example will only have one record per petric so always grabbing 0
    metric_value = 0
    with open("/tmp/results.csv", newline='') as f:
        reader = csv.DictReader(f)
        for row in reader:
            metric_value = row['_col0']

    os.remove("/tmp/results.csv")
    return metric_value


# Save result to DDB for fast access from Alexa/Lambda
def upsert_into_DDB(nm, value, context):
    region = os.environ['region']
    dynamodb = boto3.resource('dynamodb', region_name=region)
    table = dynamodb.Table(os.environ['vpa_ddb_table'])
    try:
        response = table.put_item(
            Item={
                'metric': nm,
                'value': value
            }
        )
        return 0
    except Exception:
        logger.error("ERROR: Failed to write metric to DDB")
        return 1

```

<details>
<summary><strong>Watch how to update the function code, execution role, and basic settings</strong></summary><p>

![Watch how to update the function](https://github.com/awslabs/voice-powered-analytics/blob/master/media/images/Alexa_lab_lambda-code-role.gif)

</details>


### Environment variables

You will need the S3 bucket name you selected from the CloudFormation template. 
If you forgot the name of your bucket you can locate the name on the output tab of the CloudFormation stack.

![S3 Bucket Name on CloudFormation Outputs Tab](https://github.com/awslabs/voice-powered-analytics/blob/master/media/images/vpa-s3-buckename.png)

1. Set the following Environment variables:

```
vpa_athena_database = tweets
vpa_ddb_table = VPA_Metrics_Table
vpa_metric_name = Reinvent Twitter Sentiment
vpa_athena_query = SELECT count(*) FROM default."tweets"
region = eu-west-1 (if running out of Ireland) or us-east-1 (if running out of Northern Virginia)
vpa_s3_output_location = s3://<your_s3_bucket_name>/poller/
```

<details>
<summary><strong>Screenshot of the Lambda env's - note use your bucket name</strong></summary><p>

![Lambda env](https://github.com/awslabs/voice-powered-analytics/blob/master/media/images/vpa-lambda-env.png)


</details>

### Basic Settings

1. Set the timeout to 2 min


### Triggers Pane

We will use a CloudWatch Event Rule created from the CloudFormation template to trigger this Lambda. 
Scroll up to the top of the screen, select the pane **Triggers**. 

1. Under the **Add trigger**, click the empty box icon, followed by **CloudWatch Events**
1. Scroll down, and under *Rule*, select **VPAEvery5Min**
1. Leave the box checked for **Enable trigger**
1. Click the **Add** button, then scroll up and click **Save** (the Lambda function)

<details>
<summary><strong>Watch how to update the trigger</strong></summary><p>

![Watch how to update the trigger](https://github.com/awslabs/voice-powered-analytics/blob/master/media/images/Alexa_lab_CWE_1.gif)

</details>


## Step 4 - Create a test event and test the Lambda

At this point we are ready to test the Lambda. Before doing that we have to create a test event. 
Lambda provides many test events, and provides for the ability to create our own event. 
In this case, we will use the standard CloudWatch Event.

### To create the test event

1. In the upper right, next to test, select **Configure test events**
1. Select **Create new test event**
1. Select **Select Amazon CloudWatch** for the event template
1. Use **VPASampleEvent** for the event name
1. Click **Create** in the bottom right of the window

<details>
<summary><strong>Watch how to configure a test event</strong></summary><p>

![Watch how to configure a test event](https://github.com/awslabs/voice-powered-analytics/blob/master/media/images/vpa-lambda-test-cwe.gif)

</details>


### Now we should test the Lambda. 

1. Click **test** in the upper right
1. Once the run has completed, click on the **Details** link to see how many reinvent tweets are stored in s3.

Note, there should be > 10,000 tweets. If you get a number lower than this please ask a lab assistant for help.

<details>
<summary><strong>Watch how to test the lambda</strong></summary><p>

![Watch how to test the lambda](https://github.com/awslabs/voice-powered-analytics/blob/master/media/images/vpa-lambda-test-run.gif)

</details>

## Start working on the Alexa skill

You are now ready to build the Alexa skill, you can get started here [Amazon Alexa Section](README-Alexav2.md)
