---

## Infrastructure Components

All application infrastructure is provisioned via Terraform in us-east-1:

- **VPC** with public and private subnets across 2 availability zones
- **Internet Gateway** for public subnet internet access
- **NAT Gateway** for private subnet outbound-only internet access
- **ECS Cluster** running on Fargate (no EC2 management required)
- **ECR Repositories** for frontend and backend Docker images
- **Application Load Balancer** with path-based routing rules
- **Auto Scaling** — target tracking at 50% CPU, min 1 task, max 4 tasks
- **CloudWatch Log Groups** for container log retention
- **IAM Roles** — separate execution role (infrastructure) and task role (app)
- **Security Groups** — ALB open to internet, ECS tasks only reachable via ALB

### Jenkins Infrastructure (manually provisioned, not Terraform-managed)

Per challenge requirements, Jenkins infrastructure was set up manually:

- **EC2 Instance** — t3.small, Amazon Linux 2023, public subnet, us-east-1
- **Elastic IP** — static public IP for consistent Jenkins access
- **Security Group** — ports 22 (SSH), 80, 443, 8080 (Jenkins UI) open
- **Jenkins** — runs as a Docker container with Docker-in-Docker via socket
  mount, persisted via /var/jenkins_home volume mount
- **IAM credentials** — stored in Jenkins encrypted credential store,
  referenced by ID in Jenkinsfile, never hardcoded

---

## Prerequisites

- Node.js v16 (use nvm: nvm install 16 && nvm use 16)
- Docker Desktop
- AWS CLI configured with appropriate IAM permissions
- Terraform >= 1.0
- Git

---

## Local Setup and Running

### 1. Clone the repository


git clone https://github.com/SomoneL/devops-tech-challenge-1.git
cd devops-tech-challenge-1

### 2. Run the backend locally

cd backend
nvm use 16
npm ci
npm start

Verify: 'http://localhost:8080' returns a JSON GUID.

### 3. Run the frontend locally

In a new terminal tab (keep backend running):

cd frontend
nvm use 16
npm ci
npm start

Verify: 'http://localhost:3000' displays a GUID — confirming
frontend-to-backend communication works.

**Note:** 'frontend/src/config.js' must point to 'http://localhost:8080/'
for local development. For production, it points to the ALB DNS name.

---

## Terraform Deployment

cd terraform
terraform init
terraform plan
terraform apply

**Important notes:**
- Requires 'mysql-keypair' key pair to exist in us-east-1
- Region is set to us-east-1 in variables.tf
- Do not commit '.terraform/', 'terraform.tfstate', or '*.tfvars
- After apply, note the outputs — you will need 'alb_dns_name',
  'frontend_repository_url', 'backend_repository_url', and
  'jenkins_master_public_ip' for subsequent steps

---

## Jenkins Setup

1. SSH into the Jenkins EC2 instance:

ssh -i '/Users/somoneletman/Documents/ALL Cloud Engineering /Action Steps/Week 3/mysql-keypair.pem' ec2-user@23.20.157.205'

2. Jenkins runs as a Docker container — access the UI at:
http://23.20.157.205:8080

3. Credentials configured in Jenkins:
   - **github-PAT** — GitHub username + Personal Access Token
   - **aws-credentials** — IAM Access Key ID + Secret Access Key

4. Plugins installed: Docker, Amazon EC2, Amazon ECS/Fargate

---

## CI/CD Pipeline

The 'Jenkinsfile' at the project root defines a 5-stage pipeline:

1. **Checkout code** — pulls latest from GitHub via SCM
2. **Build Docker images** — builds frontend and backend images
3. **Authenticate to ECR** — generates temporary ECR login token
4. **Tag and Push to ECR** — pushes both images tagged 'latest'
5. **Update ECS services** — triggers force-new-deployment on both services

Pipeline triggers automatically via GitHub webhook on every push to 'main'.

**Webhook URL:** http://23.20.157.205:8080/github-webhook/

---

## Configuration

### frontend/src/config.js
- Local development: 'http://localhost:8080/'
- Production: 'devops-challenge-alb-1218229428.us-east-1.elb.amazonaws.com'

### backend/config.js
- Local development: 'CORS_ORIGIN: 'http://localhost:3000''
- Production: 'CORS_ORIGIN: devops-challenge-alb-1218229428.us-east-1.elb.amazonaws.com

**Important:** After updating config files, Docker images must be rebuilt
with '--no-cache' to prevent stale cached layers from baking in old values.

---

## Troubleshooting

| Issue | Root Cause | Fix |
|---|---|---|
| 'npm ci' fails | Node version too new | Run 'nvm use 16' first |
| Frontend Docker build fails ('ERR_OSSL_EVP_UNSUPPORTED') | Node 18 incompatible with webpack | Use 'node:16' in frontend Dockerfile |
| "Failed to fetch" in browser | 'host.docker.internal' used in config | Use 'localhost' — React runs in browser, not container |
| Docker build uses old config | Build cache stale | Always use 'docker build --no-cache' after config edits |
| ALB returns frontend HTML on '/api' | Path pattern '/api/*' doesn't match '/api' exactly | Change rule to '/api*' |
| Terraform subnet deletion stuck | ALB still attached to subnet | Manually delete ALB first, then retry |
| Terraform cross-region VPC error | State file tracking wrong region | Run 'terraform destroy' in original region, wipe state, rebuild |

---

## Load Testing Results

**Tool:** Siege ('siege -c 250 -t 5M')
**Target:** 'http://devops-challenge-alb-1218229428.us-east-1.elb.amazonaws.com'

| Metric | Result |
|---|---|
| Total transactions | 48,111 hits |
| Availability | 99.96% |
| Transaction rate | 160.19 trans/sec |
| Failed transactions | 21 |
| Avg response time | 1,556ms |
| Peak CPU (frontend) | ~99.9% |

### Auto Scaling Behavior

- **Scale-out:** Frontend scaled from 1 → 4 tasks (maximum) within ~3
  minutes of load test start
- **Trigger:** CPU exceeded 50% threshold (peaked at ~99.9%)
- **First scale event:** 11:59:17 (1 → 2 tasks)
- **Maximum reached:** 12:01:47 (4 tasks)
- **Scale-in:** Began ~15 minutes after load dropped (12:17:49),
  ECS gracefully drained connections before deregistering tasks
- **Backend:** Remained at 1 task — insufficient direct '/api' traffic
  to trigger its own scaling policy

---

## Submission

- **Jenkins URL:** 'http://23.20.157.205:8080'
- **Jenkins credentials:** (provided separately in submission form)
- **Frontend URL:** 'http://devops-challenge-alb-1218229428.us-east-1.elb.amazonaws.com'
- **GitHub repo:** 'https://github.com/SomoneL/devops-tech-challenge-1'
  (private, shared with michaeltayo96@outlook.com)
