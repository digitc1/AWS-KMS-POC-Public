# AWS-KMS-POC - Proof of Concept on KMS

The purpose of this exercise is to develop a quick implementation of a lambda reading data from an encrypted queue (SQS) and write the content of the message in a table of an encrypted database (DynamoDB) with a Customer Managed Key (in KMS)

## Before you begin

With encryption at rest, DynamoDB transparently encrypts all customer data in a DynamoDB table, including its primary key and local and global secondary indexes, whenever the table is persisted to disk. (If your table has a sort key, some of the sort keys that mark range boundaries are stored in plaintext in the table metadata.) When you access your table, DynamoDB decrypts the table data transparently. You do not need to change your applications to use or manage encrypted tables.

Encryption at rest also protects DynamoDB streams, global tables, and backups whenever these objects are saved to durable media.

Concerning the encryption on SQS, SSE encrypts messages as soon as Amazon SQS receives them. The messages are stored in encrypted form and Amazon SQS decrypts messages only when they are sent to an authorized consumer.

**SSE encrypts the body of a message in an Amazon SQS queue.**

SSE doesn't encrypt the following:
* Queue metadata (queue name and attributes)
* Message metadata (message ID, timestamp, and attributes)
* Per-queue metrics

Here are some interesting links :

https://docs.aws.amazon.com/whitepapers/latest/introduction-aws-security/data-encryption.html

https://d1.awsstatic.com/events/reinvent/2019/REPEAT_2_Deep_dive_into_AWS_KMS_SEC322-R2.pdf

https://docs.aws.amazon.com/kms/latest/developerguide/services-dynamodb.html

https://aws.amazon.com/blogs/database/bring-your-own-encryption-keys-to-amazon-dynamodb/

https://docs.aws.amazon.com/kms/latest/developerguide/create-keys.html

AWS services that are integrated with AWS KMS use symmetric KMS keys to encrypt your data.

https://aws.amazon.com/kms/features/#AWS_Service_Integration

https://d0.awsstatic.com/whitepapers/aws-kms-best-practices.pdf

https://docs.aws.amazon.com/whitepapers/latest/logical-separation/logical-separation.pdf

https://aws.amazon.com/kms/faqs/


## Necessary roles and groups

First, we deploy the following roles :
* AdminRole - Full admin permission
* DeployerRole - Full admin permission
* DBAccessRole - Amazon DynamoDB Full Access
* DBAccessNotAllowedRole - Amazon DynamoDB Full Access
* SQSAccessRole - Amazon SQS Full Access and AWS CloudTrail ReadOnly Access
* SQSAccessNotAllowedRole - Amazon SQS Full Access and AWS CloudTrail ReadOnly Access
* KeyManagerRole - AWS KMS Admin 

Then we deploy :
* AllUsers : a group to give directly the permissions to the user to manage its credentials
* AssumeAllRoles : a group to allow a member of the group to assume the different roles if MFA token is present (The members do not receive directly admin rights, they have to assume it)

To deploy the ressources, you can run the following AWS CLI commands or you can deploy it via the console
```
$ aws cloudformation deploy --template-file ./cf/roles.yaml --stack-name KMS-POC-Roles --capabilities CAPABILITY_NAMED_IAM
$ aws cloudformation deploy --template-file ./cf/groups.yaml --stack-name KMS-POC-Groups --capabilities CAPABILITY_NAMED_IAM
```
Remark : the parameter --profile WHATEVER_PROFILE could be used

## User creation

We connect to the console and we create an IAM user (e.g. johndoe) with MFA and we add the user to the 2 groups.

Now, we will create a profile for AWS CLI.
In the file credentials in .aws directory, we will add a new section for the Access key of the recently created user :
```
[johndoe_creds]
aws_access_key_id = AKIAZ...
aws_secret_access_key = ****************************************
```
In the file config, we will add a new PROFILE authenticated in the account with MFA and assuming the DeployerRole:
```
[profile MY_PROFILE]
role_arn = arn:aws:iam::999999999999:role/DeployerRole
mfa_serial = arn:aws:iam::999999999999:mfa/johndoe
source_profile=johndoe_creds
region = eu-west-1
```
999999999999 is your account id.

## SQS, Lambda and DynamoDB with CMK encryption at rest

We implemet the solution using Customer managed key (CMK) in AWS KMS.
We want to create 2 keys : one for SQS and one for DynamoDB. The producer of data is different from the consumer.
For the DynamoDB table, we have to create a new table and give the specifications for the Server-Side Encryption for a DB dedicated key.
For the CMK encryption, we need to create a KMS symetric key.

To deploy the ressources you can run the following commands in this order or via the console
```
$ aws cloudformation deploy --template-file ./cf/keys.yaml --stack-name KMS-POC-Keys --capabilities CAPABILITY_NAMED_IAM --parameters ParameterKey=IAMUserId,ParameterValue=johndoe --profile MY_PROFILE
$ aws cloudformation deploy --template-file ./cf/queue.yaml --stack-name KMS-POC-Queue --capabilities CAPABILITY_NAMED_IAM --profile MY_PROFILE
$ aws cloudformation deploy --template-file ./cf/db.yaml --stack-name KMS-POC-Database --capabilities CAPABILITY_NAMED_IAM --profile MY_PROFILE
$ aws cloudformation deploy --template-file ./cf/lambda.yaml --stack-name KMS-POC-Lambda --capabilities CAPABILITY_NAMED_IAM --profile MY_PROFILE
```

#### Observations
- When looking at the table in the DynamoDB service in the console,
we see that encryption is based on the KMS key we just created.
This is CMK encryption type. The key is stored in your account and is created, owned, and managed by you. You have full control over the KMS key.
- We can go directly to the Items and add some entries without issues.

#### What we have to do to make it work
- in the key policy, allow the lambda to decrypt with the key for reading the table.
- in the key policy, allow a user to encrypt/decrypt for inserting items and reading them in the console.
- in the key policy, allow all principals from the account with the necessary permissions (role/policy) to administer the key.

#### What should be improved to be production ready
- The permission for the administration of the key is too lax
    - creating a specific role to admin the keys in the account and a group in our authentication account to assume that role
- We should block the possibility to use the role in another lamba (it's the role that has the permission to decrypt)

### Lambda, DynamoDB and SQS Queue with CMK encryption at rest

You can find this implementation in the directory ReadFromQueueWithEncryption.
At the level of the Lambda, we add the code for reading the records from the queue, the permission to acces the queue and a trigger for the lambda based on the event on the queue.
For the queue it-self, we have to create an SQS Queue with a reference to the KMS Key.

To deploy the ressources you can run the following commands in this order or via the console (it replaces the previous version of the lambda)
```
$ aws cloudformation deploy --template-file ./ReadFromQueueWithEncryption/lambda.yaml --stack-name KMS-POC-Lambda --capabilities CAPABILITY_NAMED_IAM
$ aws cloudformation deploy --template-file ./ReadFromQueueWithEncryption/queue.yaml --stack-name KMS-POC-Queue --capabilities CAPABILITY_NAMED_IAM
```
Remark : the parameter --profile YOUR_PROFILE could be used

#### Remarks
- We have used the same KMS key for the encryption of the dynamoDB table and the SQS Queue, depending on the requirement this, we could have to create another KMS Key dedicated to SQS Queue.
- The producers of the messages in the Queue could be different from the consumers, this has to be managed at the level of the policy of the SQS Queue and at the level of the KMS Key policy.