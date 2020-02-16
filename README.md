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
yum install -y git docker vim
sudo curl -L "https://github.com/docker/compose/releases/download/1.25.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
service docker start
docker-compose --version
```

After please clone our `devopsology-monit-workshop` repo

```bash
git clone https://github.com/Dgadavin/devopsology-monit-workshop.git
cd devopsology-monit-workshop/monitoring/prometheus
docker-compose up
```

## Setup instance for auto-discovery

```bash
export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault, Values=true" --query 'Vpcs[*].{id:VpcId}' --output text --region us-east-1)
export PROM_SG=$(aws ec2 create-security-group --description "discovery server" --vpc-id $VPC_ID --group-name "auto-discovery-sg" --region us-east-1 --output text)
aws ec2 authorize-security-group-ingress --group-id $PROM_SG --protocol tcp --port 0-65000 --cidr 0.0.0.0/0 --region us-east-1
aws ec2 run-instances --image-id ami-09d069a04349dc3cb --count 1 --instance-type t3.micro --key-name devopsology --security-group-ids $PROM_SG --region us-east-1
```

Please create IAM role for EC2 and attach `AmazonEC2ReadOnlyAccess` policy.
**Attach this role to Prometheus server EC2 instance**

Login to instance and run `node-exporter`

```bash
docker run -d -p 9100:9100 prom/node-exporter
```

Got to Prometheus UI and verify that in `targets` and `service-discovery` tab appear new `node-exporter`

## Grafana setup Telegram alert notification
Save this json to telegram.json file

```json
{
  "name": "Telegram",
  "type": "telegram",
  "isDefault": false,
  "sendReminder": true,
  "disableResolveMessage": false,
  "frequency": "15m",
  "settings": {
    "autoResolve": true,
    "bottoken": "905329307:AAFhAYD1cj9VxC3SWa_1_NWUy7BNHE5041w",
    "chatid": "-1001497391700",
    "httpMethod": "POST",
    "uploadImage": true
  }
}
```

```bash
curl -XPOST -H "Content-Type:application/json" http://admin:12345@54.82.186.94:3000/api/alert-notifications -d @telegram.json
```

## Grafana dashboard builder

Please clone the repo https://github.com/jakubplichta/grafana-dashboard-builder.git

```bash
sudo pip install virtualenv
virtualenv venv
source venv/bin/activate
python setup.py install
cd ../devopsology-monit-workshop/monitoring/prometheus/dashboard-builder
# Please add to config.yaml your Grafana server IP
grafana-dashboard-builder -c config.yaml -p dashboard-builder.yaml --exporter grafana
```

## Opsgenie

###Create an alert with API

Save this json to alert.json file

```json
{
    "message": "[P0][SALES] Test message with only team",
    "responders": [
        {
            "name": "ops_team",
            "type": "team"
        }
    ]
}
```

```bash
curl -XPOST -H "Content-Type:application/json" -H "Authorization:GenieKey YOUR_KEY" -d @alert.json https://api.opsgenie.com/v2/alerts
```
