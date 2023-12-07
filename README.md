# How to set up the infrastructure

Steps:
- Create an AWS account
- Enable Cloud Trail for logging operations
- Create DNS settings
- Create generic a HTTPS certificate
- Deploy VPC
- Deploy Authentication
- Deploy the API Gateway
- Deploy Miscellaneous tools
- Setup secrets
- Deploy Slack noticifications
- Deploy a Postgres DB
- Deploy a backend service as a container
- Deploy a backend service as serverless
- Deploy a frontend

## Create an AWS account

Assuming you already have an AWS account.

It's a very simple and straightfoward way of handling service-level perimissions in different SDLC stages to deploy the related environments into different AWS accounts.

This can easily achived without additional costs by using the `AWS Organizations > AWS Accounts > Add an AWS account` service on the AWS console.

It's highly recommended to not to use the newly added account with root privileges. Adding a user to your root account that has enforced MFA and that can switch to your new account, but can't mess up things in the root account is a good idea in general.

Once the newly added user has the MFA set and working it's time to add restrictive policies to have to use it every time.

Policies for requiring MFA:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "EnableUsersManageTheirPassword",
            "Effect": "Allow",
            "Action": [
                "iam:ChangePassword",
                "iam:GetAccountPasswordPolicy"
            ],
            "Resource": "arn:aws:iam::123456789012:user/${aws:username}"
        },
        {
            "Sid": "EnableUsersManageTheirAccessKeys",
            "Effect": "Allow",
            "Action": [
                "iam:CreateAccessKey",
                "iam:ListAccessKeys",
                "iam:DeleteAccessKey"
            ],
            "Resource": "arn:aws:iam::123456789012:user/${aws:username}"
        },
        {
            "Sid": "EnableUsersManageTheirLoginProfiles",
            "Effect": "Allow",
            "Action": [
                "iam:CreateLoginProfile",
                "iam:GetLoginProfile",
                "iam:DeleteLoginProfile",
                "iam:UpdateLoginProfile"
            ],
            "Resource": "arn:aws:iam::123456789012:user/${aws:username}"
        },
        {
            "Sid": "EnableUsersManageTheirSigningCertificates",
            "Effect": "Allow",
            "Action": [
                "iam:ListSigningCertificates",
                "iam:DeleteSigningCertificate",
                "iam:UpdateSigningCertificate",
                "iam:UploadSigningCertificate"
            ],
            "Resource": "arn:aws:iam::123456789012:user/${aws:username}"
        },
        {
            "Sid": "EnableUsersManageTheirSSHPublicKeys",
            "Effect": "Allow",
            "Action": [
                "iam:ListSSHPublicKeys",
                "iam:GetSSHPublicKey",
                "iam:DeleteSSHPublicKey",
                "iam:UpdateSSHPublicKey",
                "iam:UploadSSHPublicKey"
            ],
            "Resource": "arn:aws:iam::123456789012:user/${aws:username}"
        },
        {
            "Sid": "EnableUsersManageTheirMFA",
            "Effect": "Allow",
            "Action": [
                "iam:ListMFADevices",
                "iam:CreateVirtualMFADevice",
                "iam:DeleteVirtualMFADevice",
                "iam:EnableMFADevice",
                "iam:ResyncMFADevice"
            ],
            "Resource": [
                "arn:aws:iam::123456789012:user/${aws:username}",
                "arn:aws:iam::123456789012:mfa/${aws:username}"
            ]
        },
        {
            "Sid": "EnableUsersBasicIAMOperations",
            "Effect": "Allow",
            "Action": [
                "iam:ListAccountAliases",
                "iam:ListUsers",
                "iam:ListServiceSpecificCredentials",
                "iam:GetAccountSummary"
            ],
            "Resource": "arn:aws:iam::123456789012:*"
        },
        {
            "Sid": "EnableUsersBasicSTSOperations",
            "Effect": "Allow",
            "Action": [
                "sts:GetSessionToken"
            ],
            "Resource": "arn:aws:sts::123456789012:*"
        },
        {
            "Sid": "DenyAccessWithoutMFA",
            "Effect": "Deny",
            "NotAction": [
                "iam:ChangePassword",
                "iam:GetAccountPasswordPolicy",
                "iam:CreateAccessKey",
                "iam:DeleteAccessKey",
                "iam:CreateLoginProfile",
                "iam:GetLoginProfile",
                "iam:DeleteLoginProfile",
                "iam:UpdateLoginProfile",
                "iam:ListSigningCertificates",
                "iam:DeleteSigningCertificate",
                "iam:DeleteSSHPublicKey",
                "iam:ListMFADevices",
                "iam:CreateVirtualMFADevice",
                "iam:DeleteVirtualMFADevice",
                "iam:EnableMFADevice",
                "iam:ResyncMFADevice"
            ],
            "Resource": "*",
            "Condition": {
                "BoolIfExists": {
                    "aws:MultiFactorAuthPresent": "false"
                }
            }
        }
    ]
}
```

Where `123456789012` is the number of your root account.


Policies for switching to the newly added account:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::210987654321:role/OrganizationAccountAccessRole",
            "Condition": {
                "Bool": {
                    "aws:MultiFactorAuthPresent": "true"
                }
            }
        }
    ]
}
```

