+++
author = "Daniel Ancuta"
title = "Strategies to Migrate Users to AWS Cognito Pool"
date = "2023-06-01"
description = "Strategies to Migrate Users to AWS Cognito Pool"
tags = ["aws", "aws cognito", "devops"]
+++

If you work in the AWS ecosystem, sooner or later, you will deal with [AWS Cognito](https://aws.amazon.com/cognito/).

AWS Cognito is meant to help you with customer identity and access management.

So you've decided to use AWS Cognito for the first time, or you've used it already. In both cases, you might want to migrate your existing users from either a legacy system or one Cognito Pool to another.

This post is not meant to go into implementation details of every scenario. That is well covered by AWS documentation.

What I want to show you are possible techniques and my subjective advantages and disadvantages of them. As there is no "one-size-fits-all" solution, you might want to use a single technique or a combination of a few.

## Migrate User Lambda Trigger
That would be the most common case. You had your legacy database of users. PostgreSQL, MySQL, MSSQL, MongoDB, Cognito Pool, it doesn't matter.

You have decided that you're migrating your database of users to Cognito. But at the same time, you want to:
1. Keep your application up and running.
2. Make the whole migration as transparent for the end user as possible.
3. Allow users to use their old password.
4. You don't really have to migrate ALL users in one go.

AWS Cognito has a concept of [Migrate User Lambda Trigger](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-migrate-user.html) which might fit perfectly in that case.

How AWS documentation describes its functionality?
> When a user doesn't exist in the user pool at sign-in with a password or in the forgot-password flow, Amazon Cognito invokes this trigger. After the Lambda function returns successfully, Amazon Cognito creates the user in the user pool.

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

1. The user starts the whole flow by initializing the "Sign-in" or "Forgot Password" flow.
2. The Cognito User Pool triggers the migration flow.
3. The trigger invokes your migration Lambda.
4. The migration Lambda is executed. That's where you validate your user and their password.

Let's talk a little bit more about points `4.` and `5.`.

### 4. Lambda Function (Data Migration)
That's where the whole magic happens!

It's the place where you validate your user's identity with the legacy database, check if the user's password is compliant with Cognito Pool rules set by you.

If you successfully validate your user (identity, password requirements), you can decide about the next steps in the flow.

### 5. Successful Migration
Your migration Lambda can return back values for `finalUserStatus`, `messageAction`, `desiredDeliveryMediums`, `forceAliasCreation`, `enableSMSMFA`. These values shape how the flow for your user will look like.

Please familiarize yourself with the current documentation on [Migrate User Response Parameters](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-migrate-user.html#cognito-user-pools-lambda-trigger-syntax-user-migration-response) section.

Based on your response, you will decide if the user will be `CONFIRMED` and will be able to sign in, or still will have to change the password, or if you will send an invitation message (set by the `messageAction` attribute), which you'd rather not do if you want to make the whole process transparent.

### Advantages & Disadvantages
| Advantages                                | Disadvantages                                          |
|-------------------------------------------|--------------------------------------------------------|
| &#128077; transparent for users           | &#128078; not all users are migrated at this same time |
| &#128077; user is able to use old password |                                                        |

{{< alert >}}
**Keep in mind**
{{< /alert >}}
1. Make sure to set `messageAction` to `SUPPRESS`, otherwise, your newly imported users will get a "Welcome" email.
2. When using the migration Lambda, Cognito will not validate the password against it being compliant with Cognito Pool rules. It's something you need to do on your own in your Lambda.

## Create Users with SDK

Another option is to use an SDK and create users on your own. I will be using Python's SDK, [boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html), when it comes to code snippets.
But I assume SDKs in other languages will behave the same way.

In this particular approach, it's up to you how you will fetch users from your legacy database, and it's up to you to handle calls to AWS through the SDK.

There are two strategies (that come to my mind) with the SDK that you can think of, both described below.

### Create User and Send Temporary Password
This strategy requires you to send a temporary password to the user after their account is created. This temporary password they can use for the first "Sign-in" and change it.

You can create such a user with this code:

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
That's it. Your user is created. In Cognito, it will look like:
![Cognito User](/img/migrate-users-to-cognito-pool/1.png)

Obviously, in real code, you would have to add a few extra things like:
1. Error handling, the call to `admin_create_user` can fail and throw an exception.
2. If you have plenty of users, then it might take ages to loop through it this way.
3. You will have to deliver a temporary password to your user.

{{< alert >}}
**Keep in mind**
{{< /alert >}}
1. Users with `FORCE_CHANGE_PASSWORD` confirmation status **WILL NOT** be able to start the [Forgot Password](https://repost.aws/knowledge-center/cognito-forgot-password) flow. That's why it's important that you send them a temporary password.
2. In this snippet, I haven't set `email_verified` nor `phone_number_verified`. Set those accordingly to your business logic.

### Advantages & Disadvantages
| Advantages                                           | Disadvantages                                |
|------------------------------------------------------|----------------------------------------------|
| &#128077; you can import all users at the same time  | &#128078; user is unable to use the old password |
| &#128077; simple implementation                      |                                              |
| &#128077; user able to use "Forgot Password" flow    |                                              |

### Create User with `CONFIRMED` Confirmation Status
Another variation of SDK import is creating a user with `CONFIRMED` confirmation status. You might want to do that in two cases:
1. You are able to set the password from the legacy database.
2. You want to have users in `CONFIRMED` confirmation status, as that will let them use the "Forgot Password" flow.

The first case is rather simple; you have (in fact, you shouldn't) access to the user's unencrypted password and you're able to set it in the new pool.

The other case is slightly more complex; you don't have access to the user's unencrypted password, and you also don't really want to send them a temporary one, but you want them to be able to use the "Forgot Password" and reset it themselves.
Sounds complicated? Trust me, sometimes business requirements can be crazy.

You can create such a user with this code:

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
That's it. Your user is created. In Cognito, it will look like:
![Cognito User](/img/migrate-users-to-cognito-pool/2.png)

So as you can see, the only difference is the call to `admin_set_user_password`. This way we set the permanent user's password, and they can use the "Forgot Password" flow.

{{< alert >}}
**Keep in mind**
{{< /alert >}}

1. In this snippet, I haven't set `email_verified` nor `phone_number_verified`. Set those accordingly to your business logic.

### Advantages & Disadvantages
| Advantages                                           | Disadvantages                                |
|------------------------------------------------------|----------------------------------------------|
| &#128077; you can import all users at the same time  | &#128078; user is unable to use the old password |
| &#128077; simple implementation                      |                                              |
| &#128077; user able to use the "Forgot Password" flow |                                              |

## Import Users from a CSV File
AWS provides a way to import users [through a CSV file](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-using-import-tool.html?icmpid=docs_cognito_console_help_panel).
You download a CSV template from Cognito, dump all users from the legacy database in that format, upload it, and the rest is handled by AWS.

An example of the template looks like:

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
That's it! I hope you found something useful in this post! If your company needs some help with AWS, [get in touch]({{< ref "/contact" >}} "Get in touch").
