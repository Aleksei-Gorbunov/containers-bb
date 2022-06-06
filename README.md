# steps
create env cloud9 us-east-2
#
npm install -g aws-cdk@2.25.0
cdk --version
mkdir containers-bb
cd containers-bb
cdk init app --language typescript
#
# upgrade ENV

pip3 install --user --upgrade boto3
export instance_id=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
python -c "import boto3
import os
from botocore.exceptions import ClientError 
ec2 = boto3.client('ec2')
volume_info = ec2.describe_volumes(
    Filters=[
        {
            'Name': 'attachment.instance-id',
            'Values': [
                os.getenv('instance_id')
            ]
        }
    ]
)
volume_id = volume_info['Volumes'][0]['VolumeId']
try:
    resize = ec2.modify_volume(    
            VolumeId=volume_id,    
            Size=30
    )
    print(resize)
except ClientError as e:
    if e.response['Error']['Code'] == 'InvalidParameterValue':
        print('ERROR MESSAGE: {}'.format(e))"
if [ $? -eq 0 ]; then
    sudo reboot
fi
#
# tools
#Install kubectl
sudo curl --silent --location -o /usr/local/bin/kubectl \
   https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl

sudo chmod +x /usr/local/bin/kubectl
#
#Update awscli
#
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
#
#Install jq, envsubst (from GNU gettext utilities) and bash-completion
#
sudo yum -y install jq gettext bash-completion moreutils

#
#Verify the binaries are in the path and executable
for command in kubectl jq envsubst aws
  do
    which $command &>/dev/null && echo "$command in path" || echo "$command NOT FOUND"
  done
#
#Enable kubectl bash_completion
#
kubectl completion bash >>  ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
#
# IAM role
eks-blueprints-cdk-admin
#
# assign the role to EC2 instance (Cloud9)
# Switch off temp creds in IDE
# remove existing temp creds
rm -vf ${HOME}/.aws/credentials

#
#create env variables
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
#
#check variables 
echo $ACCOUNT_ID
echo $AWS_REGION
#
#save these into bash_profile
echo "export ACCOUNT_ID=${ACCOUNT_ID}" | tee -a ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" | tee -a ~/.bash_profile
aws configure set default.region ${AWS_REGION}
aws configure get default.region
#
#validate IAM role
aws sts get-caller-identity --query Arn | grep eks-blueprints-cdk-admin -q && echo "IAM role valid" || echo "IAM role NOT valid"
#
#
