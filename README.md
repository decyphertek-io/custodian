# custodian Setup
# No Root
sudo mkdir /opt/awscli/
sudo chown -R adminotaur /opt/awscli/
# Install AWS CLI under /opt/awscli/ and configure without using sudo
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install -i /opt/awscli -b /opt/awscli/bin
vim ~/.bashrc

export PATH=$HOME/.local/bin:$PATH
export PATH=$PATH:/opt/awscli/bin
export AWS_PROFILE=personal
export AWS_DEFAULT_REGION=us-east-1
source ~/.bashrc


python3 -m pip install c7n --break-system-packages

# Create an assume Role : IAM > Roles > Custom Trust Policy

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::561908028191:user/decyphertek"
            },
            "Action": "sts:AssumeRole",
            "Condition": {
                "StringEquals": {
                    "sts:ExternalId": "cloud-custodian-external-id"
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

# Add role permisisons for User: Iam > Users > decyphertek > Json

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::561908028191:role/Assume_Role_Policy",
            "Condition": {
                "StringEquals": {
                    "sts:ExternalId": "cloud-custodian-external-id"
                }
            }
        }
    ]
}


# AwsCli profile setup
aws configure --profile personal

# Cloud Custodian Config ( Does not load correctly when testing? )
vim ~/.custodian/config.json

{
  "cache": "~/.cache/cloud-custodian.cache",
  "profile": "personal",
  "account_id": "561908028191",
  "assume_role": "arn:aws:iam::561908028191:role/Assume_Role_Policy",
  "external_id": "cloud-custodian-external-id",
  "log_group": null,
  "tracer": null,
  "metrics_enabled": null,
  "metrics": null,
  "output_dir": ".",
  "cache_period": 15,
  "dryrun": false,
  "authorization_file": null,
  "subparser": "run",
  "config": null,
  "configs": []
}

# Export aws settings
vim ~/.bashrc
export AWS_PROFILE=personal
export AWS_DEFAULT_REGION=us-east-1
source ~/.bashrc

# Create and test a policy
vim dev.yaml

# Test and run the policy
custodian validate dev.yml
custodian run dev.yaml --output-dir . --verbose --dryrun
