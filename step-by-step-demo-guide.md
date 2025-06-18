# AWS KMS Workshop: Step-by-Step Guide

This guide walks you through a hands-on demonstration of AWS Key Management Service (KMS) focusing on envelope encryption - a security best practice for protecting sensitive data.

**Time to complete**: ~15 minutes

## Prerequisites

- AWS CLI installed and configured
- Access to an AWS account
- Basic understanding of command line operations

## Workshop Steps

### Part 1: Environment Setup

1. **Deploy the CloudFormation template**

   First, identify a VPC to use for the security group:
   
   ```bash
   aws ec2 describe-vpcs --query "Vpcs[*].{VpcId:VpcId,CidrBlock:CidrBlock,Name:Tags[?Key=='Name'].Value|[0]}"
   ```

   Then deploy the CloudFormation stack with the VPC parameter:

   ```bash
   aws cloudformation create-stack --stack-name kms-workshop --template-body file://kms-workshop.yaml --parameters ParameterKey=VpcId,ParameterValue=YOUR_VPC_ID --capabilities CAPABILITY_NAMED_IAM
   ```

   This template creates:
   - An S3 bucket for the workshop
   - An EC2 key pair for SSH access
   - IAM role with necessary permissions for KMS operations
   - Security group allowing SSH access (in the specified VPC)
   - IAM policy with required KMS permissions

2. **Get CloudFormation outputs**

   Once the stack creation is complete, retrieve the outputs:

   ```bash
   aws cloudformation describe-stacks --stack-name kms-workshop --query "Stacks[0].Outputs"
   ```

   Note the `SecurityGroupId` and `InstanceProfileName` values for the next steps.

3. **Create an EC2 instance**

   You have two options for running this workshop:

   **Option A: Use AWS CloudShell** (No EC2 instance needed) - but your IAM user/role needs to have all necessary permissions to perform the lab operations.
   
   Simply open AWS CloudShell in your AWS Management Console and proceed to Part 2.

   **Option B: Launch an EC2 instance**

   a. Identify a VPC and public subnet to use:
   
   ```bash
   aws ec2 describe-vpcs --query "Vpcs[*].{VpcId:VpcId,CidrBlock:CidrBlock,Name:Tags[?Key=='Name'].Value|[0]}"
   aws ec2 describe-subnets --filters "Name=vpc-id,Values=YOUR_VPC_ID" --query "Subnets[*].{SubnetId:SubnetId,CidrBlock:CidrBlock,AZ:AvailabilityZone,Public:MapPublicIpOnLaunch}"
   ```

   b. Launch an EC2 instance:

   ```bash
   aws ec2 run-instances \
     --image-id ami-0c7217cdde317cfec \
     --instance-type t2.micro \
     --subnet-id YOUR_SUBNET_ID \
     --security-group-ids YOUR_SECURITY_GROUP_ID \
     --key-name KMSWorkshopKeyPair \
     --associate-public-ip-address \
     --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=KMSWorkshop}]'
   ```

   Note: Replace `ami-0c7217cdde317cfec` with an appropriate Amazon Linux 2 AMI ID for your region.

   c. Attach the instance profile:

   ```bash
   aws ec2 associate-iam-instance-profile \
     --instance-id YOUR_INSTANCE_ID \
     --iam-instance-profile Name=KMSWorkshop-InstanceInitRole
   ```

4. **Connect to your EC2 instance**

   You have two options to connect:

   **Option 1: Using SSH from your local machine**

   a. Retrieve the key pair created by CloudFormation:
   
   ```bash
   aws ec2 describe-key-pairs --key-name KMSWorkshopKeyPair
   ```
   
   Note: The private key material is only available when the key is created. For this workshop, you can either:
   - Use the key pair with EC2 Instance Connect (Option 2 below)
   - Create a new key pair if needed:
   
   ```bash
   aws ec2 create-key-pair --key-name KMSWorkshopKey --query "KeyMaterial" --output text > KMSWorkshopKey.pem
   chmod 400 KMSWorkshopKey.pem
   ```

   b. Launch the instance with the key pair:
   
   ```bash
   aws ec2 run-instances \
     --image-id ami-0c7217cdde317cfec \
     --instance-type t2.micro \
     --subnet-id YOUR_SUBNET_ID \
     --security-group-ids YOUR_SECURITY_GROUP_ID \
     --key-name KMSWorkshopKeyPair \
     --associate-public-ip-address \
     --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=KMSWorkshop}]'
   ```

   c. Get your instance's public IP:
   
   ```bash
   aws ec2 describe-instances --instance-ids YOUR_INSTANCE_ID --query "Reservations[0].Instances[0].PublicIpAddress"
   ```

   d. Connect via SSH:
   
   ```bash
   ssh -i KMSWorkshopKey.pem ec2-user@YOUR_INSTANCE_PUBLIC_IP
   ```

   **Option 2: Using EC2 Instance Connect**

   a. In the AWS Management Console, navigate to EC2 > Instances
   
   b. Select your instance and click "Connect"
   
   c. Choose "EC2 Instance Connect" and click "Connect"

