# AWS IAM Roles Anywhere Demo

This repository contains a step-by-step demonstration of AWS IAM Roles Anywhere, a service that enables AWS Identity and Access Management (IAM) roles to be used for workloads running outside of AWS.

![AWS IAM Roles Anywhere](aws-iam-roles-anywhere.png)

## Overview

IAM Roles Anywhere allows on-premises servers, containers, and applications to obtain temporary AWS credentials by using X.509 certificates. This eliminates the need to manage long-term AWS access keys for workloads running outside of AWS.

This demo guides you through:

1. Creating your own Certificate Authority (CA)
2. Setting up IAM Roles Anywhere in AWS
3. Configuring an EC2 instance to use IAM Roles Anywhere
4. Testing access to AWS services using temporary credentials

## Prerequisites

- An AWS account with administrator access
- Basic knowledge of AWS services (IAM, EC2)
- OpenSSL installed on your local machine
- SSH client for connecting to EC2 instances

## Repository Contents

- `README.md` - This file
- `Step-by-step-demo-guide.md` - Detailed instructions for the demo
- `config-files/` - Configuration files for certificate creation

## Getting Started

Follow the instructions in the [Step-by-step Demo Guide](./Step-by-step-demo-guide.md) to begin the demonstration.

## Security Note

This demo is for educational purposes only. In a production environment, you should follow AWS security best practices, including:

- Using a properly secured Certificate Authority
- Implementing least privilege access for IAM roles
- Regularly rotating certificates
- Monitoring credential usage with AWS CloudTrail

## License

This project is licensed under the MIT License - see the LICENSE file for details.