# acit4640_lab-week3-aws-cli
Week 3 lab - create 3 scripts 

Lab 3 - Jasmeen Sandhu, Augustin Nguyen, Maksym Buhai
    1. Download the AWS CLI : `curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"`
    2. Unzip the file: `unzip awscliv2.zip`
    3. Install AWS CLI to `/usr/local/bin`:
        
        `sudo ./aws/install`
        
    4. Creating IAM User
        1. Go to IAM Console → User → Create user
        2. User name - acit4640-user, no need to provide AWS Management Console
            
            ![image.png](attachment:139ee538-7723-48a2-bb3f-b0463d60313a:image.png)
            
        3. Set permissions - Attach policies directly → AdministratorAccess
            
            ![image.png](attachment:bb3534cf-e717-4838-b8b1-0adfcb69b97a:image.png)
            
        4. Create User
        5. Select the User → Security Credentials 
        6. Create access key 
            - CLI: Will give an Alternatives Recommended ⇒ ignore this since we are not using this for production for real users
                
                ![image.png](attachment:468aae0c-51bd-42e6-ad09-84cbdba533c5:image.png)
                
            - Descriptions Tag: up to you
            - Retrieve access keys
            
            ![image.png](attachment:ee22fbfe-7874-49c2-8641-fd94828b2601:image.png)
            
    5. Configure AWS CLI: `aws configure --profile acit4640_admin`
        - Provide access key, secret access key and default region
        
        ![image.png](attachment:f02fe65f-5b66-46f2-b58a-8f3165eb9521:image.png)
        

  ## Script 1 - import-key script
    
    ```bash
    #!/usr/bin/env bash
    
    set -eu
    
    # Function to handle errors
    err() {
      error_messsage="$@"
      echo -e "\033[1;31m ERROR:\033[0m ${error_messsage}" >&2
      exit 1
    }
    
    # define variable to hold public key
    public_key_file=""
    
    # check to see the correct number of arguments have been passed to your script
    if [[ $# -ne 1 ]]; then
      err "script requires the path to the public key file you would like to import"
    fi
    
    # check to see if the public key file passed as the first positional parameter is a regular file
    # if it is, set the path to the value of the public_key_file variable
    if [[ ! -f $1 ]]; then
      err "path to public key for import is incorrect, file does not exist"
    else
      public_key_file="$1"
    fi
    
    # use aws to import the public key into aws account, write info about import to key_data file
    # aws will generate errors, so I am just letting the aws cli handle error messages here
    aws ec2 import-key-pair --key-name "COMPLETE ME" --public-key-material fileb://${public_key_file} > key_data
    
    ```
    
    - Change key-name to “bcitkey”

## Script 2 - create-bucket

```bash
#!/usr/bin/env bash

set -euo pipefail

# Check if the number of command-line arguments is correct
if [ "$#" -ne 1 ]; then
    echo "Usage: $0 <bucket_name>"
    exit 1
fi

# pass the bucket name as a positional parameter
bucket_name=$1

# Check if the bucket exists
if aws s3api head-bucket --bucket "$bucket_name" 2>/dev/null; then
    echo "Bucket $bucket_name already exists."
else
  # change the line below
  echo $bucket_name
fi
```

- Add create bucket command
`aws s3api create-bucket --bucket "$bucket_name"  --create-bucket-configuration LocatLocationConstraint=us-west-2`

## Script 3 - create-ec2

```bash
#!/usr/bin/env bash

set -euo pipefail

source "$(dirname "$0")/infrastructure_data"

region="us-west-2"
key_name="bcitkey"

source ./infrastructure_data

# Get most recent Debian AMI
debian_ami=$(aws ec2 describe-images \
  --owners "136693071363" \
  --filters 'Name=name,Values=debian-*-amd64-*' 'Name=architecture,Values=x86_64' 'Name=virtualization-type,Values=hvm'>
  --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
  --output text)

# Create security group allowing SSH and HTTP from anywhere
security_group_id=$(aws ec2 create-security-group --group-name MySecurityGroup \
 --description "Allow SSH and HTTP" --vpc-id $vpc_id --query 'GroupId' \
 --region $region \
 --output text)

aws ec2 authorize-security-group-ingress --group-id $security_group_id \
 --protocol tcp --port 22 --cidr 0.0.0.0/0 --region $region

aws ec2 authorize-security-group-ingress --group-id $security_group_id \
 --protocol tcp --port 80 --cidr 0.0.0.0/0 --region $region

# Launch an EC2 instance in the public subnet
# use run-instances command to create a new instance and filter to get instance_id
instance_id=$(aws ec2 run-instances \
  --image-id $debian_ami \
  --instance-type t3.micro \
  --key-name $key_name \
  --security-group-ids $security_group_id \
  --subnet-id $subnet_id \
  --associate-public-ip-address \
  --query 'Instances[0].InstanceId' \
  --region $region \
  --output text)

# wait for ec2 instance to be running
aws ec2 wait instance-running --instance-ids $instance_id

# Get the public IP address of the EC2 instance
# use decribe-instance command to filter for the public_ip of the instance
public_ip=$(aws ec2 describe-instances \
  --instance-ids $instance_id \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --region $region \
  --output text)
  
# Write instance data to a file
# echo the Instance ID and Public IP into the instance_info file
echo "Public IP: $public_ip"

echo "Instance ID: $instance_id" > instance_info
echo "Public IP: $public_ip" >> instance_info
```

- Add create ec2 instance command and get instance_id:
    
    ```bash
    instance_id=$(aws ec2 run-instances \
      --image-id $debian_ami \
      --instance-type t3.micro \
      --key-name $key_name \
      --security-group-ids $security_group_id \
      --subnet-id $subnet_id \
      --associate-public-ip-address \
      --query 'Instances[0].InstanceId' \
      --region $region \
      --output text)
     
    ```
    
- Get public IP of the created instance
    
    ```bash
    
    public_ip=$(aws ec2 describe-instances \
      --instance-ids $instance_id \
      --query 'Reservations[0].Instances[0].PublicIpAddress' \
      --region $region \
      --output text)
    ```
    
- Write Instance ID and Public IP to file
