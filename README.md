# devopsology-monit-workshop

## AWS authentication

Please use your crenetials.csv file that you download when create IAM user or generate
new one.
Create file `~/aws_creds.txt` with such content:

```bash
export AWS_ACCESS_KEY_ID=""
export AWS_SECRET_ACCESS_KEY=""
```

Before start terraform commands please do:

```bash
source ~/aws_creds.txt
```

## Install awscli

```bash
easy_install pip
pip install awscli
aws configure
```

## Create EC2 instance for Prometheus setup

```bash
export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault, Values=true" --query 'Vpcs[*].{id:VpcId}' --output text --region us-east-1)
export PROM_SG=$(aws ec2 create-security-group --description "Prometheus server" --vpc-id $VPC_ID --group-name "prometheus-server-sg" --region us-east-1 --output text)
aws ec2 authorize-security-group-ingress --group-id $PROM_SG --protocol tcp --port 0-65000 --cidr 0.0.0.0/0 --region us-east-1
aws ec2 run-instances --image-id ami-09d069a04349dc3cb --count 1 --instance-type t3.micro --key-name devopsology --security-group-ids $PROM_SG --region us-east-1
```

## Setup Prometheus

Login into it and do:

```bash
yum install git docker vim
sudo curl -L "https://github.com/docker/compose/releases/download/1.25.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
docker-compose --version
```

After please clone our `devopsology-mon-workshop` repo

```bash
git clone https://github.com/Dgadavin/devopsology-mon-workshop.git
cd devopsology-mon-workshop/monitoring/prometheus
docker-compose up
```
