# Auto-Scalable Web Architecture

![image.png](Auto-Scalable%20Web%20Architecture/image.png)

# Table of contents (clickable plan)

1. Create two S3 buckets (app & ALB logs)
2. Create VPC and 4 subnets (2 public, 2 private)
3. Create Internet Gateway & Route Tables (public + private)
4. Create Security Groups (ALB and EC2)
5. Upload the web app to S3 (app.zip)
6. Create IAM Role + Instance Profile for EC2 (S3 read & CloudWatch)
7. Create Launch Template with user-data (aswa-lt)
8. Create Target Group (port 8080)
9. Create Application Load Balancer (aswa-alb) in public subnets
10. Create Auto Scaling Group (aswa-asg) in private subnets (min=0 desired=0)
11. Configure WAF (aswa-waf) and associate with ALB (optional)
12. Enable ALB access logs to S3 (aswa-s3-logs)
13. Create CloudWatch Dashboard & Alarms (CPU & 5XX) + SNS topic for notifications
14. Configure ASG scaling policy (target tracking CPU)
15. Demo: start instance, test in browser, run load test (k6) to trigger scaling
16. Stop & teardown checklist (cost saver)

Each step below includes **what to click**, exact values to enter, and **checks** to verify success.

---

## Step 0 — Set your region & naming

- Console top-right: select a region (example: **US East (N. Virginia) us-east-1**).
- Project name prefix: `aswa` (we will use this in every resource).

---

## Step 1 — Create S3 buckets (app host + ALB logs)

A. **Create app bucket** (host app zipped artifact)

1. Console → S3 → Create bucket
    - Bucket name: `aswa-s3-app-<your-unique-suffix>` (e.g., `aswa-s3-app-aman01`)
    - Region: same region as you selected
    - Keep default options (Block all public access ON)
    - Create bucket

B. **Create logs bucket** (ALB access logs)

1. S3 → Create bucket
    - Bucket name: `aswa-s3-logs-<your-unique-suffix>` (e.g., `aswa-s3-logs-aman01`)
    - Keep Block Public Access ON
    - Create bucket

Check: you should see both buckets in the S3 console.

---

## Step 2 — Create a VPC (aswa-vpc)

1. Console → VPC → Your VPCs → Create VPC
    - Name tag: `aswa-vpc`
    - IPv4 CIDR: `10.10.0.0/16`
    - IPv6 CIDR: No
    - Tenancy: Default
    - Create VPC

Check: `aswa-vpc` appears in VPC list.

---

## Step 3 — Create subnets (2 public & 2 private, across 2 AZs)

We use two AZs in your region (pick the first two AZs the console shows).

Create these four subnets (repeat “Create subnet” for each):

- `aswa-sub-public-a`
    - VPC: `aswa-vpc`
    - Availability Zone: choose AZ A (e.g., `us-east-1a`)
    - IPv4 CIDR: `10.10.1.0/24`
- `aswa-sub-public-b`
    - AZ B (e.g., `us-east-1b`)
    - CIDR: `10.10.2.0/24`
- `aswa-sub-private-a`
    - AZ A
    - CIDR: `10.10.101.0/24`
- `aswa-sub-private-b`
    - AZ B
    - CIDR: `10.10.102.0/24`

Check: All four subnets appear and are in `aswa-vpc`.

---

## Step 4 — Internet Gateway + Route tables

A. Create IGW

1. VPC → Internet Gateways → Create internet gateway
    - Name tag: `aswa-igw`
    - Create & then Attach to VPC: select `aswa-vpc`

B. Public route table

1. VPC → Route Tables → Create route table
    - Name: `aswa-rt-public`
    - VPC: `aswa-vpc`
2. Edit routes → Add route: `0.0.0.0/0` → Target: `aswa-igw`
3. Associations → Subnet associations → Associate with `aswa-sub-public-a` and `aswa-sub-public-b`

C. Private route table

1. Route Tables → Create route table
    - Name: `aswa-rt-private`
    - VPC: `aswa-vpc`
2. Do NOT add 0.0.0.0/0 route (no NAT) — instances will have no public Internet (we will use S3 + role for app fetch).
3. Associate with `aswa-sub-private-a` and `aswa-sub-private-b`.

Check: public subnets have route to IGW; private subnets are associated with private RT.

