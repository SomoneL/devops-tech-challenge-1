# **Tech Challenge 1**

DevOps Tech Challenge — React frontend and Express backend deployed to AWS ECS Fargate with Terraform and Jenkins

<h2>Overview</h2>

A React frontend and an Express backend, each containerized with Docker and deployed as Fargate tasks on an ECS cluster. All application infrastructure is provisioned with Terraform. A Jenkins server running on EC2 builds both images, pushes them to ECR, and forces a new ECS deployment on every commit to `main`.

The frontend is publicly reachable through an Application Load Balancer, which also routes `/api*` traffic to the backend target group. Both services scale from 1 to 4 tasks at 50% CPU utilization.

<b>Frontend:</b> [http://devops-challenge-alb-1850196483.us-east-1.elb.amazonaws.com]

<b>Jenkins:</b> http://JENKINS_ELASTIC_IP:8080

<img width="635" height="119" alt="Screenshot 2026-09-01 at 9 19 57 PM" src="https://github.com/user-attachments/assets/81d642bd-56bd-4815-96ea-7b36b46fff87" />


<br />

<h2>Requirements Met</h2>

<ul>
  <li><b>Jenkins server</b> on AWS, publicly accessible over the internet</li>
  <li><b>Frontend and backend</b> deployed as containers on ECS with Fargate</li>
  <li><b>Deployment automated</b> in a Jenkins pipeline triggered by a GitHub webhook</li>
  <li><b>Minimum 1 task, desired 1, maximum 4</b> for both services</li>
  <li><b>0.5 vCPU (512 units) and 1 GB memory</b> per task</li>
  <li><b>Auto Scaling</b> triggering at 50% CPU utilization</li>
  <li><b>All application infrastructure in Terraform</b> — ECS cluster, services, task definitions, networking, ALB, IAM, ECR</li>
</ul>

<br />

<h2>Architecture</h2>

```
Internet
  → Application Load Balancer   (public subnets, us-east-1a and us-east-1b)
      ├── default rule  → Frontend target group  → Fargate task, port 3000
      └── /api* rule    → Backend target group   → Fargate task, port 8080
```

<b>Application infrastructure — Terraform managed</b>

<ul>
  <li><b>VPC</b> 10.0.0.0/16 with two public and two private subnets across two availability zones</li>
  <li><b>Internet Gateway</b> for public subnet access; <b>NAT Gateway</b> giving private subnets outbound-only access</li>
  <li><b>ECS cluster</b> devops-challenge-cluster running on Fargate, no EC2 management required</li>
  <li><b>ECR repositories</b> devops-challenge-frontend and devops-challenge-backend</li>
  <li><b>Application Load Balancer</b> devops-challenge-alb, internet-facing, with path-based routing rules</li>
  <li><b>Auto Scaling</b> target tracking policies at 50% CPU, 1 minimum and 4 maximum tasks</li>
  <li><b>CloudWatch Log Groups</b> for container logs from both services</li>
  <li><b>IAM roles</b> — separate task execution role (ECR pull and log writes) and task role (application permissions)</li>
  <li><b>Security groups</b> — ALB open to the internet, ECS tasks reachable only from the ALB</li>
</ul>

ECS tasks run in private subnets. Nothing reaches them directly; all traffic passes through the ALB first, and the NAT Gateway permits outbound traffic only.

<b>Jenkins infrastructure</b>

<ul>
  <li><b>EC2 instance</b> t3.small, Amazon Linux 2023, in a public subnet, 30 GB encrypted gp3 root volume</li>
  <li><b>Elastic IP</b> so the Jenkins URL does not change on restart</li>
  <li><b>Security group</b> allowing port 22 (SSH) and port 8080 (Jenkins UI)</li>
  <li><b>Jenkins</b> runs as a Docker container on the instance, with the host Docker socket mounted so pipeline builds can build images</li>
  <li><b>IAM credentials</b> stored in Jenkins, never hardcoded in the Jenkinsfile</li>
</ul>

Jenkins is a single persistent server rather than a containerized, scaling service. It does not need to scale, and keeping it separate from ECS means a broken cluster never takes the pipeline down with it.

<br />

<h2>Repository Layout</h2>

```
frontend/                 React application
  src/config.js           Backend URL the frontend calls
  Dockerfile
backend/                  Express application
  config.js               Allowed origin for the CORS header
  Dockerfile
terraform/
  provider.tf             AWS provider and region
  networking.tf           VPC, subnets, IGW, NAT Gateway, route tables
  ecs.tf                  Cluster, task definitions, services
  ecr.tf                  Frontend and backend image repositories
  security-iam.tf         Security groups and IAM roles
  alb.tf                  Load balancer, target groups, listener rules
  autoscaling.tf          Target tracking scaling policies
  jenkins.tf              Jenkins EC2 instance and Elastic IP
  variables.tf
  outputs.tf
Jenkinsfile               Five-stage CI/CD pipeline
```

<br />

<h2>Setting Up Your Environment</h2>

<ul>
  <li><b>Node.js 16</b> — run <code>nvm use 16</code> before any npm command. Both Dockerfiles pin <code>node:16</code>.</li>
  <li><b>Docker Desktop</b> — running before any build</li>
  <li><b>AWS CLI v2</b> — configured with credentials that can create VPC, ECS, ECR, IAM, and EC2 resources</li>
  <li><b>Terraform</b> — 1.5 or later</li>
  <li><b>Git</b></li>
  <li><b>Siege</b> — optional, load testing only: <code>brew install siege</code></li>
</ul>

<br />

<h2>Running the Project Locally</h2>

<h3>Step 1: Clone the repository</h3>

```bash
git clone https://github.com/SomoneL/devops-tech-challenge-1.git
cd devops-tech-challenge-1
nvm use 16
```

<h3>Step 2: Start the backend first</h3>

```bash
cd backend
npm ci
npm start
```

Verify at http://localhost:8080 — it should return a JSON GUID.

<h3>Step 3: Start the frontend</h3>

In a second terminal, with the backend still running:

```bash
cd frontend
npm ci
npm start
```

Verify at http://localhost:3000 — a **SUCCESS** message followed by a GUID confirms the frontend reached the backend. An error message means the connection failed.

<img width="1916" height="535" alt="02-local-success" src="https://github.com/user-attachments/assets/5c7fbae1-50b5-4b9e-85a2-95d396e4402c" />


For local development, `frontend/src/config.js` must point to `http://localhost:8080`. In production it points to the ALB DNS name.

<br />

<h2>Deploying the Infrastructure</h2>

<h3>Step 1: Provision with Terraform</h3>

```bash
cd terraform
terraform init
terraform plan
terraform apply
```

Note the outputs — you will need the ECR repository URLs, the ALB DNS name, and the Jenkins Elastic IP in the steps that follow.

<img width="475" height="908" alt="03-terraform-outputs" src="https://github.com/user-attachments/assets/81b77852-a46e-4ba2-aa73-225f93740ede" />


<h3>Step 2: Set up Jenkins</h3>

```bash
ssh -i ~/.ssh/mysql-keypair.pem ec2-user@JENKINS_ELASTIC_IP

sudo dnf update -y
sudo dnf install docker -y
sudo systemctl start docker
sudo systemctl enable docker

sudo docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

Open `http://JENKINS_ELASTIC_IP:8080` and unlock with the initial admin password:

```bash
sudo docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

<b>Plugins:</b> Docker, Docker Pipeline, Amazon ECR, Amazon EC2, and the suggested defaults.

<b>Credentials:</b>

<ul>
  <li><code>github-pat</code> — username and Personal Access Token, required because the repository is private</li>
  <li><code>aws-credentials</code> — AWS Credentials type, an access key scoped to ECR push and ECS update permissions</li>
</ul>

<h3>Step 3: Configure the pipeline job</h3>

New Item → Pipeline, named `devops-tech-challenge-pipeline`. Under Pipeline, set Definition to **Pipeline script from SCM**, SCM to Git, point it at the repository with the `github-pat` credential, branch specifier `*/main`, script path `Jenkinsfile`.

Add a GitHub webhook pointing at `http://JENKINS_ELASTIC_IP:8080/github-webhook/` so every push to `main` triggers a build.

<h3>Step 4: Point the applications at the ALB</h3>

After the first Terraform apply, update both config files with the real ALB DNS name and commit. This triggers the pipeline, which rebuilds both images with the correct values baked in.

<ul>
  <li><code>frontend/src/config.js</code> → <code>http://devops-challenge-alb-1218229428.us-east-1.elb.amazonaws.com</code></li>
  <li><code>backend/config.js</code> → same ALB DNS name, used as the <code>Access-Control-Allow-Origin</code> value</li>
</ul>

These values are compiled into the image at build time, so changing them requires a rebuild — editing the file alone does nothing to the running task.

<br />

<h2>Explanation of the Terraform Code</h2>

<ul>
  <li><code>provider.tf</code> — AWS provider and region, read from <code>var.aws_region</code></li>
  <li><code>networking.tf</code> — VPC, two public and two private subnets built with <code>cidrsubnet()</code>, Internet Gateway, NAT Gateway, and route tables</li>
  <li><code>ecs.tf</code> — the Fargate cluster, both task definitions, and both services</li>
  <li><code>ecr.tf</code> — one image repository per application</li>
  <li><code>security-iam.tf</code> — security groups and the task execution and task roles</li>
  <li><code>alb.tf</code> — load balancer, two target groups with health checks, listener and path-based routing rules</li>
  <li><code>autoscaling.tf</code> — target tracking policies on ECS service CPU</li>
  <li><code>jenkins.tf</code> — the Jenkins EC2 instance and its Elastic IP</li>
  <li><code>variables.tf</code> — every tunable value, defaulted to the challenge spec: CPU 512, memory 1024, min 1, desired 1, max 4, CPU threshold 50</li>
  <li><code>outputs.tf</code> — VPC and subnet IDs, ECR repository URLs, ALB DNS name, Jenkins IP</li>
</ul>

Three decisions worth noting:

<ol>
  <li><b>ECS tasks in private subnets.</b> Containers are never directly reachable from the internet. Even with a task's IP there is no inbound route; the NAT Gateway is outbound-only.</li>
  <li><b>Separate task execution role and task role.</b> The execution role lets ECS pull from ECR and write logs. The task role is what the application itself would use. Splitting them keeps the application from inheriting registry permissions it never needs.</li>
  <li><b>Target type <code>ip</code> rather than <code>instance</code>.</b> Fargate tasks have their own ENIs and no host instance to register, so <code>ip</code> is the only valid target type.</li>
</ol>

<br />

<h2>Explanation of the Jenkins Pipeline</h2>

The `Jenkinsfile` at the project root defines a five-stage pipeline.

<ol>
  <li><b>Checkout code</b> — pulls the latest commit from GitHub via <code>checkout scm</code></li>
  <li><b>Build Docker images</b> — builds <code>frontend:latest</code> and <code>backend:latest</code>, the same commands as a manual build, automated</li>
  <li><b>Authenticate to ECR</b> — generates a temporary login token with <code>aws ecr get-login-password</code>. <code>withCredentials</code> makes the stored AWS keys available without printing them into build logs.</li>
  <li><b>Tag and push images to ECR</b> — <code>docker tag</code> creates the alias pointing at the ECR address, then pushes both images</li>
  <li><b>Update ECS services</b> — <code>aws ecs update-service --force-new-deployment</code> on both services, telling ECS to pull the newly pushed image and replace the running tasks</li>
</ol>

A `post { always { cleanWs() } }` block clears the workspace after every run so a failed build never leaves stale artifacts for the next one.

Builds trigger automatically on push to `main` through the GitHub webhook.

<img width="1107" height="427" alt="04-jenkins-build" src="https://github.com/user-attachments/assets/156ed898-1f41-48e6-8fba-ccf2129af24b" />

<img width="1116" height="845" alt="05-ecs-service-healthy" src="https://github.com/user-attachments/assets/a137c534-7049-4ae9-b755-94cb073cba81" />


<br />

<h2>Load Testing and Scaling Results</h2>

```bash
siege -c 250 -t 2M http://devops-challenge-alb-1218229428.us-east-1.elb.amazonaws.com
```
<img width="1266" height="1514" alt="08-siege-results" src="https://github.com/user-attachments/assets/c02ab5fc-9bb6-45d8-8c9d-6d832fef61cf" />

250 concurrent users for two minutes. While it ran, task counts and CloudWatch CPU were watched in two additional terminals:

```bash
watch -n 10 "aws ecs describe-services \
  --cluster devops-challenge-cluster \
  --services devops-challenge-frontend-service devops-challenge-backend-service \
  --region us-east-1 \
  --query 'services[*].{Name:serviceName,Running:runningCount,Desired:desiredCount,Pending:pendingCount}'"
```

<b>Results</b>

<ul>
  <li><b>Frontend service:</b> scaled from 1 task to 4, the configured maximum</li>
  <li><b>Backend service:</b> remained at 1 task throughout</li>
  <li><b>Peak CPU:</b> approximately 99.9% on the frontend, well above the 50% threshold</li>
  <li><b>Time to first new task:</b> roughly 2 to 3 minutes after load began</li>
  <li><b>Time to maximum:</b> roughly 3 to 4 minutes total to reach 4 running tasks</li>
  <li><b>Scale-in:</b> back to 1 task about 14 to 15 minutes after Siege stopped, with connections drained gracefully before deregistration</li>
</ul>

The backend did not scale because Siege hit the ALB root path, which the default listener rule routes to the frontend target group. Only the frontend's own calls reach the backend, so backend CPU never approached the threshold. Driving backend scaling would require pointing the load test at `/api` directly.

<img width="1638" height="536" alt="06-scale-up-4-tasks" src="https://github.com/user-attachments/assets/19219a40-dc53-4e8d-8c15-79ad1dae81ee" />

<img width="1232" height="952" alt="07-cloudwatch-cpu" src="https://github.com/user-attachments/assets/151c6741-aa31-4e45-a05f-382ee56eac62" />

<img width="2048" height="781" alt="09-scale-in-events" src="https://github.com/user-attachments/assets/89f77085-9417-417f-8e4c-7d705dc5b9b8" />

<br />

<h2>Configuration</h2>

<ul>
  <li><b><code>frontend/src/config.js</code></b> — the URL the frontend calls for the backend. <code>http://localhost:8080</code> locally, the ALB DNS name in production.</li>
  <li><b><code>backend/config.js</code></b> — the origin allowed in the <code>Access-Control-Allow-Origin</code> CORS header. <code>http://localhost:3000</code> locally, the ALB DNS name in production.</li>
</ul>

Both values are baked into the image at build time. After changing either one, the image must be rebuilt — and if Docker serves a cached layer, rebuild with <code>--no-cache</code>.

<br />

<h2>Teardown</h2>

```bash
cd terraform
terraform destroy
```

If destroy stalls on subnet deletion, an ENI is still attached. Check for lingering ECS tasks or load balancer interfaces in the console and remove them, then re-run.

<br />

<h2>Troubleshooting Notes</h2>

<ul>
  <li><b><code>npm ci</code> fails</b> — Node version mismatch. Run <code>nvm use 16</code> first; the lockfile was generated under Node 16.</li>
  <li><b>Frontend Docker build fails with <code>ERR_OSSL_EVP_UNSUPPORTED</code></b> — the base image is a Node version newer than 16. Both Dockerfiles must use <code>node:16</code>.</li>
  <li><b>"Failed to fetch" in the browser</b> — <code>frontend/src/config.js</code> is pointing at <code>host.docker.internal</code>, which the browser cannot resolve. It must be <code>localhost</code> or the ALB DNS name; the browser makes that request, not the container.</li>
  <li><b>Docker build uses the old config</b> — a cached layer. Rebuild with <code>docker build --no-cache</code> after editing any file baked into the image.</li>
  <li><b>ALB returns the frontend HTML on <code>/api</code></b> — the listener rule path pattern must be <code>/api*</code>, not <code>/api/*</code>. The second pattern does not match <code>/api</code> itself.</li>
  <li><b>Terraform destroy stuck on subnet deletion</b> — a live ENI is still attached. Remove the lingering interface manually, then re-run.</li>
  <li><b>Terraform cross-region error on the key pair</b> — <code>mysql-keypair</code> exists only in <code>us-east-1</code>. Confirm the region and any stale state before applying.</li>
  <li><b>Do not commit <code>.terraform/</code> or <code>terraform.tfstate</code></b> — the provider binary alone will blow past GitHub's file size limit, and state files contain resource details that do not belong in Git.</li>
</ul>

<br />

<h2>Submission</h2>

<ul>
  <li>Private GitHub repository, shared with michaeltayo96@outlook.com as a collaborator</li>
  <li>Frontend URL and Jenkins URL with credentials provided in the submission form</li>
  <li>Jenkins infrastructure documented above as required by the challenge brief</li>
</ul>