5. **Configure AWS CLI**

   If using your local machine or EC2 instance, ensure AWS CLI is configured:

   ```bash
   aws configure
   ```

   Set your preferred region and output format (json recommended).

### Part 2: Creating a KMS Key

6. **Create a KMS Customer Master Key (CMK)**

   ```bash
   aws kms create-key
   ```

   Take note of the `KeyId` in the output - you'll need it for subsequent steps.

7. **Create an alias for your key**

   ```bash
   aws kms create-alias --alias-name alias/KmsWorkshopCMK --target-key-id YOUR_KEY_ID
   ```

   Replace `YOUR_KEY_ID` with the KeyId from the previous step.

### Part 3: Envelope Encryption Demonstration

8. **Create a sample secret text file**

   ```bash
   echo "Sample Secret Text to Encrypt" > samplesecret.txt
   ```

9. **Generate a data key**

   ```bash
   aws kms generate-data-key --key-id alias/KmsWorkshopCMK --key-spec AES_256 --encryption-context project=kms-workshop
   ```

   The output includes:
   - `Plaintext`: The unencrypted data key (base64-encoded)
   - `CiphertextBlob`: The encrypted data key (base64-encoded)
   - `KeyId`: The ARN of the KMS key used to encrypt the data key

10. **Extract the plaintext data key**

    ```bash
    echo 'PLAINTEXT_FROM_PREVIOUS_STEP' | base64 --decode > datakeyPlainText.txt
    ```

    Replace `PLAINTEXT_FROM_PREVIOUS_STEP` with the Plaintext value from step 9.

11. **Save the encrypted data key**

    ```bash
    echo 'CIPHERTEXT_FROM_PREVIOUS_STEP' | base64 --decode > datakeyEncrypted.txt
    ```

    Replace `CIPHERTEXT_FROM_PREVIOUS_STEP` with the CiphertextBlob value from step 9.

12. **Encrypt the secret message using the data key**

    ```bash
    openssl enc -e -aes256 -in samplesecret.txt -out encryptedSecret.txt -k fileb://datakeyPlainText.txt
    ```

13. **View the encrypted content**

    ```bash
    cat encryptedSecret.txt
    ```

    You'll see that the content is now encrypted.

14. **Delete the plaintext data key**

    ```bash
    rm datakeyPlainText.txt
    ```

    In a real scenario, after encrypting your data, you would discard the plaintext data key and only keep the encrypted version.

### Part 4: Decryption Process

15. **Decrypt the data key using KMS**

    ```bash
    aws kms decrypt --encryption-context project=kms-workshop --ciphertext-blob fileb://datakeyEncrypted.txt
    ```

    Note that the same encryption context must be provided for successful decryption.

16. **Extract the decrypted data key**

    ```bash
    echo 'PLAINTEXT_FROM_PREVIOUS_STEP' | base64 --decode > datakeyPlainText.txt
    ```

    Replace `PLAINTEXT_FROM_PREVIOUS_STEP` with the Plaintext value from step 15.

17. **Decrypt the secret message**

    ```bash
    openssl enc -d -aes256 -in encryptedSecret.txt -k fileb://datakeyPlainText.txt
    ```

    You should now see your original secret message.

## Understanding Envelope Encryption

What you've just performed is called "envelope encryption":

1. **Data Key Generation**: AWS KMS generates a data key
2. **Key Encryption**: KMS encrypts the data key with your CMK
3. **Data Encryption**: You encrypt your data using the plaintext data key
4. **Key Management**: You store the encrypted data key alongside your encrypted data
5. **Decryption Process**: To decrypt, you first decrypt the data key using KMS, then use the plaintext data key to decrypt your data

This approach offers several advantages:
- Your sensitive data never leaves your environment
- KMS only needs to decrypt the small data key, not your entire dataset
- You can encrypt large amounts of data efficiently

## Clean Up

To avoid incurring charges, clean up the resources created:

1. **Terminate the EC2 instance (if created)**

   ```bash
   aws ec2 terminate-instances --instance-ids YOUR_INSTANCE_ID
   ```

2. **Delete the CloudFormation stack**

   ```bash
   aws cloudformation delete-stack --stack-name kms-workshop
   ```

3. **Delete the KMS key (optional)**

   Note: KMS keys have a mandatory waiting period before deletion (minimum 7 days).

   ```bash
   aws kms schedule-key-deletion --key-id YOUR_KEY_ID --pending-window-in-days 7
   ```

4. **Delete local files**

   ```bash
   rm samplesecret.txt encryptedSecret.txt datakeyEncrypted.txt datakeyPlainText.txt
   ```

5. **Delete any manually created key pairs (if applicable)**

   ```bash
   aws ec2 delete-key-pair --key-name KMSWorkshopKey
   rm KMSWorkshopKey.pem
   ```
   
   Note: The KMSWorkshopKeyPair created by CloudFormation will be automatically deleted when you delete the stack.