---

## Step 5 — Create Security Groups

We will make two SGs: ALB SG and EC2 SG.

A. **ALB Security Group** (`aswa-sg-alb`)

1. EC2 console → Security Groups → Create security group
    - Name: `aswa-sg-alb`
    - VPC: `aswa-vpc`
    - Description: `Allow HTTP from Internet`
    - Inbound rule: Type `HTTP`, Protocol `TCP`, Port `80`, Source `0.0.0.0/0` (and ::/0 if using IPv6)
    - Outbound: default allow all
    - Create

B. **EC2 Security Group** (`aswa-sg-ec2`)

1. Create security group
    - Name: `aswa-sg-ec2`
    - VPC: `aswa-vpc`
    - Description: `Allow 8080 from ALB`
    - Inbound rule: Custom TCP — Port `8080` — Source: choose **Security group** and select `aswa-sg-alb` (this makes ALB the only source)
    - Outbound: default allow all
    - Create

Check: Confirm inbound rule on `aswa-sg-ec2` references `aswa-sg-alb` as the source.

---

## Step 6 — Upload a ready-made web app to S3 (app.zip)

We will use the Heroku python-sample-flask (simple GUI). On your laptop/PC:

A. Download and zip:

```bash
git clone https://github.com/heroku/python-sample-flask
cd python-sample-flask
zip -r app.zip *
# Now upload app.zip to S3
aws s3 cp app.zip s3://aswa-s3-app-<your-unique-suffix>/app.zip

```

If you don’t have AWS CLI, you can manually upload via S3 Console:

- S3 → open `aswa-s3-app-...` → Upload → choose `app.zip` → Upload.

Check: `app.zip` appears inside the S3 bucket.

---

## Step 7 — Create IAM Role & Instance Profile for EC2

We need the EC2 instances to read `app.zip` from the app bucket and optionally send CloudWatch logs.

A. Create role

1. IAM → Roles → Create role → AWS service → EC2 → Next.
2. Attach policies:
    - Create and attach **custom inline policy** that allows `s3:GetObject` for your bucket. (Or create a managed policy.)

