---
title: Unauthenticated reconnaissance of account IDs
date: 2023-10-06 18:39:00 +0300
categories: [AWS Security, Reconnaissance]
tags: [aws security, reconnaissance]
image: /assets/img/posts/unauthenticated-reconnaissance-of-account-id/cover.png
---
## Why
Before diving deep into the methods of AWS account ID enumeration, it's crucial to understand the importance of this seemingly innocuous piece of information. 

AWS itself states, "While account IDs, much like other identifiers, should be used judiciously and shared with caution, they aren't classified as secret, sensitive, or confidential information."<br>
On their own, AWS account IDs don't grant direct access to the corresponding AWS account.<br> Yet, they can be a potent tool. 
Here are some potential scenarios:

- **Enumerating Organization's Resources**: Using the account ID, attackers can list public EBS and RDS snapshots linked to the organization.
- **Enumerating IAM**: The account ID can be used to enumerate IAM Users and Roles, offering insights into potential misconfigurations.
- **Service Reconnaissance**: Gaining knowledge about the AWS services in use can help attackers tailor their strategies.
- **Customized Phishing Campaigns**: The account ID can be a valuable addition to personalized phishing attempts, targeting users who have AWS console access specifically.

For readers curious about how an exposed account ID can be exploited, I recommend the following resources:

**Read**: Check out this <a href="https://rhinosecuritylabs.com/aws/assume-worst-aws-assume-role-enumeration" target="_blank">post </a> by RhinoSecurityLabs that delves into the topic.

**Watch**: Additionally, I suggest watching <a href="https://m.youtube.com/watch?v=HXM1rBk_wXs&pp=ygUJI2JlbmVic2Jl" target="_blank">xBen Benmap Morris's presentation</a> "Finding Secrets in Publicly Exposed EBS Volumes" from the DEF CON 27 Conference.


## Generic Reconnaissance vs AWS-focused Techniques

### 1. Generic Reconnaissance Techniques
These are methods that aren't unique to any specific platform. They're general techniques used across various systems, often leading to unintentional disclosures.

- **Screenshots & Screen Recordings:**
Account IDs can be disclosed through screenshots & screen recordings. For instance, during a tutorial, an employee might accidentally expose the company's account ID.
- **Documentation Leaks:**
When tech firms provide documentation on platform integrations, such as AWS, these documents can sometimes unintentionally contain ARNs, which could reveal valid account IDs.
- **Forum Troubleshooting:**
Developers sometimes share their issues on forums. For example, they might post an error message like: "User: arn:aws:iam::123456789102:user/johan is not authorized..." This could inadvertently make the account ID public, especially if attackers monitor employees' forum activities.
- **Public Git Repositories:**
Public repositories are a frequent target for adversaries. Unintentionally shared AWS details like ARNs, Access keys, API keys, and Bucket names can potentially reveal an organization's AWS account ID and may lead to more dire consequences.
Here is how the git secret scanning tool, TruffleHog, can be utilized for scanning repositories.
1. Installing TruffleHog:
```bash
pip install trufflehog
```
2. Creating a Rule File: TruffleHog uses regex patterns to detect secrets. Here's an example configuration tailored for AWS:
```
{
    "AWS API Key1": "((?:A3T[A-Z0-9]|AKIA|AGPA|AIDA|AROA|AIPA|ANPA|ANVA|ASIA)[A-Z0-9]{16})",
    "Amazon MWS Auth Token": "amzn\\.mws\\.[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}",
    "AWS API Key2": "AKIA[0-9A-Z]{16}",
    "AWS AppSync GraphQL Key": "da2-[a-z0-9]{26}",
    "AWS ARN": "^(?=.*\\b\\d{12}\\b)(?=.*\\bARN\\b)(?=.*\\baws\\b).*$",
    "S3 URI": "s3:\\/\\/[a-zA-Z0-9._-]+(?:\\/|$)",
    "S3 URL": "(?:s3:\\/\\/[a-zA-Z0-9._-]+\\/.+)|(?:https:\\/\\/[a-zA-Z0-9._-]+\\.s3\\.amazonaws\\.com\\/.+)"
}
```
3. Scanning a Repository with TruffleHog: Use the following command to scan a repository using the aforementioned rules:
```bash
trufflehog --entropy=False --regex --rules config.txt /path/to/repository
```
### 2. AWS-focused Techniques
#### Leveraging Public RDS/EBS snapshots or AMIs for Account ID Disclosure
Within AWS, the account IDs are often inadvertently exposed through certain public resources. Specifically:
- For **Public EBS snapshots**, the account ID is stored in the "owner" field. However, identifiable descriptions can help in tracing an EBS snapshot back to a specific company.
- For **Public RDS snapshots**, the account ID is stored within the ARN of the snapshots, unique DB instance or cluster names can hint at the ownership of the instance or cluster.
- For **Public AMIs**, the account ID is stored within the  "owner" field, but recognizable AMI names can assist in identifying which organization might have created it.

