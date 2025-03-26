---
date: 2025-03-21


categories:
    - AWS
    - Security

tags: 
    - EC2
authors:
    - nicolas
---

A few weeks ago, I came across a fantastic [post](https://securitylabs.datadoghq.com/articles/whoami-a-cloud-image-name-confusion-attack/) from Datadog security's team about name confusion attacks in AWS AMI's and how using the Allowed AMI feature can help prevent this attack vector. While digging into it and declarative policies, I realized there aren't many blog posts out there covering them since they are recent features, hence this post. 

<!-- more -->

## Understanding AMIs

Before we dive in, we need to understand what AMIs (Amazon Machine Images) are. They are pre-configured virtual machines that contain the operating system, application server, and applications needed to run an instance on AWS. They are used to quickly launch instances in the cloud with specific configurations and software. AMIs allow for consistency, scalability, and easy replication of environments. Essentially, they act as templates for EC2 instances.

## Why is important to control them?

Using public/community AMIs may introduce vulnerabilities or malicious software that may compromise your infrastructure. AWS warns about the use of these types of AMIs in their docs:
> *You use a shared AMI at your own risk. __Amazon can't vouch for the integrity or security of AMIs shared by other Amazon EC2 users__. Therefore, you should treat shared AMIs as you would any foreign code that you might consider deploying in your own data center, and perform the appropriate due diligence. We recommend that you get an AMI from a trusted source, such as a verified provider.*

So, what can we do to prevent their use? The answer my friend is __Allowed AMIs__,

## Allowed AMIs feature
Allowed AMIs is a new account-wide setting introduced in December 2024 that enables organizations to limit which AMIs can be discovered and used within their AWS accounts. By default, this setting is disabled in all AWS accounts.  

!!! note

    Before we move any further, I'd like to highlight that this feature only restricts AMIs at the AMI provider level (i.e. if an image comes from Amazon, from the marketplace, from another AWS account, etc.). __It cannot restrict specific AMIs by ID.__

Enabling this feature is pretty straightforward:

1. Navigate to the EC2 service
2. Choose Dashboard
3. Click on __Allowed AMIs__
4. Configure it!

It's worth noting that this feature is regional, so you will need to configure it in every AWS region you use. If you have a handful of AWS accounts and only operate in a few regions, you probably can simply use the Web UI as it would be a one-off configuration. 

However, maintaining this can be hard, especially when the AWS organization grows so we would need an alternative to ClickOps. These could be some sort of automation via the AWS CLI, but even when the CLI is used, it doesn't scale well in large AWS organizations with hundreds or even thousands of accounts, so we need a better solution. 

## Declarative Policies for the win
Alongside the Allowed AMIs feature, AWS announced a new type of organization policy that allows organizations to define and enforce standardized configurations for AWS services across multiple accounts: __Declarative Policies.__

This policy allows organizations to enforce consistent configurations across accounts and across regions, thus simplifying security governance. 

You can configure the policy in three modes:

* `Enabled`

The policy is in effect and as such, users will only be able to see and use AMIs coming from the providers you allowed.

* `Disabled`

Policy is not enforced so it behaves as if the policy wasn't there.

* `Audit Mode`

Policy is in monitoring mode. Users will see a visual red label indicating `Not Allowed` on non-compliant AMIs but it won't prevent users from using them.

## Deployment

At the time of writing this article, the AWS provider in Terraform doesn't mention that it supports Declarative Policies. However, digging into the [PRs](https://github.com/hashicorp/terraform-provider-aws/issues/40534), I found out that it's only a documentation issue, they are indeed supported. So, how do we use it?

Firstly, we need to define the Declarative Policy, and for that, we create a json file. Let's call it `my_ami_policy.json`

```
{
    "ec2_attributes": {
        "allowed_images_settings": {
            "state": {
                "@@assign": "enabled"
            },
            "image_criteria": {
                "criteria_1": {
                    "allowed_image_providers": {
                        "AWS Account ID",
                        "amazon",
                        "aws-marketplace",
                        "aws-backup-vault"
                    }
                }
            }
        }
    }
}
```

Secondly, we then need to create the actual Declarative Policy resource in Terraform:

```
resource "aws_organizations_policy" "example" {
  name    = "example"
  content = file("${path.module}/path/to/your/my_ami_policy.json")
  type = "DECLARATIVE_POLICY_EC2"
}
```

The last thing to do is to attach it to either root, an OU, or an AWS account.

```
resource "aws_organizations_policy_attachment" "custom_scp_policy_attachment" {
  policy_id = aws_organizations_policy.example.id
  target_id = "Account or OU ID"
}
``` 

And just as simple as that, you can implement an additional layer of defense to only use images from trusted sources.

!!! note

    A side benefit of using declarative policies is that the Allowed AMI setting cannot be overridden in member accounts, eliminating the need to manage user permissions related to this setting.". 

    

Thanks for reading!