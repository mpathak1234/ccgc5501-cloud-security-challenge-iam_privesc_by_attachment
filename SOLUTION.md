terminated the target EC2 instance
The key takeaway is that cloud security requires understanding how different services interact. A permission that seems safe in isolation can become a critical vulnerability when combined with other permissions. Organizations must implement least privilege, monitor high-risk API calls, and regularly audit IAM policies to prevent such escalation paths.

9.	Appendix
9.1 Key Resources
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

9.2 Commands Reference
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





10.	Cleanup 

After completing the challenge, I cleaned up all resources:

1. Exit from SSH (if you were connected)
exit

2. Terminate the instance created with mighty role
aws ec2 terminate-instances \
  --instance-ids i-0942a2562190e7752 \
  --profile kerrigan-tf \
  --region us-east-1

# 3. Delete the key pair
aws ec2 delete-key-pair \
  --key-name privesc-key \
  --profile kerrigan-tf \
  --region us-east-1

# 4. Remove local key file
del privesc-key.pem

# 5. Swap roles back to original configuration
aws iam remove-role-from-instance-profile \
  --instance-profile-name cg-ec2-meek-instance-profile-iam_privesc_by_attachment-abc123xyz \
  --role-name cg-ec2-mighty-role-iam_privesc_by_attachment-abc123xyz \
  --profile admin

aws iam add-role-to-instance-profile \
  --instance-profile-name cg-ec2-meek-instance-profile-iam_privesc_by_attachment-abc123xyz \
  --role-name cg-ec2-meek-role-iam_privesc_by_attachment-abc123xyz \
  --profile admin

# 6. Destroy terraform infrastructure
terraform destroy -auto-approve

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