> Scenario (Focusing on EBS)
>
> As an example, let's evaluate a hypothetical company, PurpleNeon, with the domain purpleneon.com. If they have used a unique descriptor for their EBS snapshot, we can search for public snapshots with matching descriptions across all regions. 
> 
While the account ID remains in the "owner" field, the description can serve as a clue, linking the snapshot to PurpleNeon.
{: .prompt-info }
- **Using AWS-CLI**:
You can leverage the AWS Command-Line Interface to describe snapshots that match certain criteria:
```bash
aws ec2 describe-snapshots --query "Snapshots[?contains(Description, 'YOUR KEYWORD')]" --region <REGION>
```
- **Using the AWS Console**:
```
https://region.console.aws.amazon.com/ec2/home?region=region#Snapshots:visibility=public;v=3;description=:keyword
```

> Remember to replace 'region' with the specific region in which you are conducting your search.
{: .prompt-tip }


- **Efficient Approach, automating with Pacu**:
For efficiency, I've developed a module for Pacu that automates this. For this demonstration, here's It can be found <a href="https://github.com/RhinoSecurityLabs/pacu/blob/master/pacu/modules/ebs__enum_snapshots_unauth/main.py" target="_blank">here</a> :

1. Creating a wordlist of potential EBS snapshot names:
```
purple-neon
neonpurple
purpleneon.com
```

2. Running the Pacu module:

```bash
Pacu (snap_recon:demo) > run ebs__enum_snapshots_unauth --keyword-wordlist wordlist.txt
  Running module ebs__enum_snapshots_unauth...
[ebs__enum_snapshots_unauth] Starting region ap-northeast-1...
[ebs__enum_snapshots_unauth] Starting region ap-northeast-2...
[ebs__enum_snapshots_unauth] Starting region ap-northeast-3...
[ebs__enum_snapshots_unauth] Starting region ap-south-1...
[ebs__enum_snapshots_unauth] Starting region ap-southeast-1...
[ebs__enum_snapshots_unauth] Starting region ap-southeast-2...
[ebs__enum_snapshots_unauth] Starting region ca-central-1...
[ebs__enum_snapshots_unauth] Starting region eu-central-1...
[ebs__enum_snapshots_unauth] Starting region eu-north-1...
[ebs__enum_snapshots_unauth] Starting region eu-west-1...
[ebs__enum_snapshots_unauth] Starting region eu-west-2...
[ebs__enum_snapshots_unauth] Starting region eu-west-3...
[ebs__enum_snapshots_unauth] Starting region sa-east-1...
[ebs__enum_snapshots_unauth] Starting region us-east-1...
[ebs__enum_snapshots_unauth] [+] Snapshot found: snap-*************
[ebs__enum_snapshots_unauth] Starting region us-east-2...
[ebs__enum_snapshots_unauth] Starting region us-west-1...
[ebs__enum_snapshots_unauth] Starting region us-west-2...
[ebs__enum_snapshots_unauth] ebs__enum_snapshots_unauth completed.

[ebs__enum_snapshots_unauth] MODULE SUMMARY:

  1 EBS Snapshots found
  
Pacu (snap_recon:demo) > data ec2
{
  "Snapshots": [
    {
      "AccountId": "",
      "Description": "purpleneon.com-staging",
      "Encrypted": false,
      "Keyword": "purpleneon.com",
      "OwnerId": "1234567891011",
      "Progress": "100%",
      "Region": "us-east-1",
      "SnapshotId": "snap-*************",
      "StartTime": "Fri, 06 Oct 2023 12:33:12",
      "State": "completed",
      "VolumeId": "vol-************",
      "VolumeSize": 100
    }
  ],
  "Volumes": []
}
```

