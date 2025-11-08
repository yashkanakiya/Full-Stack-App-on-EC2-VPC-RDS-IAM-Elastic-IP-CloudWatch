# ☁️ Full-Stack App on EC2 + VPC + RDS + IAM + Elastic IP + CloudWatch

## 🎯 Goal
Host a Node.js web app (or backend for your Vue portfolio) on an EC2 instance inside a custom VPC, connected to an RDS MySQL database — with a static Elastic IP, IAM role for secure access, and CloudWatch monitoring.

 - Secure VPC design
- Network routing
- Database connectivity
- Elastic IP management
- IAM-based security
- Cloud monitoring

## Step 1️⃣ — Create a Custom VPC

- Go to VPC → Create VPC
- Name: prod-vue-vpc
- IPv4 CIDR block: 10.0.0.0/16
- Tenancy: Default
- Click Create VPC

<img width="2556" height="1170" alt="image" src="https://github.com/user-attachments/assets/5bc918ea-3828-4896-afab-7b3841205d82" />


## Step 2️⃣ — Create Subnets (Two AZs for RDS)

RDS requires at least two subnets in different Availability Zones (AZs) in the same region.

  Public Subnet 1
- Name: public-subnet-1
- VPC: prod-vue-vpc
- AZ: us-east-1a
- CIDR: 10.0.1.0/24
- Enable auto-assign public IP ✅
  
  Private Subnet 1
- Name: private-subnet-1
- VPC: prod-vue-vpc
- AZ: us-east-1a
- CIDR: 10.0.2.0/24

Private Subnet 2
- Name: private-subnet-2
- VPC: prod-vue-vpc
- AZ: us-east-1b
- CIDR: 10.0.3.0/24

<img width="2557" height="1217" alt="image" src="https://github.com/user-attachments/assets/e9b3f53b-91ea-460d-bd06-e786915b866a" />


## Step 3️⃣ — Create and Attach Internet Gateway

Go to VPC → Internet Gateways
- Create: Name = prod-igw
- Attach it to your prod-vue-vpc

<img width="2557" height="1183" alt="image" src="https://github.com/user-attachments/assets/d0f78ecf-d650-45f1-8d50-49094aa0b02a" />


## Step 4️⃣ — Create Route Tables

Public Route Table
- Name: public-rt
- Attach to: prod-vue-vpc
Add Route:
- Destination: 0.0.0.0/0
- Target: Internet Gateway (prod-igw)
- Associate subnet: public-subnet-1
Private Route Table
- Name: private-rt
- Attach to: prod-vue-vpc
- No external route (kept private)
- Associate subnets: private-subnet-1 and private-subnet-2

<img width="2556" height="1171" alt="image" src="https://github.com/user-attachments/assets/b3c320a4-4e98-41f3-b83b-b1a290f041eb" />


## Step 5️⃣ — Create RDS MySQL (Private)

Go to RDS → Databases → Create database
- Method: Standard Create
- Engine: MySQL
- Template: Free Tier
- DB instance ID: vue-db
- Username: admin
- Password: YourPassword123
- VPC: prod-vue-vpc
Subnet group:
- Click Create new DB Subnet Group
- Add both private-subnet-1 and private-subnet-2
- Save and select it.
- Public Access: No ❌
- VPC Security Group: Create new one rds-sg
- Storage: 20 GB (default)
- Availability zone: No preference
- Click Create database

<img width="2542" height="1182" alt="image" src="https://github.com/user-attachments/assets/8d82ae68-8a22-4f84-b09b-4b8a5739c8f1" />


## Step 6️⃣ — Create EC2 (Public)

- Go to EC2 → Launch Instance
- Name: vue-ec2-backend
- AMI: Ubuntu 22.04 LTS
- Type: t2.micro
- Key Pair: (create new or use existing)
- Network: prod-vue-vpc
- Subnet: public-subnet-1
- Auto-assign Public IP: Enabled
- Security Group: Create ec2-sg
Inbound:
- SSH (22) → My IP
- HTTP (80) → Anywhere
Outbound: All traffic
Advanced → User Data (paste below):
```
#!/bin/bash
apt update -y
apt install -y nodejs npm mysql-client
mkdir -p /home/ubuntu/app
cd /home/ubuntu/app
cat <<EOF > server.js
const http = require('http');
const mysql = require('mysql2');
const connection = mysql.createConnection({
  host: 'RDS-ENDPOINT-HERE',
  user: 'admin',
  password: 'YourPassword123',
  database: 'mysql'
});
const server = http.createServer((req, res) => {
  connection.query('SELECT NOW() AS time', (err, result) => {
    if (err) return res.end('DB Error: ' + err.message);
    res.end('Connected to DB. Time: ' + result[0].time);
  });
});
server.listen(80, () => console.log('App running on port 80'));
EOF
node /home/ubuntu/app/server.js &
```
Click Launch Instance

