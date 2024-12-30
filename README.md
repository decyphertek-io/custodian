Cloud Custodian 
===============

"Cloud Custodian enables you to manage your cloud resources by filtering, tagging, and then applying actions to them. The YAML DSL allows defininition of rules to enable well-managed cloud infrastructure that's both secure and cost optimized."

Install & Setup:
----------------

    # Running AWS as a regular user instead of root
    sudo mkdir /opt/awscli/
    sudo chown -R $USER /opt/awscli/
    # Install AWS CLI under /opt/awscli/ and configure without using sudo
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    unzip awscliv2.zip
    ./aws/install -i /opt/awscli -b /opt/awscli/bin
    # AwsCli profile setup , Need an admin user with aws keys, root acocunt doesnt work.
    

    vim ~/.bashrc

    export PATH=$HOME/.local/bin:$PATH
    export PATH=$PATH:/opt/awscli/bin
    export AWS_PROFILE=decyphertek
    export AWS_DEFAULT_REGION=us-east-1

    source ~/.bashrc
    aws configure --profile decyphertek
    python3 -m pip install c7n --break-system-packages

    # Create an assume Role : IAM > Roles > Custom Trust Policy

    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "AWS": "arn:aws:iam::xxxxxxxxxx:user/decyphertek"
                },
                "Action": "sts:AssumeRole",
                "Condition": {
                    "StringEquals": {
                        "sts:ExternalId": "cloud-custodian-external-id-xxxxxxxxx"
                    }
                }
            }
        ]
    }



    # Go to New Role Created : Assume_role_Policy > Attach Permisisons > Iniline Policy

    {
    "Version": "2012-10-17",
    "Statement": [
        {
        "Effect": "Allow",
        "Action": "*",
        "Resource": "*"
        }
    ]
    }

    # Add Administrator Permissions: IAM > Roles > Assume_Role_Policy > add permisisons > attach Policy > AdministratorAccess

    # Add role permisisons for User: Iam > Users > decyphertek > Json

    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": "sts:AssumeRole",
                "Resource": "arn:aws:iam::xxxxxxxxxxxxx:role/Assume_Role_Policy",
                "Condition": {
                    "StringEquals": {
                        "sts:ExternalId": "cloud-custodian-external-id-xxxxxxxxx"
                    }
                }
            }
        ]
    }

    

    # make sure to add the role arn and external ID to the aws config
    vim ~/.aws/config
    [profile decyphertek]
    region = us-east-1
    output = yaml
    source_profile = decyphertek
    role_arn = arn:aws:iam::xxxxxxxxxxxx:role/Assume_Role_Policy
    external_id = cloud-custodian-external-id-xxxxxxxxxxxx

    # Create and test a policy
    vim dev.yaml

    policies:
    - name: update-ec2-tags
        resource: aws.ec2
        region: us-east-1
        filters:
        - "tag:decyphertek": present
        actions:
        - type: remove-tag
            tags: ["decyphertek"]
        - type: tag
            key: cloud-custodian
            value: production

    # Test and run the policy
    custodian validate dev.yml
    custodian run dev.yml --output-dir . --verbose --dryrun
    # See if data is being found
    cd dev  
    cat resources.json
    # Run the remediaiton
    custodian run dev.yml --output-dir . --verbose

References:
------------

    https://cloudcustodian.io/
    https://cloudcustodian.io/docs/index.html

## AWS CIS V3 Cloud Custodian Policies

This repository contains Cloud Custodian policies that implement AWS CIS V3 controls with automated remediation.

### Prerequisites

1. AWS credentials configured
2. Cloud Custodian installed (`pip install c7n`)
3. Appropriate AWS permissions to create and manage resources

### SNS Notification Setup

The policies use SNS notifications to alert about remediation actions. To set up the required SNS topic:

1. Run the SNS setup policy first:
```bash
custodian run -s output policies/setup/sns-setup.yml
```

2. After the topic is created, subscribe to notifications:
```bash
aws sns subscribe \
    --topic-arn arn:aws:sns:REGION:ACCOUNT_ID:cloud-custodian-alerts \
    --protocol email \
    --notification-endpoint your-email@domain.com
```

3. Confirm the subscription by clicking the link in the email you receive

### Running the Policies

To run all CIS policies:
```bash
custodian run -s output policies/aws-cis-v3/**/*.yml
```

To run a specific section:
```bash
custodian run -s output policies/aws-cis-v3/1-IAM/*.yml
```

### Policy Structure

Each policy follows this pattern:
1. Checks for non-compliant resources
2. Applies automated remediation where possible
3. Sends notification about the remediation action taken

### Notifications

All remediation actions will send notifications via the `cloud-custodian-alerts` SNS topic. The notifications will indicate:
- What CIS control was enforced
- What remediation action was taken
- Which resources were affected