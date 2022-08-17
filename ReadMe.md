# AWS-KMS-POC - Proof of Concept on KMS

The purpose of this exercise is to develop a quick implementation of a lambda reading data from an encrypted queue (SQS) and write the content of the message in a table of an encrypted database (DynamoDB) with a Customer Managed Key (in KMS)

## Before you begin

With encryption at rest, DynamoDB transparently encrypts all customer data in a DynamoDB table, including its primary key and local and global secondary indexes, whenever the table is persisted to disk. (If your table has a sort key, some part of the sort keys are stored in plaintext in the table metadata.) When you access your table, DynamoDB decrypts the table data transparently. You do not need to change your applications to use or manage encrypted tables.

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
* DBAccessNotAllowedRole - Amazon DynamoDB Full Access (but not present in the key policy)
* SQSAccessRole - Amazon SQS Full Access and AWS CloudTrail ReadOnly Access
* SQSAccessNotAllowedRole - Amazon SQS Full Access and AWS CloudTrail ReadOnly Access (but not present in the key policy)
* KeyManagerRole - AWS KMS Admin (for all keys)
* KMS-POC-MsgProcessor-role - Amazon SQS Read message, Access DynamoDB Table, Write logs permission

Then we deploy :
* AllUsers : a group to give directly the permissions to the user to manage its credentials
* AssumeAllRoles : a group to allow its members to assume the different roles if MFA token is present (The members do not receive directly admin rights, they have to assume AdminRole)

To deploy the ressources, you can run the following AWS CLI commands or you can deploy it via the console
```
$ aws cloudformation deploy --template-file ./cf/roles.yml --stack-name KMS-POC-Roles --capabilities CAPABILITY_NAMED_IAM
$ aws cloudformation deploy --template-file ./cf/groups.yml --stack-name KMS-POC-Groups --capabilities CAPABILITY_NAMED_IAM
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

## KMS, SQS, Lambda and DynamoDB with CMK encryption at rest

We implement the solution using Customer managed key (CMK) in AWS KMS to be completely in control of the keys.

As we need to make reference to the keys when creating some resources, we have to create the keys first : one for SQS and one for DynamoDB. The producer of data could be different from the consumer in term of permissions.
The keys need
- to be managed by the created user assuming the role KeyManagerRole.
- to be used by the created user assuming their dedicated roles (DBAccessRole and SQSAccessRole)
- to be managed by the created user assuming the role DeployerRole.

For the DynamoDB table, we have to create a new table and add the specifications for the Server-Side Encryption with the DynamoDB dedicated key.

For the SQS, we need to define the _KmsMasterKeyId_ with the SQS dedicated key.

For the lambda, we have to attach the SQS event to it.

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
This is CMK encryption type. The key is stored in the account and is created, owned, and managed by the user. You have full control over the KMS key.
- If we assume the role DBAccessRole with the user johndoe, we can go directly to the items of the table, see the items and add some entries without issues. This is because the role has the permission to use the key for encryption.
- If we assume the role DBAccessNotAllowedRole with the user johndoe and we try to read the items of the table, we cannot see them and we cannot create new item. This is because the role is missing the permission to use the key.

#### What should be improved to be production ready
- The permission for the administration of the key is too lax
    - creating a specific role to admin the keys in the account and a group in our authentication account to assume that role
- We should block the possibility to use the role in another lamba (it's the role that has the permission to decrypt)