<img width="2557" height="1172" alt="image" src="https://github.com/user-attachments/assets/5118c84b-71e4-4eed-abc4-edc56a052456" />


## Step 7️⃣ — Allocate and Attach Elastic IP

Go to EC2 → Elastic IPs
- Click Allocate Elastic IP
- Associate it with your vue-ec2-backend instance.

<img width="2558" height="1172" alt="image" src="https://github.com/user-attachments/assets/4123d5e6-bdcf-44bb-8f03-4dfffcce9e85" />


## Step 8️⃣ — Add IAM Role (for CloudWatch)

Go to IAM → Roles → Create Role
- Trusted Entity: AWS Service → EC2
- Permission Policy: CloudWatchAgentServerPolicy
- Name: EC2MonitoringRole
Go to EC2 → Instances → Select instance → Actions → Security → Modify IAM Role
- Attach EC2MonitoringRole

<img width="2558" height="795" alt="image" src="https://github.com/user-attachments/assets/9b5710c7-7f96-4265-a0e3-b3718cd29f85" />


## Step 9️⃣ — Link EC2 → RDS security groups (the critical step)

allow inbound MySQL (3306) on the RDS SG only from your EC2 SG.
- EC2 → Instances → click vue-ec2-backend → Security → click ec2-sg link → note the Security group ID, e.g. sg-0a1b2c3d.
- RDS → Databases → click vue-db → Connectivity & security → under VPC security groups click rds-sg (this opens the EC2 Security Group page for rds-sg).
On the rds-sg page:
- Click Edit inbound rules
Add rule
- Type: MySQL/Aurora
- Protocol: TCP
- Port range: 3306
- Source: Custom → search/select the EC2 SG ID (sg-0a1b2c3d (ec2-sg))
Save rules

<img width="2552" height="1085" alt="image" src="https://github.com/user-attachments/assets/c7e76330-cba6-4ef8-9a73-172b80963843" />


## Step 🔟 - Verify internals & connectivity from EC2

SSH into the EC2 instance (use key and Elastic IP):
```
chmod 400 your-key.pem
ssh -i EC2-project.pem ubuntu@vue-db.cet6ci6q4cmf.us-east-1.rds.amazonaws.com
```

On EC2 run these commands:

Check DNS resolves:
```
nslookup vue-db.cet6ci6q4cmf.us-east-1.rds.amazonaws.com
```


Test port connectivity (install netcat if needed):
```
nc -zv vue-db.cet6ci6q4cmf.us-east-1.rds.amazonaws.com 3306
# expected: succeeded
```

Try connecting with mysql client:
```
mysql -h vue-db.cet6ci6q4cmf.us-east-1.rds.amazonaws.com -u admin -p
# enter Password123
```


If you get the MySQL prompt, connectivity works.

Create your app file

Create server.js file:
```
nano server.js
```

Paste this code (replace <RDS-ENDPOINT>):
```
const http = require('http');
const mysql = require('mysql2');

const connection = mysql.createConnection({
  host: 'vue-db.cet6ci6q4cmf.us-east-1.rds.amazonaws.com',
  user: 'admin',
  password: 'Password123',
  database: 'mysql'
});

const server = http.createServer((req, res) => {
  connection.query('SELECT NOW() AS time', (err, result) => {
    if (err) return res.end('DB Error: ' + err.message);
    res.end('Connected to DB. Time: ' + result[0].time);
  });
});

server.listen(80, () => console.log('Server running on port 80'));
```

Save with Ctrl + O, press Enter, then Ctrl + X.

Install dependencies
```
npm install mysql2
```
Start the server
```
sudo node server.js
```
Server running on port 80

Test the Node app locally:
```
curl http://localhost:80
# expect: Connected to DB. Time: 2025-...
```

Test externally:
```
curl http://<Elastic-IP>
# or open in  browser
```

<img width="2547" height="1177" alt="image" src="https://github.com/user-attachments/assets/82666223-cdb4-4278-ad12-fb2b7ec14e9f" />


<img width="2558" height="442" alt="image" src="https://github.com/user-attachments/assets/16f5c4a1-a1c2-434d-9843-59dab477bb3d" />