Inline policy JSON example (create a policy and attach to role; or use managed via console “Create policy”):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::aswa-s3-app-<your-unique-suffix>/*"]
    },
    {
      "Effect": "Allow",
      "Action": ["logs:CreateLogGroup","logs:CreateLogStream","logs:PutLogEvents"],
      "Resource": ["*"]
    }
  ]
}

```

1. Name role: `aswa-role-ec2-s3`
2. Create role.

B. Create Instance Profile (automatically created when role created). Note its name.

Check: Role `aswa-role-ec2-s3` exists and has S3/GetObject permission for your bucket.

---

## Step 8 — Create Launch Template (aswa-lt)

This Launch Template will instruct EC2 to fetch `app.zip` from S3 and start the Flask app on port 8080.

1. EC2 → Launch Templates → Create launch template
    - Launch template name: `aswa-lt-v1`
    - Source AMI: **Amazon Linux 2 AMI (latest)**
    - Instance type: `t3a.nano` (or `t2.micro` if free tier)
    - Key pair: optional (only if you need SSH)
    - Network settings → No need to pick subnet here
    - Security groups → attach `aswa-sg-ec2` (this assigns the EC2 SG)
    - IAM instance profile → choose `aswa-role-ec2-s3`
    - Advanced details → User data: paste the following user-data script (replace bucket name)

**User-data script** (copy & paste exactly; replace `<YOUR_BUCKET_NAME>`):

```bash
#!/bin/bash
set -eux

# update OS, install tools
yum update -y
yum install -y unzip python3 awscli
pip3 install flask gunicorn

# prepare app dir
cd /home/ec2-user
mkdir -p app
cd app

# download app.zip from S3 (needs IAM role with s3:GetObject)
aws s3 cp s3://aswa-s3-app-<your-unique-suffix>/app.zip ./app.zip
unzip -o app.zip

# make sure app file exists; run with gunicorn on port 8080
nohup /usr/local/bin/gunicorn --bind 0.0.0.0:8080 app:app >/home/ec2-user/gunicorn.log 2>&1 &

```

1. Create the launch template.

Check: `aswa-lt-v1` appears. Note: the EC2 security group attached in template is `aswa-sg-ec2`.

---

## Step 9 — Create Target Group for ALB (aswa-tg)

1. EC2 → Load Balancing → Target Groups → Create target group
    - Target type: **Instance**
    - Name: `aswa-tg`
    - Protocol: HTTP, Port: **8080**
    - VPC: `aswa-vpc`
    - Health checks: Path: `/` (if your app has `/health`, use `/health`. Heroku sample serves `/`)
    - Advanced: use default settings (interval 30s is ok)
    - Create target group

Check: `aswa-tg` appears in target groups.

---

## Step 10 — Create the Application Load Balancer (aswa-alb)

1. EC2 → Load Balancers → Create Load Balancer → Application Load Balancer
    - Name: `aswa-alb`
    - Scheme: Internet-facing
    - IP type: IPv4
    - VPC: `aswa-vpc`
    - Availability Zones: select **aswa-sub-public-a** and **aswa-sub-public-b** (public subnets)
    - Security groups: choose `aswa-sg-alb`
    - Listeners: HTTP 80 → Default action: forward to target group `aswa-tg`
    - Advanced: **Enable access logs** (we will point them to `aswa-s3-logs-...`) — you can enable later too
    - Create load balancer

Check: ALB appears in Load Balancers list and has a DNS name (copy it for later).

---

## Step 11 — Registering Security groups relationship

Ensure `aswa-sg-ec2` inbound rule uses `aswa-sg-alb` as source. You already set this earlier; if not, edit `aswa-sg-ec2` Inbound → add source `aswa-sg-alb` on port 8080.

Why: ALB and EC2 are different ENIs and need separate SGs. ALB must be able to reach EC2 on 8080.

---

## Step 12 — Create Auto Scaling Group (aswa-asg)

1. EC2 → Auto Scaling → Auto Scaling Groups → Create Auto Scaling group
    - Name: `aswa-asg`
    - Choose Launch template → `aswa-lt-v1` (Latest)
    - Choose VPC: `aswa-vpc`
    - Subnets: select **private subnets** `aswa-sub-private-a` and `aswa-sub-private-b`
    - Attach to a load balancer: select the ALB and the target group `aswa-tg`
    - Group size: **Min = 0**, **Desired = 0**, **Max = 2** (keeps cost low)
    - Health checks: ELB & EC2 (default)
    - Scaling policies: skip for now — we will add target tracking in Step 14
    - Create Auto Scaling group

Check: `aswa-asg` appears with desired capacity 0.

---

## Step 13 — (Optional) Configure ALB access logs to S3

If you did not enable access logs earlier, configure them now:

1. EC2 → Load Balancers → Select `aswa-alb` → Description tab → Edit attributes → Access logs: Enable → S3 bucket: enter `aswa-s3-logs-<your-unique-suffix>` → Prefix: `alb` → Save.

Check: Access logs enabled. After traffic, logs will appear in that bucket (give it a few minutes).

---

## Step 14 — Create WAF Web ACL (Optional — enable only during demo)

WAF costs money when active. Use only during demo.

1. AWS WAF & Shield → Web ACLs → Create web ACL
    - Name: `aswa-waf`
    - Region: same region
    - Scope: Regional
    - Resource type to associate: Application Load Balancer
    - Association: select `aswa-alb`
    - Default action: Allow
    - Add managed rule groups: e.g., `AWSManagedRulesCommonRuleSet` (use Count first if you want to test)
    - Create

Check: WAF is attached to ALB. In the WAF console you will see matched / blocked counters after traffic.

---

## Step 15 — Create CloudWatch Alarm & Dashboard + SNS topic (notifications)

A. Create SNS topic for notifications

1. SNS → Topics → Create topic
    - Type: Standard
    - Name: `aswa-sns-alerts`
    - Create topic
2. Create subscription for your email:
    - Topic → Create subscription → Protocol: Email → Endpoint: your-email@example.com
    - Confirm subscription from your email.

B. Create CloudWatch alarm for ASG CPU (or EC2 avg CPU)

We will alarm on average CPU across the Auto Scaling Group or EC2 instance.

1. CloudWatch → Alarms → Create Alarm → Select metric
    - Choose EC2 → Per-Instance Metrics → CPUUtilization → choose the EC2 instance (or choose AutoScaling group metrics later)
    - For demo simplicity: create an alarm when CPU > 70% for 2 consecutive periods (2 × 1 minute)
    - Notification: send to `aswa-sns-alerts`
    - Name alarm: `aswa-cw-alarm-highcpu`
    - Create

C. Create CloudWatch alarm for ALB 5xx

1. CloudWatch → Alarms → Create alarm → Select metric
    - Choose `ApplicationELB` → `HTTPCode_Target_5XX_Count` for `aswa-tg` → Sum over 5 minutes > 0 → Notify SNS topic `aswa-sns-alerts`
    - Name: `aswa-cw-alarm-alb-5xx`
    - Create

D. Create a Dashboard

1. CloudWatch → Dashboards → Create dashboard `aswa-cw-dash`
    - Add widgets:
        - ALB RequestCount (select `aswa-alb` or target group)
        - ALB TargetResponseTime
        - EC2 CPUUtilization (for instances)
        - AutoScalingGroup Desired / InService instances (you can add metric widget for `ASG` → `GroupDesiredCapacity` and `GroupInServiceInstances`)
    - Save dashboard

Check: SNS subscription confirmed, alarms created, dashboard shows no data until you spin instances / run tests.

---

## Step 16 — Configure ASG scaling policy (target tracking CPU)

1. EC2 → Auto Scaling Groups → Select `aswa-asg` → Automatic scaling → Add policy
    - Policy type: Target tracking scaling policy
    - Metric type: `ASGAverageCPUUtilization` (predefined)
    - Target value: `60` (scale out if avg CPU > 60%)
    - Cooldown: leave default
    - Create policy

Check: Policy listed under the ASG.

---

## Step 17 — Start demo: scale to 1 instance, check health & access app

A. Start instance

1. EC2 → Auto Scaling Groups → select `aswa-asg` → Edit desired capacity → set to **1** (or use CLI):

```bash
aws autoscaling set-desired-capacity --auto-scaling-group-name aswa-asg --desired-capacity 1 --honor-cooldown

```

B. Wait 2–4 minutes for instance launch and registration into target group.

C. Check Target Group health

1. EC2 → Target Groups → `aswa-tg` → Targets → the instance should appear and become **healthy**.

D. Open the app in browser

1. EC2 → Load Balancers → select `aswa-alb` → Description → **DNS name** → Open `http://<alb-dns>` in browser.
    - You should see the Flask sample app UI (a simple web page).
    - If it doesn’t load: check Target Group health, check SG rules and user-data logs.

Troubleshooting tips:

- If target shows **unhealthy**, check user-data succeeded: use SSM Session Manager or temporarily change ASG to launch instance in public subnet with a key pair to SSH in and inspect `/home/ec2-user/gunicorn.log` (advanced). Usually fixes are: wrong health check path or user-data failed to download or run app.

---

## Step 18 — Traffic Simulation (k6) to trigger ALB & ASG scaling

You will run k6 on your laptop (no AWS cost).

A. Install k6: https://k6.io/docs/getting-started/installation

B. Save this `bot_and_users_test.js` locally:

```jsx
import http from 'k6/http';
import { sleep } from 'k6';

export let options = {
  stages: [
    { duration: '30s', target: 5 },
    { duration: '60s', target: 30 }, // ramp up (will stress CPU)
    { duration: '30s', target: 0 }
  ],
};

export default function () {
  const base = __ENV.TARGET || 'http://REPLACE_ALB_DNS';
  const r = Math.random();
  if (r < 0.2) {
    // bot behavior: multiple quick hits
    for (let i = 0; i < 4; i++) {
      http.get(`${base}/?q=bot${Math.random()}`);
    }
  } else {
    // user behavior
    http.get(`${base}/`);
    sleep(Math.random() * 2 + 1);
  }
}

```

C. Replace `REPLACE_ALB_DNS` with your ALB DNS or run with env var:

```bash
k6 run -e TARGET=http://<alb-dns> bot_and_users_test.js

```

D. While k6 runs:

- Watch CloudWatch Dashboard: EC2 CPU should climb, ALB RequestCount should climb. If average CPU crosses 60% and your ASG policy is active, ASG will scale out (increase InService instances).
- Watch ASG Activity History: it will show scale-out events.

Check: You should see ASG launch a second instance when CPU target is reached (if traffic is sufficient and template/ASG settings permit).

---

## Step 19 — After demo: stop instances & disable WAF

A. Scale down to zero:

```bash
aws autoscaling set-desired-capacity --auto-scaling-group-name aswa-asg --desired-capacity 0 --honor-cooldown

```

Or in console → ASG → Edit desired capacity → 0.

B. Disable / disassociate WAF (if you enabled it) to avoid charges: WAF console → Web ACLs → Select `aswa-waf` → Disassociate or delete (if not needed).

C. Optionally keep ALB & S3 buckets (small bill) or delete ALB to completely avoid ALB hourly charge. Deleting ALB will require you to re-create it later for demos.

---

## Step 20 — Stop / Teardown (optional) — full cleanup

If you want to remove everything after finishing the interview:

1. Set ASG desired = 0, then delete Auto Scaling group.
2. Delete Load Balancer (you must detach WAF first).
3. Delete Target Group.
4. Delete Launch Template.
5. Terminate any EC2 instances (cleanup happens when ASG removed).
6. Delete IAM role (after detaching).
7. Delete S3 buckets (remove objects first).
8. Delete Security Groups, Route Tables, Internet Gateway, Subnets, VPC.
9. Delete CloudWatch alarms and dashboard, delete SNS topic.
    
    This is manual but safe — do it when you won’t need the demo for a while.
    

---

## Quick checks & troubleshooting summary

- ALB returns 503/unhealthy targets → check Target Group health path (`/` or `/health`) and confirm the app is running on port 8080. Ensure `aswa-sg-ec2` inbound allows `aswa-sg-alb` on port 8080.
- EC2 failure to fetch app → verify role `aswa-role-ec2-s3` has `s3:GetObject` on your app bucket and the bucket name in user-data is correct.
- App not visible → confirm ALB DNS open in browser and no local firewall blocking.
- Need logs from EC2? Use SSM Session Manager (install agent in user-data) or temporarily allow SSH by adding a key pair and putting instance in public subnet for debugging.

---

## Files & copy-paste snippets (again for convenience)

### A. User-data (again) — replace bucket:

```bash
#!/bin/bash
set -eux
yum update -y
yum install -y unzip python3 awscli
pip3 install flask gunicorn

cd /home/ec2-user
mkdir -p app
cd app
aws s3 cp s3://aswa-s3-app-<your-unique-suffix>/app.zip ./app.zip
unzip -o app.zip
nohup /usr/local/bin/gunicorn --bind 0.0.0.0:8080 app:app >/home/ec2-user/gunicorn.log 2>&1 &

```

### B. k6 test (again)

(Replace ALB DNS or use -e TARGET)

```jsx
import http from 'k6/http';
import { sleep } from 'k6';

export let options = {
  stages: [
    { duration: '30s', target: 5 },
    { duration: '60s', target: 30 },
    { duration: '30s', target: 0 }
  ],
};

export default function () {
  const base = __ENV.TARGET || 'http://REPLACE_ALB_DNS';
  const r = Math.random();
  if (r < 0.2) {
    for (let i = 0; i < 4; i++) {
      http.get(`${base}/?q=bot${Math.random()}`);
    }
  } else {
    http.get(`${base}/`);
    sleep(Math.random() * 2 + 1);
  }
}

```

### C. Start/stop CLI commands

Start (1 instance):

```bash
aws autoscaling set-desired-capacity --auto-scaling-group-name aswa-asg --desired-capacity 1 --honor-cooldown

```

Stop (scale to 0):

```bash
aws autoscaling set-desired-capacity --auto-scaling-group-name aswa-asg --desired-capacity 0 --honor-cooldown

```

---

## Final checklist before you start Step 1

- AWS Console login with permissions to create VPC, EC2, IAM, S3, ALB, WAF, CloudWatch.
- Decide unique suffix for S3 bucket names (they must be globally unique).
- Pick your AWS region (keep consistent).
- Optional: Install AWS CLI and k6 locally for testing.

---

If you want, I can now:

- Produce a **one-page printable checklist** (PDF) with every click condensed — ready to follow offline.
- Or **generate GitHub repo files** (README + user-data + k6 script + a short walkthrough).
- Or produce **exact screenshots** for each console step (helps if you want visual guidance).

---

created policy and will attach role to the same policy