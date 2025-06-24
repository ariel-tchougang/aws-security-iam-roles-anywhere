# AWS IAM Roles Anywhere - Step-by-Step Demo Guide

This guide provides detailed instructions for demonstrating AWS IAM Roles Anywhere. The demo shows how to use X.509 certificates to authenticate and obtain temporary AWS credentials for workloads running outside of AWS.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Creating an EC2 Instance](#creating-an-ec2-instance)
3. [Setting Up Your Certificate Authority](#setting-up-your-certificate-authority)
4. [Configuring IAM Roles Anywhere](#configuring-iam-roles-anywhere)
5. [Testing IAM Roles Anywhere](#testing-iam-roles-anywhere)
6. [Cleanup](#cleanup)

## Prerequisites

Before starting this demo, ensure you have:

- An AWS account with administrator access
- SSH client for connecting to EC2 instances
- A VPC with at least one public subnet and internet connectivity

## Creating an EC2 Instance

In this section, you'll create an EC2 instance in a public subnet to demonstrate IAM Roles Anywhere.

### Step 1: Create a Security Group

1. Sign in to the AWS Management Console and navigate to the EC2 service.
2. In the left navigation pane, choose **Security Groups**.
3. Choose **Create security group**.
4. Enter the following details:
   - **Security group name**: `IAMRolesAnywhere-Demo-SG`
   - **Description**: `Security group for IAM Roles Anywhere demo`
   - **VPC**: Select your VPC
5. Add the following inbound rules:
   - Type: SSH (TCP/22)
   - Source: Your IP address (for security) or 0.0.0.0/0 (not recommended for production)
6. Choose **Create security group**.

### Step 2: Create or Use an Existing Key Pair

If you already have an SSH key pair, you can use it. Otherwise, create a new one:

1. In the EC2 console, navigate to **Key Pairs** in the left navigation pane.
2. Choose **Create key pair**.
3. Enter a name for your key pair (e.g., `IAMRolesAnywhere-Demo-Key`).
4. Select the key pair type (RSA is recommended) and file format (.pem for OpenSSH or .ppk for PuTTY).
5. Choose **Create key pair** and save the private key file securely.

### Step 3: Launch an EC2 Instance

1. In the EC2 console, choose **Instances** in the left navigation pane.
2. Choose **Launch instances**.
3. Enter a name for your instance (e.g., `IAMRolesAnywhere-Demo`).
4. Select an Amazon Machine Image (AMI) - Amazon Linux 2023 is recommended for this demo.
5. Select an instance type (t2.micro/t3.micro/t3.nano is sufficient for this demo).
6. For the key pair, select the key pair you created or an existing one.
7. For the network settings:
   - Select your VPC
   - Select a public subnet
   - Enable auto-assign public IP
   - Select the security group you created earlier
8. Configure storage as needed (default is sufficient for this demo).
9. Choose **Launch instance**.

### Step 4: Connect to Your EC2 Instance

1. Wait for the instance to be in the "Running" state.
2. Select the instance and choose **Connect**.
3. Follow the instructions to connect to your instance using SSH.

#### Option 1: Connect using SSH from your local machine

```bash
ssh -i /path/to/your-key.pem ec2-user@your-instance-public-ip
```

#### Option 2: Connect using EC2 Instance Connect (Optional)

1. In the EC2 console, select your instance.
2. Choose **Connect**.
3. Select the **EC2 Instance Connect** tab.
4. Choose **Connect**.

## Setting Up Your Certificate Authority

In this section, you'll create your own Certificate Authority (CA) and client certificates directly on the EC2 instance.

### Step 1: Install Required Packages

First, ensure OpenSSL is installed on your EC2 instance:

```bash
# For Amazon Linux 2023
sudo dnf update -y
sudo dnf install -y openssl

# For Amazon Linux 2
# sudo yum update -y
# sudo yum install -y openssl

# For Ubuntu
# sudo apt update
# sudo apt install -y openssl
```

### Step 2: Create a Directory for Certificate Files

```bash
mkdir -p ~/certificates
cd ~/certificates
```

### Step 3: Create the Certificate Authority (CA)

1. Generate an elliptic curve key for the certificate authority:

```bash
openssl ecparam -genkey -name secp384r1 -out rootCA.key
```

2. Create a configuration file for the CA certificate signing request (CSR):

```bash
cat > root_request.config << 'EOF'
[req]
prompt             = no
string_mask        = default
distinguished_name = req_dn
[req_dn]
countryName = US
stateOrProvinceName = NY
localityName = New York
organizationName = Acme Inc.
organizationalUnitName = AC
commonName = Acme CA
EOF
```

3. Create the CSR for the CA certificate:

```bash
openssl req -new -key rootCA.key -out rootCA.req -nodes -config root_request.config
```

4. Create database and serial files required for signing certificates:

```bash
touch index
echo 01 > serial.txt
```

5. Create the CA certificate configuration file:

```bash
cat > root_certificate.config << 'EOF'
# This is used with the 'openssl ca' command to sign a request
[ca]
default_ca = CA
[CA]
# Where OpenSSL stores information
dir             = .
certs           = $dir
crldir          = $dir
new_certs_dir   = $certs
database        = $dir/index
certificate     = $dir/rootCA.pem
private_key     = $dir/rootCA.key
crl             = $crldir/crl.pem   
serial          = $dir/serial.txt
RANDFILE        = $dir/.rand
# How OpenSSL will display certificate after signing
name_opt    = ca_default
cert_opt    = ca_default
# How long the CA certificate is valid for
default_days = 3650
# The message digest for self-signing the certificate
default_md = sha256
# Subjects don't have to be unique in this CA's database
unique_subject    = no
# What to do with CSR extensions
copy_extensions    = copy
# Rules on mandatory or optional DN components
policy      = simple_policy
# Extensions added while singing with the `openssl ca` command
x509_extensions = x509_ext
[simple_policy]
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = optional
domainComponent         = optional
emailAddress            = optional
name                    = optional
surname                 = optional
givenName               = optional
dnQualifier             = optional
[ x509_ext ]
# These extensions are for a CA certificate
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid:always
basicConstraints            = critical, CA:TRUE
keyUsage = critical, keyCertSign, cRLSign, digitalSignature
EOF
```

6. Create the CA certificate (answer "yes" to all prompts):

```bash
openssl ca -out rootCA.pem -keyfile rootCA.key -selfsign -config root_certificate.config -in rootCA.req
```

7. Verify your CA certificate details (note that the version should be 3 (0x2)):

```bash
openssl x509 -noout -text -in rootCA.pem
```

### Step 4: Create Client Certificates

1. Generate a key for the client certificate:

```bash
openssl ecparam -genkey -name secp384r1 -out server.key
```

2. Create a configuration file for the client certificate signing request:

> **Note:** The OU (Organizational Unit) value is important as it will be used in the IAM trust policy. In this example, we're using **"Innovation"**, but you can customize it to your needs.

```bash
cat > server_request.config << 'EOF'
[ req ]
prompt = no
distinguished_name = dn
[ dn ]
C = US
O = Acme
CN = Acme.com
OU = Innovation
EOF
```


3. Create the client certificate signing request (CSR):

```bash
openssl req -new -sha512 -nodes -key server.key -out server.csr -config server_request.config
```

4. Create the client certificate configuration file:

```bash
cat > server_cert.config << 'EOF'
basicConstraints = critical, CA:FALSE
keyUsage = critical, digitalSignature
EOF
```

5. Create the CA-signed client certificate:

```bash
openssl x509 -req -sha512 -days 365 -in server.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out server.pem -extfile server_cert.config
```

6. Verify the client certificate:

```bash
openssl verify -verbose -CAfile rootCA.pem server.pem
```

## Configuring IAM Roles Anywhere

In this section, you'll set up IAM Roles Anywhere in the AWS Management Console.

### Step 1: Create an IAM Role

1. Sign in to the AWS Management Console and navigate to the IAM service.
2. In the left navigation pane, choose **Roles**.
3. Choose **Create role**.
4. For trusted entity type, select **Custom trust policy**.
5. Replace the default policy with the following (replace **`Innovation`** with the OU value you used in your client certificate):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "rolesanywhere.amazonaws.com"
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession",
                "sts:SetSourceIdentity"
            ],
            "Condition": {
                "StringEquals": {
                    "aws:PrincipalTag/x509Subject/OU": "Innovation"
                }
            }
        }
    ]
}
```

6. Choose **Next**.
7. Search for and select the `AmazonS3ReadOnlyAccess` policy (or another policy of your choice).
8. Choose **Next**.
9. Enter a name for the role (e.g., `DemoRolesAnywhere`).
10. Choose **Create role**.

### Step 2: Create a Trust Anchor

1. Navigate to the IAM Roles Anywhere service in the AWS Management Console.
2. Choose **Trust anchors** in the left navigation pane.
3. Choose **Create trust anchor**.
4. Enter the following details:
   - **Name**: `DemoTrustAnchor`
   - **Source**: `External Certificate Bundle`
   - **Certificate bundle**: Upload your `rootCA.pem` file from the EC2 instance
     (You'll need to download it from the EC2 instance to your local machine first):
     ```bash
     # On your local machine
     scp -i /path/to/your-key.pem ec2-user@your-instance-public-ip:~/certificates/rootCA.pem ./
     ```

     *You can also open the file and copy its content, then paste into a newly created file on your local machine.*
5. Choose **Create trust anchor**.
6. Note the Trust Anchor ARN for later use.

### Step 3: Create a Profile

1. In the IAM Roles Anywhere console, choose **Profiles** in the left navigation pane.
2. Choose **Create profile**.
3. Enter the following details:
   - **Name**: `DemoProfile`
   - **Role**: Select the `DemoRolesAnywhere` role you created earlier
4. Choose **Create profile**.
5. Note the Profile ARN for later use.

## Testing IAM Roles Anywhere

In this section, you'll test IAM Roles Anywhere by obtaining temporary credentials on your EC2 instance.

### Step 1: Download and Configure the AWS Signing Helper

1. Download the AWS Signing Helper on your EC2 instance:

```bash
cd ~
wget https://rolesanywhere.amazonaws.com/releases/1.7.0/X86_64/Linux/aws_signing_helper
chmod +x aws_signing_helper
```

2. Use the AWS Signing Helper to obtain temporary credentials:

```bash
./aws_signing_helper credential-process \
  --certificate ~/certificates/server.pem \
  --private-key ~/certificates/server.key \
  --trust-anchor-arn <your-trust-anchor-arn> \
  --profile-arn <your-profile-arn> \
  --role-arn <your-role-arn>
```

Replace `<your-trust-anchor-arn>`, `<your-profile-arn>`, and `<your-role-arn>` with the ARNs you noted earlier.

3. The command should return temporary AWS credentials (access key, secret key, and session token).

### Step 2: Use the Temporary Credentials

1. Export the credentials as environment variables:

```bash
export AWS_ACCESS_KEY_ID=<access-key-from-previous-output>
export AWS_SECRET_ACCESS_KEY=<secret-key-from-previous-output>
export AWS_SESSION_TOKEN=<session-token-from-previous-output>
```

2. Install the AWS CLI if it's not already installed:

```bash
# For Amazon Linux 2023 or Amazon Linux 2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# For Ubuntu
# sudo apt install -y unzip
# curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
# unzip awscliv2.zip
# sudo ./aws/install
```

3. Verify the credentials by listing S3 buckets:

```bash
aws s3 ls
```

4. Try an API call that's not allowed by your role's permissions:

```bash
aws ec2 describe-instances --region us-east-1
```

This should fail with an "Access Denied" error since the role only has S3 read permissions.

### Step 3: Configure AWS CLI Profile (Optional)

For convenience, you can configure an AWS CLI profile to use IAM Roles Anywhere:

```bash
mkdir -p ~/.aws

cat >> ~/.aws/config << EOF
[profile roles-anywhere-demo]
credential_process = ~/aws_signing_helper credential-process --certificate ~/certificates/server.pem --private-key ~/certificates/server.key --trust-anchor-arn <your-trust-anchor-arn> --profile-arn <your-profile-arn> --role-arn <your-role-arn>
EOF
```

Then use the profile with AWS CLI commands:

```bash
aws --profile roles-anywhere-demo s3 ls
```

## Cleanup

To avoid incurring charges, clean up the resources you created:

1. Delete the IAM Roles Anywhere profile:
   - Navigate to the IAM Roles Anywhere console
   - Select the profile you created
   - Choose **Delete**

2. Delete the IAM Roles Anywhere trust anchor:
   - Navigate to the IAM Roles Anywhere console
   - Select the trust anchor you created
   - Choose **Delete**

3. Delete the IAM role:
   - Navigate to the IAM console
   - Select the role you created
   - Choose **Delete**

4. Terminate the EC2 instance:
   - Navigate to the EC2 console
   - Select the instance you created
   - Choose **Instance state** > **Terminate instance**

5. Delete the security group:
   - Navigate to the EC2 console
   - Select **Security Groups**
   - Select the security group you created
   - Choose **Actions** > **Delete security group**

## Conclusion

You have successfully demonstrated AWS IAM Roles Anywhere by:

1. Creating an EC2 instance to simulate an on-premises server
2. Setting up a Certificate Authority and client certificates on the EC2 instance
3. Configuring IAM Roles Anywhere in AWS
4. Testing access to AWS services using temporary credentials

This approach can be extended to actual on-premises servers, containers, and applications, allowing them to securely access AWS services without managing long-term access keys.