Where `210987654321` is the number of your new account, and `OrganizationAccountAccessRole` is the rolename you chose at account creation.

I'd highly recommend denying any other permissions for the user on the root account. For example if the top level DSNs is registered in the root account's Route63 service, then it might be useful to attach the `AmazonRoute53FullAccess` to the new user.

Once this is done, sign in with the new user and switch roles to the new account.

## Enable CloudTrail for logging operations

CloudTrail logs changes in the account, which is important for security and compliance purposes, so it's helpful to set it up as soon as possible. Storing the logs have a monthly fee though.

Go to `CloudTrail` on the AWS Console and push `Create a trail`, it's OK to use the default settings.

## Create DNS Settings

Assuming that the root AWS account has the top level domain registered, let's say `example.com`. Then the idea is to have these templates deploy everything under a subdomain of it, like `dev.example.com`.

For this, in the newly created account, go to Route 53, and create a new hosted zone for `dev.example.com`, as a public hosted zone.

This new hosted zone will only be known and used by the public, when it's connected to the root DNS account. There will be an NS type record automatically created for the subdomain (`dev.example.com`). Then go to the AWS root account, Route 53, and select the hosted zone of the root DNS (`example.com`). Select `Create record`. For the subdomain use the new subdomain (`dev`), the record type has to be `NS`. 

## Create generic a HTTPS certificate

We'll need a certificate that can be used for serving any backends or frontends using HTTPS.

Go to `AWS Certificate Manager` on the AWS Console, and proceed with `Request certificate`. Select requesting a public certificate.

For the domain names, specify the new domain (`dev.example.com`). Add another domain with a wildcard (`*.dev.example.com`). If the DNS was set up using Route 53, then you can choose DNS validation.

To accomplish DNS validation, you can view the details of the certificate, and in the Domains section select `Create records in Route 53`. Validation should be done in less than a minute.

The ARN of the newly created certificate will be used later. 

## Deployment ID

After this point a couple of templates are going to be deployed. They have references to each-other. In order for them to find each other, an identifier is used in all of them. It's the `DeploymentID` parameter in every template.

Choose an id for your deployment, and use that in all cases below. For example it can be `dev`.

## Deployment Type

Two different types of deployments are supported:
- `PROD` adds more monitoring features
- `DEV` is cheaper

## Deploy VPC

Deploying the following template sets up a private network with private and public subnets, with the required routing tables, internet gateways, NAT settings and a set of security groups intended to be used for generic purposes.

Go to `AWS Console > CloudFormation` and select `Create stack`. Select that the template is ready, and choose `Upload a template file`. Select `vpc-template.yml` from this repo.

Use the following parameters:
- Stack name: `vpc` (can be whatever)
- DeploymentId: `dev` (see the Deployment ID section above)
- DeploymentType: `DEV` (or `PROD`, see the Deployment Type section above)
- PrimaryPrivateCidr: `10.51.0.0/19` (feel free to use different CIDR settings)
- PrimaryPublicCidr: `10.51.128.0/20`
- SecondaryPrivateCidr: `10.51.32.0/19`
- SecondaryPublicCidr: `10.51.144.0/20`
- VPCCidr: `10.51.0.0/16`

