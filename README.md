## Dicoogle on AWS

This repository contains Cloudformation templates and resources for deploying Dicoogle on AWS. The architecture and deployment instructions are described in a two-part series blog "Running Dicoogle, an open source PACS solution, on AWS" in AWS Open Source Blog channel.

## How to prepare for solution deployment

1. Log in to the AWS Cloud9 service [console](https://console.aws.amazon.com/cloud9/home). Click “Create environment” to create an AWS Cloud9 environment. Enter a name of your choice in step one and then click “Next step.” Use the default settings in step two “Configure settings” and click “Next step.” Lastly, click “Create environment.”
2. Wait until the AWS Cloud9 environment is created, then go to the terminal window at the bottom of the screen.
3. Run the following commands to create a keypair, which will be used from the AWS Cloud9 terminal to ssh into Amazon EC2 instances. Then, save the output (private key) to a file.
   
   ```
   sudo yum install -y jq
   ```
   ```
   aws ec2 create-key-pair --key-name "dicoogle" | jq -r ".KeyMaterial" > ~/dicoogle.pem
   ```
   ```
   chmod 400 ~/dicoogle.pem
   ```
4. Run the following commands to create a Amazon Simple Storage Service (Amazon S3) bucket. Note the bucket name in the output.
   ```
   SUFFIX=$( echo $RANDOM | md5sum | head -c 20 )
   ```
   ```
   BUCKET=dicoogle-$SUFFIX
   ```
   ```
   aws s3 mb s3://$BUCKET
   ```
   ```
   echo “Bucket name: $BUCKET”
   ```
5. Run the following commands to create an AWS Key Management Service (AWS KMS) key first and then three Amazon Elastic Container Registry (Amazon ECR) repositories: one for Dicoogle, one for Nginx, and one for Ghostunnel.
   ```
   KMS_KEY=$( aws kms create-key | jq -r .KeyMetadata.KeyId )
   ```
   ```
   aws ecr create-repository --repository-name dicoogle --encryption-configuration encryptionType=KMS,kmsKey=$KMS_KEY
   ```
   ```
   aws ecr create-repository --repository-name nginx --encryption-configuration encryptionType=KMS,kmsKey=$KMS_KEY
   ```
   ```
   aws ecr create-repository --repository-name ghostunnel --encryption-configuration encryptionType=KMS,kmsKey=$KMS_KEY
   ```
6. Run the following commands to clone the GitHub code repository.
   ```
   cd ~/environment
   ```
   ```
   git clone https://github.com/aws-samples/dicoogle-on-aws
   ```
7. Run the following commands to build docker images and push to Amazon ECR. Note each image name in the output. We’ll use them in step 12.
   ```
   cd ~/environment/dicoogle/docker/dicoogle
   ```
   ```
   ./build.sh
   ```
   ```
   cd ~/environment/dicoogle/docker/nginx
   ```
   ```
   ./build.sh
   ```
   ```
   cd ~/environment/dicoogle/docker/ghostunnel
   ```
   ```
   ./build.sh
   ```
8. Run the following commands to package the AWS Lambda function and upload all deployment artifacts to an Amazon S3 bucket. The uploaded deployment artifacts include all AWS CloudFormation templates, as well as the Lambda function packaged in zip format. They will be used to deploy the solution. Note the Amazon S3 template URL in the output – it will be used in step 11.
   ```
   cd ~/environment/dicoogle
   ```
   ```
   chmod 755 ./artifacts.sh
   ```
   ```
   ./artifacts.sh $BUCKET
   ```
9. The next step requires you to generate and bring your own SSL certificates and keys when setting up your environment. Then, you need to create entries in AWS Secrets Manager and populate them with your SSL certificates and keys.
   For demonstration purposes, I’ll provide instructions to generate self-signed SSL certificates. Note that self-signed certificates are not for production use.
   ```
   cd ~/environment/dicoogle/cert
   ```
   For the instructions below, accept all the default values when prompted. Choose “y” when prompted to sign the certificate or commit.
   Generate root CA certificate:
   ```
   openssl req -x509 -config openssl-ca.cnf -newkey rsa:4096 -sha256 -nodes -out cacert.pem -outform PEM
   ```
   Generate nginx certificate signing request (CSR):
   ```
   openssl req -config openssl-nginx.cnf -newkey rsa:2048 -sha256 -nodes -out nginxcert.csr -outform PEM
   ```
   Sign the CSR for nginx certificate
   ```
   openssl ca -config openssl-ca.cnf -policy signing_policy -extensions signing_req -out nginxcert.pem -infiles nginxcert.csr
   ```
   Generate ghostunnel certificate signing request (CSR)
   ```
   openssl req -config openssl-ghostunnel.cnf -newkey rsa:2048 -sha256 -nodes -out ghostunnelcert.csr -outform PEM
   ```
   Sign the CSR for ghostunnel certificate
   ```
   openssl ca -config openssl-ca.cnf -policy signing_policy -extensions signing_req -out ghostunnelcert.pem -infiles ghostunnelcert.csr
   ```
   Generate client EC2 certificate signing request (CSR)
   ```
   openssl req -config openssl-client.cnf -newkey rsa:2048 -sha256 -nodes -out clientcert.csr -outform PEM
   ```
   Sign the CSR for client EC2 certificate
   ```
   openssl ca -config openssl-ca.cnf -policy signing_policy -extensions signing_req -out clientcert.pem -infiles clientcert.csr
   ```
   Generate storage EC2 certificate signing request (CSR)
   ```
   openssl req -config openssl-storage.cnf -newkey rsa:2048 -sha256 -nodes -out storagecert.csr -outform PEM
   ```
   Sign the CSR for storage EC2 certificate
   ```
   openssl ca -config openssl-ca.cnf -policy signing_policy -extensions signing_req -out storagecert.pem -infiles storagecert.csr
   ```
   Create entries in Secrets Manager
   ```
   chmod 755 ./secrets.sh
   ```
   ```
   ./secrets.sh
   ```
   Note each secret ARN (Amazon Resource Name) in the output. We’ll use them in step 12.
10. Go to the AWS Route 53 service [console](https://console.aws.amazon.com/route53/v2/hostedzones). Locate an existing public hosted zone to use for the solution. Make a note of the “Domain name” and “Hosted zone ID.” We’ll use them in step 12.

## How to deploy the solution

11. Go to the CloudFormation service [console](https://console.aws.amazon.com/cloudformation). Click “Create stack” and select “With new resources (standard).” In “Prerequisite – Prepare template,” choose “Template is ready.” In “Specify template,” select “Amazon S3 URL,” and enter the S3 template URL noted in step 8. Click “Next.”
12. Enter “dicoogle” in “Stack name.” For parameters under “Require input” section, Select “dicoogle” in “KeyName.” Enter the bucket name from step 4 in “S3BucketName.” Enter each image name from step 7 in “DicoogleImage,” “NginxImage,” and “GhostunnelImage.” Enter each secret ARN from step 9 in “NginxCert,” “NginxKey,” “GhostunnelCert,” “GhostunnelKey,” and “CACert.” Enter the domain name from step 10 in “DomainName” and select the hosted zone id from step 10 in “HostedZone.” Select two availability zones from “AvailabilityZones.”
13. For the parameters under the “Contain default value. Input is optional” section, leave the default values as is. Click Next to proceed to “Configure stack options.” Then click “Next” to proceed to “Review.” Select the two checkboxes to acknowledge that CloudFormation might create IAM resources with custom names and might require CAPABILITY_AUTO_EXPAND capability. Then click “Create stack.” The deployment should take about 20 minutes.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

The sample DICOM files in data folder are from [Stage II-Colorectal-CT](https://wiki.cancerimagingarchive.net/pages/viewpage.action?pageId=117113567) without modification. Files are licensed under the [CC BY 4.0 license](https://creativecommons.org/licenses/by/4.0/). See [Citations & data usage policy](https://wiki.cancerimagingarchive.net/pages/viewpage.action?pageId=117113567#1171135676fce6de3480e45c985db7c7181fa81cc).

