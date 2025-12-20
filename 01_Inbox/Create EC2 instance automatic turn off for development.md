On this time, I'm using t3.large AWS EC2 instance for install database development. It very cheap.

`That is script for lambda function`
Okey, that is update content

```python
import boto3
import json
from datetime import datetime, timedelta

# Create ec2 client
ec2_client = boto3.client('ec2')
scheduler = boto3.client('scheduler')

def lambda_handler(event, context):
    instance_id = event['detail']['requestParameters']['instancesSet']['items'][0]['instanceId']
    
    response = ec2_client.describe_instances(InstanceIds=[instance_id])
    tags = response['Reservations'][0]['Instances'][0].get('Tags', [])
    
    tag_dict = {tag['Key']: tag['Value'] for tag in tags}
    
    auto_stop = tag_dict.get('AutoStop', 'false').lower() == 'true'
    stop_after_hours = int(tag_dict.get('AutoStopAfterHours', 4)) 
    
    if not auto_stop:
        print(f"Instance {instance_id} AutoStop = false skip.")
        return {"status": "Skipped"}

    stop_time = datetime.utcnow() + timedelta(hours=stop_after_hours)
    timestamp = stop_time.strftime('%Y-%m-%dT%H:%M:%S')

    schedule_name = f"Stop-{instance_id}-{int(datetime.now().timestamp())}"
    
    try:
        scheduler.create_schedule(
            Name=schedule_name,
            ScheduleExpression=f"at({timestamp})",
            Target={
                'Arn': 'arn:aws:scheduler:::aws-sdk:ec2:stopInstances',
                'Input': json.dumps({'InstanceIds': [instance_id]}),
                'RoleArn': 'arn:aws:iam::119276067463:role/SchedulerStopEC2Role'
            },
            ActionAfterCompletion='DELETE',
            FlexibleTimeWindow={'Mode': 'OFF'}
        )
        print(f"Turn off instance {instance_id} after {stop_after_hours}h at {timestamp} UTC)")
    except Exception as e:
        print(f"Error create schedule: {str(e)}")

    return {"status": "Scheduled", "instanceId": instance_id}
```