#### Leveraging Public S3 buckets
Credit: This is based on the excellent <a href="https://cloudar.be/awsblog/finding-the-account-id-of-any-public-s3-bucket" target="_blank">research</a> by Ben Bridts of Cloudar in 2021.

By utilizing the `S3:ResourceAccount Policy` Condition Key, which restricts access based on the S3 bucket's associated account, you can determine the ownership of the bucket.<br> You achieve this by creating a conditional policy for a role. Once this role is assumed, you can attempt to access the bucket or its objects. If the account ID specified in the conditional policy matches the owner's account ID of the bucket, access will be granted through the assumed role. Otherwise, access to the bucket will be denied.

Given that there are a staggering one trillion potential combinations for a 12-digit AWS account ID, brute-forcing each possibility is impractical. The silver lining is the condition policy's support for wildcards. For instance, if a bucket is owned by an account ID commencing with the digit "9", a policy can be formulated permitting access to any public bucket with an owner's account ID beginning with "9". This significantly narrows down the scope, requiring at most 120 attempts to pinpoint the exact matching ID.

- **Step-by-Step Walkthrough:**
1. Testing with "8" as a Starting Point*:
We will initiate our search by targeting account IDs that start with the number "8". This is achieved using a wildcard.
```bash
aws sts assume-role \
--role-arn arn:aws:iam::XXXXXX:role/xxx \
--role-session-name testSession \
--policy '{"Statement": [{"Action": "s3:*","Resource": "*","Condition":\
 {"StringLike": {"s3:ResourceAccount": ["8*"]}}}]}' 
```
Next, we attempt to access the `flaws.cloud` bucket with the session token generated from the `assume-role` operation.
```bash
aws s3 ls s3://flaws.cloud --profile s3-enum
An error occurred (InvalidToken) when calling the ListObjectsV2 operation: The provided token is malformed or otherwise invalid.
```
This error implies the bucket's owning account ID doesn't begin with "8".
2. Repeating the Process with "9":
Employing a similar tactic with the digit "9" gives us access to the bucket's content.
```bash
aws sts assume-role \
--role-arn arn:aws:iam::XXXXXX:role/xxx \
--role-session-name testSession \
--policy '{"Statement": [{"Action": "s3:*","Resource": "*","Condition":\
 {"StringLike": {"s3:ResourceAccount": ["9*"]}}}]}' 
```
Followed by the `s3 ls` command to determine if we have access to the bucket.
```bash
aws s3 ls s3://flaws.cloud --profile s3-enum
2017-03-13 23:00:38       2575 hint1.html
...
2017-02-26 20:59:30       1051 secret-dd02c7c.html
```
The output indicates that using the session token from the assume role allowed us to access the bucket, and it also suggests that this bucket, associated with the account ID, starts with a 9.
3. Zooming In on a Potential Match:
Let's assume after several iterations, we narrow down to `975426262028` as a potential match. We try accessing with this specific account ID:
Command:
```bash
aws sts assume-role \
--role-arn arn:aws:iam::XXXXXX:role/xxx \
--role-session-name testSession \
--policy '{"Statement": [{"Action": "s3:*","Resource": "*","Condition":\
 {"StringLike": {"s3:ResourceAccount": ["975426262028"]}}}]}' 
```
Attempting to access the bucket:
```bash
aws s3 ls s3://flaws.cloud --profile s3-enum
An error occurred (AccessDenied) when calling the ListObjectsV2 operation: Access Denied.
```
This indicates that our guess was incorrect. 
After further testing, we arrive at the correct account ID, `975426262029` and confirm access.

