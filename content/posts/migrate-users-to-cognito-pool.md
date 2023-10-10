+++
author = "Daniel Ancuta"
title = "Strategies to migrate users to AWS Cognito Pool"
date = "2023-06-01"
description = "Strategies to migrate users to AWS Cognito Pool"
tags= ["aws", "aws cognito"]
+++

If you work in AWS ecosystem, sooner or later you will deal with [AWS Cognito](https://aws.amazon.com/cognito/). 

AWS Cognito is meant to help you with customer's identity and access management.

So you've decided to use AWS Cognito for the first time, or you used it already. In both cases you might want to migrate your existing users
from either legacy system or one Cognito Pool to another. 

This post is not meant to go into implementation details of every scenario. That is well covered by AWS documentation.

More I want to show you possible techniques and my, subjective, advantages and disadvantages of them. 
As there is no "one-size-fits-all" solution, you might want to use single technique or combination of few.

## Migrate user Lambda trigger
That would be the most common case. You had your legacy database of users. PostgreSQL, MySQL, MSSQL, MongoDB, Cognito Pool, it doesn't matter.

You have decided that you're migrating your database of users to Cognito. But at this same time you want to:
1. keep your application up and running;
2. make whole migration as transparent for end user as possible;
3. allow users to use their old password;
4. you don't really have to migrate ALL users in one go

AWS Cognito has a concept of [Migrate user Lambda trigger](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-migrate-user.html) which might fit perfectly in that case. 

How AWS documentation describes its functionality?
> When a user doesn't exist in the user pool at sign-in with a password, or in the forgot-password flow, Amazon Cognito invokes this trigger. After the Lambda function returns successfully, Amazon Cognito creates the user in the user pool. F

Let us see that on a very simple diagram:
{{< mermaid >}}
graph TD
    A[1. User] --> B[2. AWS Cognito User Pool]
    B --> C[3. Migrate User Lambda Trigger]
    C --> D["4. Lambda Function (Data Migration)"]
    D --> F[5. Successful Migration]
    F --> G[6. User Authentication]
    D --> H[7. Failed Migration]
    H --> I[8. Deny Access]
{{< /mermaid >}}

1. User starts whole flow by initialising "Sign in" or "Forgot password" flow
2. Cognito User Pool triggers migration flow
3. Trigger invokes your migration lambda
4. Migration lambda is executed. That's where you validate your user and its password

Let us talk a little bit more about point `4.` and `5.`.

### 4. Lambda Function (Data Migration)
That's where whole magic happens! 

It's place where you validate your user's identity with legacy database, check if user's password is complaint with Cognito Pool rules set by you.

If you successfully validated your user (identity, password requirements), you can decide about next steps in the flow.

### 5. Successful Migration
Your migration lambda can return back values for `finalUserStatus`, `messageAction`, `desiredDeliveryMediums`, `forceAliasCreation`, `enableSMSMFA`. Which shape how the flow for your user will look like.

Please familiarise yourself with current documentation on [Migrate user response parameters](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-migrate-user.html#cognito-user-pools-lambda-trigger-syntax-user-migration-response) section.

Based on your response you will decide if user will be `CONFIRMED` and will be able to sign in, or still will have to change password, or if you will send invitation message (set by `messageAction` attribute), which you rather don't want to do if you want ot make whole process transparent.

### Advantages & Disadvantages
| Advantages                                | Disadvantages                                          |
|-------------------------------------------|--------------------------------------------------------|
| &#128077; transparent for users           | &#128078; not all users are migrated at this same time |
| &#128077; user is able to use old password |                                                        |

{{< alert >}}
**Keep in mind**
{{< /alert >}}
1. Make sure to set `messageAction` to `SUPPRESS`, otherwise your newly imported users will get "Welcome" email.
2. When using migration lambda Cognito will not validate password against it being complaint with Cognito Pool.  It's something you need to do on your own in your lambda.

## Create users with SDK

Another option is to use SDK and create users on your own. I will be using Python's SDK, [boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html), when it comes to code snippets.
But I assume SDKs in other languages will behave same way. 

In this particular approach it's up to you how you will fetch users from your legacy database, and it's up to you to handle calls to AWS through SDK.

There are two strategies (that comes to my mind) with SDK that you can think of, both described below.

### Create user and send temporary password
This strategy requires you to send temporary password to user after their account is created. This temporary password they can use for first "Sign in" and change it.

You can create such user with this code:
```python
import boto3

cognito_idp_client = boto3.client("cognito-idp")

# Specify the User Pool ID
user_pool_id = "PUT OGNITO POOL ID HERE"


def fetch_users():
    yield {
        "username": "whisller@gmail.com",
    }


for user in fetch_users():
    username = user["username"]
    temporary_password = "Change it to some good temporary password compliant with your Cognito User Pool"

    # Create the user with a temporary password
    cognito_idp_client.admin_create_user(
        UserPoolId=user_pool_id,
        Username=username,
        TemporaryPassword=temporary_password,
        MessageAction="SUPPRESS"  # This suppresses the welcome email
    )
```

That's it. Your user is created. In Cognito it will look like:
![Cognito User](/img/migrate-users-to-cognito-pool/1.png)

Obviously in real code you would have to add few extra things like:
1. Error handling, call to `admin_create_user` can fail and throw exception
2. If you have plenty of users then it might take ages to loop through it this way
3. You will have to deliver temporary password to your user

{{< alert >}}
**Keep in mind**
{{< /alert >}}
1. Users with `FORCE_CHANGE_PASSWORD` confirmation status **WILL NOT** be able to start [Forgot Password](https://repost.aws/knowledge-center/cognito-forgot-password) flow. That's why it's important that you send them temporary password.
2. In this snippet I haven't set `email_verified` nor `phone_number_verified`, set those accordingly to your business logic

### Advantages & Disadvantages
| Advantages                                           | Disadvantages                                       |
|------------------------------------------------------|-----------------------------------------------------|
| &#128077; you can import all users at this same time | &#128078; user is unable to use old password        |
| &#128077; simple implementation                      | &#128078; user unable to use "Forgot Password" flow |

### Create user with `CONFIRMED` confirmation status
Another variation of SDK import is creating a user with `CONFIRMED` confirmation status. You might want to do that in two cases:
1. You are able to set password from legacy database
2. You want to have users in `CONFIRMED` confirmation status, as that will let them use "Forgot Password" flow.

First case is rather simple, you have (in fact you shouldn't) access to user's unencrypted password and you're able to set it in new pool.

Other case is slightly more complex, you don't have access to user's unencrypted password, you also don't really want to send them temporary one, but you want them to be able to use "Forgot Password" and reset it themselves.
Sounds complicated? Trust me, sometimes business requirements can be crazy ;)

You can create such user with this code:
```python
import boto3

cognito_idp_client = boto3.client("cognito-idp")

# Specify the User Pool ID
user_pool_id = "PUT OGNITO POOL ID HERE"


def fetch_users():
    yield {
        "username": "whisller@gmail.com",
    }


for user in fetch_users():
    username = user["username"]
    temporary_password = "Change it to some good temporary password compliant with your Cognito User Pool"

    # Create the user with a temporary password
    cognito_idp_client.admin_create_user(
        UserPoolId=user_pool_id,
        Username=username,
        TemporaryPassword=temporary_password,
        MessageAction="SUPPRESS"  # This suppresses the welcome email
    )

    cognito_idp_client.admin_set_user_password(
        UserPoolId=user_pool_id,
        Username=username,
        Password=temporary_password,
        Permanent=True
    )
```

That's it. Your user is created. In Cognito it will look like:
![Cognito User](/img/migrate-users-to-cognito-pool/2.png)

So as you can see the only difference is the call to `admin_set_user_password`. This way we set permanent user's password, and they can use "Forgot Password" flow.

{{< alert >}}
**Keep in mind**
{{< /alert >}}

1. In this snippet I haven't set `email_verified` nor `phone_number_verified`, set those accordingly to your business logic

### Advantages & Disadvantages
| Advantages                                           | Disadvantages                                |
|------------------------------------------------------|----------------------------------------------|
| &#128077; you can import all users at this same time | &#128078; user is unable to use old password |
| &#128077; simple implementation                      |                                              |
| &#128077; user able to use "Forgot Password" flow    |                                              |

## Import users from a CSV file
AWS provides a way to import users [through a CSV file](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-using-import-tool.html?icmpid=docs_cognito_console_help_panel). 
You download CSV template from Cognito, dump all users from legacy database in that format, upload it and rest is handled by AWS.

Example of template looks like:
```csv
name,given_name,family_name,middle_name,nickname,preferred_username,profile,picture,website,email,email_verified,gender,birthdate,zoneinfo,locale,phone_number,phone_number_verified,address,updated_at,cognito:mfa_enabled,cognito:username
Daniel Ancuta,,,,,,,,,whisller@gmail.com,TRUE,,,,,,FALSE,,,,whisller@gmail.com%
```

After your import job finished successfully in Cognito it will look like:
![Cognito User](/img/migrate-users-to-cognito-pool/3.png)

### Advantages & Disadvantages
| Advantages                                        | Disadvantages                                 |
|---------------------------------------------------|-----------------------------------------------|
| &#128077; heavy duty job shifted on AWS           | &#128078; user is unable to use old password  |
| &#128077; simple implementation                   |                                               |
| &#128077; user able to use "Forgot Password" flow |                                               | 


## The end
That's it! Hope you had found something useful in this post! If your company needs some help with AWS [Get in touch]({{< ref "/contact" >}} "Get in touch").
