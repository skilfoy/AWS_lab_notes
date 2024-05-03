# Lab: Performing Real-Time Data Analysis with Amazon Kinesis

Real-time streaming data underpins many modern businesses and solutions, and one of the most useful services for handling streaming data is Amazon Kinesis. In this lab, we will build a Kinesis Data Stream, develop both a producer and consumer of events, and also deliver them to a destination using Kinesis Data Firehose.

### Creating a Kinesis Data Stream

- Create Kinesis Stream (“TelemetricStream”)
    - Assign policy: “LambdaKinesisRolePolicy”:
``` 
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Action": [
				"logs:CreateLogGroup",
				"logs:CreateLogStream",
				"logs:PutLogEvents"
			],
			"Resource": "*",
			"Effect": "Allow",
			"Sid": "CloudWatchLogsAccess"
		},
		{
			"Action": [
				"kinesis:*"
			],
			"Resource": "*",
			"Effect": "Allow",
			"Sid": "KinesisAccess"
		}
	]
```
### Develop Kinesis Producer Lambda Function

- Create Lambda Function: “produceKinesisEvents”
    - Python:
        - Goal here is to create fake geocoordinates as we might see in an incoming gps data stream.
            - Create random uuid for id
            - Create random gps coordinates
            - Run infinite loop
                - Could be dangerous…
                - But for our purposes, we’re using it to generate data for our kinesis stream.
                - Normally, we would be getting this data from a real/live data stream
            - Sleep for random number 0-1 secs 
```
import json
import boto3
import uuid
import random
import time

def lambda_handler(event, context):
    # Create a Kinesis client object using the boto3 library
    client = boto3.client('kinesis')
    
    # Continuous loop to send data indefinitely
    while True:
        # Generate data for a new record
        data = {
            "id": str(uuid.uuid4()),  # Generate a unique identifier
            "latitude": random.uniform(-90, 90),  # Generate a random latitude
            "longitude": random.uniform(0, 180)  # Generate a random longitude
        }
    
        # Send the generated data to the specified Kinesis stream
        response = client.put_record(
            StreamName="TelemetricStream",  # Name of the Kinesis data stream
            PartitionKey="geolocation",  # Partition key to determine the shard
            Data=json.dumps(data)  # Data payload as a JSON string
        )
    
        # Print the response from the Kinesis put_record call
        print(response)
        
        # Pause the loop for a random amount of time (up to 1 second)
        time.sleep(random.random())  # Introduces variability in the request rate
```
- Lambda by default has 3 second timeout
    - We want to generate more fake data, so we’ll increase timeout to 30 for testing, and later to 300 for more extensive testing

### Develop Kinesis Consumer Lambda Function

- AWS base64 encodes data, so we want to decode it and reinsert it into the data field

- We'll use the following python code:
    - 
 
```
import json
import base64

def lambda_handler(event, context):
    # Initialize an empty list to store processed records
    records = []
    
    # Loop over each record in the incoming event
    for record in event["Records"]:
        # Decode the base64 encoded data from Kinesis record and then decode from bytes to string
        data = base64.b64decode(record["kinesis"]["data"]).decode()
        # Convert the JSON formatted string into a Python dictionary and append to list
        records.append(json.loads(data))
        
    # Prepare the output dictionary with the count of records and the actual data
    output = {
        "count": str(len(records)),  # Convert number of records to string for consistent data type
        "data": records  # List of processed records
    }
    
    # Convert the output dictionary to a JSON formatted string and print it
    print(json.dumps(output))
```
### Setup Kinesis Data Firehose

- Create Firehose data stream from Kinesis data stream to firehose delivery S3 bucket
