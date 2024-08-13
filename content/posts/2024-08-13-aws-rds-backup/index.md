+++
title = "AWS RDS Cross Region and Account Backup"
author = ["david"]
date = 2024-08-13
tags = ["AWS"]
url = "aws-rds-backup"
draft = false
summary = "Automatic backup of RDS snapshots"
+++

## Introduction {#introduction}

Although we already have RDS Automatic backups in place it is prudent to have further backups for Disaster Recovery. The objective here is to copy an automatically generated System Snapshot to another Region and then to another Account. The snapshot is copied as follows...

Account-A, Region-A -&gt; Account-A, Region-B -&gt; Account-B, Region-B

The general approach is to use EventBridge events and rules and some simple Lambda functions.


### Alternative Solutions {#alternative-solutions}

There is an official AWS [blog post](https://aws.amazon.com/blogs/storage/protecting-encrypted-amazon-rds-instances-with-cross-account-and-cross-region-backups/) with a similar solution. The main differences between that solution and the one described here are that that solution uses the AWS Backup service and the Backup account is created in the same Organization. We create a new AWS Account outside of the main Organization which allows the Account to qualify for Free Tier.


## Create a new AWS Account (Account-B) {#create-a-new-aws-account--account-b}

Create a new AWS Account that will be used for backup purposes.

Remember to setup an IAM User and MFA for both the IAM User and the Root User.


### Create a Customer Managed Key (Shared Key) {#create-a-customer-managed-key--shared-key}

From Account-B, Region-B:

-   Go to KMS
-   Create a Symmetric key with the defaults, (optionally select `Multi-Region key` to allow for the flexibility to replicate the key to other Regions if needed later).
-   Select the recently created IAM User as the Administrator.
-   Under `Other AWS accounts` add the ID of Account-A - to allow the key to be used by Account-A.


## Manually copy the Snapshot (Optional) {#manually-copy-the-snapshot--optional}

This section is optional as the objective is the automation of this process, however, it provides a quick run-through the process and uses the shared key.


### Allow use of the Customer Managed Key by an IAM User in Account-A {#allow-use-of-the-customer-managed-key-by-an-iam-user-in-account-a}

Login to Account-A.

Go to IAM and create a new policy.

The content of this policy is as given in point 4 in this AWS [blog post](https://aws.amazon.com/blogs/security/share-custom-encryption-keys-more-securely-between-accounts-by-using-aws-key-management-service/) which is reproduced here:

```js
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowUseOfTheKey",
      "Effect": "Allow",
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": [
        "<ARN-SHARED-KEY>"
      ]
    },
    {
      "Sid": "AllowAttachmentOfPersistentResources",
      "Effect": "Allow",
      "Action": [
        "kms:CreateGrant",
        "kms:ListGrants",
        "kms:RevokeGrant"
      ],
      "Resource": [
        "<ARN-SHARED-KEY>"
      ],
      "Condition": {
        "Bool": {
          "kms:GrantIsForAWSResource": true
        }
      }
    }
  ]
}
```

Next select the IAM User and attach this policy to the User under `Add permissions`


### Manually copy the Snapshot {#manually-copy-the-snapshot}

From Account-A, Region-A:

-   Go to RDS Snapshots select a snapshot and select copy.
-   Select Region-B.
-   The shared key does not display in the dropdown so the ARN must be inserted manually.
-   Copy the snapshot.

From Account A, Region-B:

-   Go to RDS and ensure that the snapshot has been copied there.
-   Select the Snapshot and `Actions`, `Share snapshot`.
-   Then add the Account-B Account ID.

From Account-B, Region-B:

-   Go to RDS and ensure that in the `Shared with me` tab the snapshot is present.
-   Select the shared key and copy the snapshot.


## Automation of Snapshot copy {#automation-of-snapshot-copy}


### Automation, Steps Outline {#automation-steps-outline}

As mentioned the approach uses EventBridge and some Lambda functions. The outline of the solution looks as follows.

1.  Account-A, Region-A:
    -   Rule which matches the system snapshot creation event
        -   propagates the event to Region-B

2.  Account-A, Region-B:
    -   Rule which matches the previously propagated event
        -   calls a Lambda function to carry out a cross region snapshot copy

3.  Account-A, Region-B:
    -   Rule which matches the completion of the cross region snapshot copy
        -   calls a Lambda function that does two things
            -   share the snapshot with Account-B
            -   emits a custom event (to Account-A, Region-B) to signal that the snapshot has been shared

4.  Account-A, Region-B:
    -   Rule which matches the previously emitted event
        -   propagates the event (cross Account) to Account-B, Region-B

5.  Account-B, Region-B:
    -   Rule which matches the previously propagated event
        -   calls a Lambda function to copy the shared snapshot


### Automation, Steps Detail {#automation-steps-detail}

Here are these steps again in more detail.


#### 1. Rule to match the system snapshot creation event and propagate the event to Region-B {#1-dot-rule-to-match-the-system-snapshot-creation-event-and-propagate-the-event-to-region-b}

Create an EventBridge rule with the following pattern:

```js
{
  "source": ["aws.rds"],
  "detail-type": ["RDS DB Snapshot Event"],
  "detail": {
    "SourceArn": [{
      "wildcard": "arn:aws:rds:<REGION-A>:<ACCOUNT-A>:snapshot:rds:<SNAPSHOT-NAME>*"
    }],
    "EventID": ["RDS-EVENT-0091"]
  }
}
```

Note:

-   The `*` wildcard will match the creation date and time.
-   The second `rds:` indicates a system snapshot.
-   A catalog of the EventIDs can be found here [AWS RDS Events](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_Events.Messages.html#USER_Events.Messages.snapshot).

Next setup the Rule Target to propagate the event to Region-B:

-   Set the `Target Type` to `EventBridge event bus`.
-   Then `Event bus in different account or Region`.
-   Add the `ARN` for the Event bus in Region-B.
-   This will automatically create the necessary permission policy.
-   More details can be found in this AWS [blog post](https://aws.amazon.com/blogs/compute/introducing-cross-region-event-routing-with-amazon-eventbridge/).


#### 2. Rule to match the previously propagated event and call a Lambda to copy the snapshot {#2-dot-rule-to-match-the-previously-propagated-event-and-call-a-lambda-to-copy-the-snapshot}

The pattern is the same as for the previous Rule. The target is a Lambda which copies the recently created snapshot to Region-B:

```python
import json
import boto3

def lambda_handler(event, context):
    print(event)

    source = event['detail']['SourceArn']
    target = event['detail']['SourceIdentifier']
    target = target.replace("rds:", "")

    rds_client = boto3.client('rds', '<TARGET-REGION>')

    response = rds_client.copy_db_snapshot(
        SourceDBSnapshotIdentifier=source,
        TargetDBSnapshotIdentifier=target,
        KmsKeyId="<ARN-SHARED-KEY>")

    print(response)
```

Note:

-   `rds:` must be removed since it is not valid for snapshot names (except for system snapshots created by AWS).
-   It may be necessary to increase the Lambda runtime from default 3 seconds to around 10 seconds.

The Lambda must have permissions:

-   to use the shared key - which is the same policy as for the IAM User setup earlier.
-   to copy the database.

Database permissions:

```js
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "rds:CopyDBSnapshot",
      "Resource": "arn:aws:rds:<REGION-A>:<ACCOUNT-A>:snapshot*"
    }
  ]
}
```


#### 3. Rule to share the snapshot and emit a custom event {#3-dot-rule-to-share-the-snapshot-and-emit-a-custom-event}

When the snapshot is copied to Region-B match the across region snapshot copy event:

```js
{
  "source": ["aws.rds"],
  "detail-type": ["RDS DB Snapshot Event"],
  "detail": {
    "SourceArn": [{
      "wildcard": "arn:aws:rds:<REGION-A>:<ACCOUNT-A>:snapshot:<SNAPSHOT-NAME>*"
    }],
    "EventID": ["RDS-EVENT-0060"]
  }
}
```

The rule target is Lambda function which does two things:

-   share the snapshot with Account-B.
-   emits a custom event (to Account-A, Region-B) to signal that the snapshot has been shared.

The Lambda function:

```python
import json
import boto3

def lambda_handler(event, context):
    print(event)

    source_arn = event['detail']['SourceArn']
    source_identifier = event['detail']['SourceIdentifier']
    rds_client = boto3.client('rds', '<REGION-B>')

    rds_client_response = rds_client.modify_db_snapshot_attribute(
        AttributeName='restore',
        DBSnapshotIdentifier=source_identifier,
        ValuesToAdd=['<ACCOUNT-B>',],
    )

    print(rds_client_response)

    events_client = boto3.client('events', '<REGION-B>')

    detail_json = {'SourceIdentifier': source_identifier, 'SourceArn': source_arn}

    events_client_response = events_client.put_events(
                Entries=[
                    {
                        'Source': 'custom.lambda',
                        'DetailType': 'Custom Lambda Snapshot Share Event',
                        'Detail': json.dumps(detail_json),
                    },
                ]
            )

    print(events_client_response)
```

The Lambda will need permissions to both share the snapshot and emit the event:

```js
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "rds:ModifyDBSnapshotAttribute"
      ],
      "Resource": [
        "arn:aws:rds:<REGION-B>:<ACCOUNT-A>:snapshot:*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "events:PutEvents"
      ],
      "Resource": [
        "arn:aws:events:<REGION-B>:<ACCOUNT-A>:event-bus/default"
      ]
    }
  ]
}
```


#### 4. Rule to matches the snapshot shared custom event and propagates that event to Account-B {#4-dot-rule-to-matches-the-snapshot-shared-custom-event-and-propagates-that-event-to-account-b}

The rule pattern looks as follows:

```js
{
  "source": ["custom.lambda"],
  "detail-type": ["Custom Lambda Snapshot Share Event"],
  "detail": {
    "SourceArn": [{
      "wildcard": "arn:aws:rds:<REGION-B>:<ACCOUNT-A>:snapshot:<SNAPSHOT-NAME>*"
    }]
  }
}
```

Next setup the Rule target to propagate the event to Account-B:

-   Set the `Target Type` to `EventBridge event bus`.
-   Then `Event bus in different account or Region`.
-   Add the `ARN` for the Event bus in Account-B, Region-B.
-   This will automatically create the necessary permission policy.

More details can be found in this AWS [User Guide](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-cross-account.html).


#### 5. From Account-B, Region-B match the custom snapshot share event and copy the snapshot there {#5-dot-from-account-b-region-b-match-the-custom-snapshot-share-event-and-copy-the-snapshot-there}

Login to Account-B and go to EventBridge in Region-B.

Add a rule with a pattern to match the previously emitted snapshot shared custom event:

```js
{
  "source": ["custom.lambda"],
  "detail-type": ["Custom Lambda Snapshot Share Event"],
  "detail": {
    "SourceArn": [{
      "wildcard": "arn:aws:rds:<REGION-B>:<ACCOUNT-A>:snapshot:<SNAPSHOT-NAME>*"
    }]
  }
}
```

The target is a Lambda similar to that in step 2:

```python
import json
import boto3

def lambda_handler(event, context):
    print(event)

    source = event['detail']['SourceArn']
    target = event['detail']['SourceIdentifier']

    rds_client = boto3.client('rds', '<REGION-B>')

    response = rds_client.copy_db_snapshot(
        SourceDBSnapshotIdentifier=source,
        TargetDBSnapshotIdentifier=target,
        KmsKeyId="<ARN-SHARED-KEY>")

    print(response)
```

The Lambda must have permissions:

-   to use the shared key - which is the same policy as for the IAM User setup earlier
-   to copy the database.

Database permissions:

```js
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "rds:CopyDBSnapshot",
            "Resource": [
                "arn:aws:rds:<REGION-B>:<ACCOUNT-A>:snapshot:*",
                "arn:aws:rds:<REGION-B>:<ACCOUNT-B>:snapshot:*"
            ]
        }
    ]
}
```


## Restore {#restore}

Remember to test that the snapshot can be restored.


## Conclusion {#conclusion}

Now each time an automatic snapshot is created by RDS the snapshot will be securely copied to another Region and Account - helping us to sleep easy ;-)