Fortunately, we don't need to go through this process manually. Cloudar has also released a <a href="https://github.com/WeAreCloudar/s3-account-search" target="_blank">tool</a> for this purpose. 

I've made some modifications to the tool to enhance its verbosity, making it clearer what's happening in the background.<bv>
```bash
Attempting to access flaws.cloud using arn:aws:iam::************:role/test without any conditions.
Starting search for account ID (this can take a while).
Testing with digits: 0
Testing with digits: 1
Testing with digits: 2
Testing with digits: 3
Testing with digits: 4
Testing with digits: 5
Testing with digits: 6
Testing with digits: 7
Testing with digits: 8
Testing with digits: 9
Found correct digits up to: 9
Testing with digits: 90
Testing with digits: 91
Testing with digits: 92
Testing with digits: 93
Testing with digits: 94
Testing with digits: 95
Testing with digits: 96
Testing with digits: 97
Found correct digits up to: 97
Testing with digits: 970
Testing with digits: 971
Testing with digits: 972
Testing with digits: 973
Testing with digits: 974
Testing with digits: 975
```
If you want to experiment with this, I highly recommend visiting the following PwnedLabs <a href="https://pwnedlabs.io/labs/identify-the-aws-account-id-from-a-public-s3-bucket" target="_blank">challenge lab</a>.


#### Unique Methods to Disclose AWS Account IDs 

In recent years, security researchers have identified and published methods to disclose AWS account IDs. When these vulnerabilities are discovered, they are reported to AWS, which then takes action to rectify them. Two notable instances of such discoveries include:

 <a href="https://frichetten.com/blog/undocumented-amplify-api-leak-account-id/" target="_blank">Using an Undocumented Amplify API to Leak AWS Account IDs by Nick Frichette</a> - 2023
 
 <a href="https://arkadiyt.com/2021/07/09/getting-partial-aws-account-ids-for-any-cloudfront-website/" target="_blank">Getting Partial AWS Account IDs for any CloudFront Website by Arkadiy Tetelman</a> - 2021

<bv>In this blog post, we've explored both generic reconnaissance techniques, such as accidental disclosures in documentation, forum posts, and public repositories, as well as AWS-focused methods, including the discovery of account IDs through public resources like EBS snapshots, AMIs, and S3 buckets.
hope you enjoyed it!

<h3>References:</h3>

<ul>
  <li><a href="https://rhinosecuritylabs.com/aws/assume-worst-aws-assume-role-enumeration/" target="_blank">Assume the Worst: AWS Assume Role Enumeration</a></li>
  <li><a href="https://cloudar.be/awsblog/finding-the-account-id-of-any-public-s3-bucket/" target="_blank">Finding the Account ID of any Public S3 Bucket</a></li>
  <li><a href="https://www.zeuscloud.io/post/aws-account-id-an-attackers-perspective" target="_blank">AWS Account ID: An Attacker's Perspective</a></li>
  <li><a href="https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-identifiers.html" target="_blank">Managing Account Identifiers - AWS Documentation</a></li>
  <li><a href="https://arkadiyt.com/2021/07/09/getting-partial-aws-account-ids-for-any-cloudfront-website/" target="_blank">Getting Partial AWS Account IDs for Any CloudFront Website</a></li>
  <li><a href="https://frichetten.com/blog/undocumented-amplify-api-leak-account-id/" target="_blank">Undocumented Amplify API Leak Account ID</a></li>
</ul>