It's OK to use the default stack option configuration.
Before submission, acknowledge that the templates are going to create IAM roles.

Expected to take 3-4 minutes to deploy.

## Deploy Authentication

Deploying the related template sets up AWS Cognito.

Go to `AWS Console > CloudFormation` and select `Create stack`. Select that the template is ready, and choose `Upload a template file`. Select `authentication-template.yml` from this repo.

Use the following parameters:
- Stack name: `authentication` (can be whatever)
- AuthUserAuthenticationSMSText: `Hi {username}, your verification code is {####}`
- AuthUserInvitationSMSText: `Hi {username}, your invitation code is {####}`
- AuthUserVerificationSMSText: `Hi {username}, your verification code is {####}`
- DeploymentId: `dev` (see the Deployment ID section above)
- DeploymentType: `DEV` (or `PROD`, see the Deployment Type section above)

It's OK to use the default stack option configuration.
Before submission, acknowledge that the templates are going to create IAM roles.

Expected to take 2-3 minutes to deploy.

## Deploy the API Gateway

Deploying the next template sets up an API Gateway. The idea is that backend services with public endpoints register resources on this API Gateway. The gateway uses IAM based authentication, assuming that the connecting frontends are using AWS Cognito from the above template.

Go to `AWS Console > CloudFormation` and select `Create stack`. Select that the template is ready, and choose `Upload a template file`. Select `authentication-template.yml` from this repo.

Use the following parameters:
- Stack name: `api` (can be whatever)
- BackendApiHostname: `api` (use anything you want, this will be the domain name of the api gateway under your subdomain)
- DeploymentId: `dev` (see the Deployment ID section above)
- DeploymentType: `DEV` (or `PROD`, see the Deployment Type section above)
- FullDomainName: the domain created in the new AWS account (`dev.example.com`)
GenericCertificateArn: the ARN of the certificate created above

It's OK to use the default stack option configuration.
Before submission, acknowledge that the templates are going to create IAM roles.

Expected to take 3-4 minutes to deploy.

## Deploy Miscellaneous tools

This template adds the following things:
- An AWS SNS topic that can be used from any backend services to send system-level alerts and notifications. The SNS messages are not delivered anywhere by these templates, but the idea is that a simple lambda function can take them and send them over to email, Slack or whatever.
- A generic secret for the deployment. It's supposed to store things like credentials for 3rd party services used by backend services, like for example Slack webhook settings for delivering the alerts. This secret will be accessible by any backend component.
- A secret specificly for GitHub credentials. It's used by CICD pipelines to get notified about changes and pull sources.

Go to `AWS Console > CloudFormation` and select `Create stack`. Select that the template is ready, and choose `Upload a template file`. Select `misc-template.yml` from this repo.

Use the following parameters:
- Stack name: `miscellaneous` (can be whatever)
- DeploymentId: `dev` (see the Deployment ID section above)
- DeploymentType: `DEV` (or `PROD`, see the Deployment Type section above)
- EmailNotificationList: provide an email address, or a comma-separated list, where CloudWatch alarms are sent (these alarms are not sent to the alerts SNS, as those are not delivered if there's no lambda to distribute them.)

It's OK to use the default stack option configuration.

Expected to take up to 1 minute to deploy.

Wohoo, at this point we're ready to deploy the actual app!

## Setup secrets

In order to be able to deploy things from GitHub, credentials for GitHub has to be provided.

Go to `AWS Secrets Manager > Secrets` and select the one of the name starting with `GitHubSecret-`. The `token`` key-value pair is supposed be there, but with the value empty.

Get an access token from GitHub: `GitHub > Settings > Developer Settings > Personal Access Tokens > Tokens (classic)`, and generate one with enabled all `repo` and `admin:repo_hook` permissions. Copy the token value to the secret as the value of the key `token`.

If you're planning to use 3rd party services, like Slack for delivering alerts, then you can put them into the secret with the name starting with `Secret-`.