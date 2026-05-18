# Java Spring Boot on AWS ECS Fargate

A production-style deployment of a Java Spring Boot application to AWS ECS Fargate.
The GitHub Actions pipeline automatically builds the app with Maven, runs tests,
builds a Docker image, and pushes it to AWS ECR. The application runs on AWS Fargate
behind an Application Load Balancer with a real public URL.

This project demonstrates the full container deployment lifecycle on AWS — from
source code to a live internet-accessible endpoint — without managing any servers.

---

## Architecture

```
Developer pushes code to GitHub
         ↓
GitHub Actions Pipeline:
  Maven compile + test
  Docker two-stage build (500MB builder → 89MB production image)
  Push to AWS ECR (private container registry)
         ↓
AWS ECS Fargate:
  Task pulls image from ECR via NAT Gateway
  Container runs in private subnet (secure, no direct internet access)
         ↓
Application Load Balancer (public facing)
  Receives internet traffic on port 80
  Health checks against /health endpoint
  Forwards to Fargate container on port 8080
         ↓
Live URL: http://your-alb-dns.ap-south-1.elb.amazonaws.com
```

---

## Project Structure

```
java-springboot-fargate/
├── src/
│   ├── main/
│   │   ├── java/com/portfolio/springbootapp/
│   │   │   ├── SpringbootappApplication.java   # Spring Boot entry point
│   │   │   └── HelloController.java            # REST API endpoints
│   │   └── resources/
│   │       └── application.properties          # App configuration
│   └── test/
│       └── java/com/portfolio/springbootapp/
│           └── SpringbootappApplicationTests.java  # Context load test
├── .github/
│   └── workflows/
│       └── ci.yml                              # CI/CD pipeline
├── Dockerfile                                  # Two-stage build
├── pom.xml                                     # Maven project manifest
└── README.md
```

---

## API Endpoints

| Endpoint | Method | Response |
|----------|--------|----------|
| `/` | GET | `{"status": "ok", "message": "Hello from Spring Boot on AWS Fargate", "service": "springboot-app"}` |
| `/health` | GET | `{"status": "healthy", "service": "springboot-app"}` |
| `/actuator/health` | GET | Detailed Spring Boot health information |

The `/health` endpoint is used by the ALB Target Group for health checks every 30 seconds.
If it fails to respond, the task is deregistered from the load balancer and replaced.

---

## How to Run Locally

**Prerequisites:** Java 17, Maven 3.9+

```bash
git clone git@github.com:mohammedkhaiserulla/java-springboot-fargate.git
cd java-springboot-fargate
mvn spring-boot:run
```

Open browser at `http://localhost:8080`

**Run with Docker:**
```bash
docker build -t springboot-app:local .
docker run -p 8080:8080 springboot-app:local
```

**Run tests:**
```bash
mvn test
```

---

## CI/CD Pipeline

The GitHub Actions pipeline has two jobs:

**Job 1 — Build and Test**
Runs on every push and pull request. Sets up Java 17 with Maven dependency caching,
runs `mvn clean verify` which compiles, tests, and packages the application. If any
test fails the pipeline stops and the build job never runs.

**Job 2 — Build and Push to ECR**
Runs only on pushes to main after tests pass. Configures AWS credentials from GitHub
Secrets, logs into ECR, builds the Docker image using the two-stage Dockerfile, and
pushes with two tags — `:latest` and the Git commit hash for traceability.

### GitHub Secrets Required

| Secret | Purpose |
|--------|---------|
| `AWS_ACCESS_KEY_ID` | IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret key |

---

## AWS Infrastructure

The application runs on the following AWS resources in `ap-south-1` (Mumbai):

| Resource | Purpose |
|----------|---------|
| ECR Repository | Private Docker image registry |
| VPC | Isolated network with public and private subnets |
| NAT Gateway | Allows private subnet tasks to pull images from ECR |
| Application Load Balancer | Public entry point, distributes traffic |
| Target Group | Routes ALB traffic to healthy Fargate tasks |
| ECS Cluster | Logical grouping of Fargate tasks |
| Task Definition | Blueprint — image, CPU, memory, ports, logging |
| ECS Service | Ensures desired number of tasks always running |
| CloudWatch Log Group | Container log storage |

---

## Deployment

After the pipeline pushes a new image to ECR, update the ECS Service:

```bash
aws ecs update-service \
  --cluster springboot-cluster \
  --service springboot-service \
  --force-new-deployment \
  --region ap-south-1
```

ECS performs a rolling update — starts new tasks with the updated image before
stopping old ones. Zero downtime deployment.

---

## Cost Considerations

This setup costs approximately 3-5 rupees per hour when running:

| Resource | Cost |
|----------|------|
| Fargate (0.25 vCPU, 0.5GB) | ~3 rupees/hour |
| ALB | ~1-2 rupees/hour |
| NAT Gateway | ~4 rupees/hour |
| ECR storage | Negligible |

**Always destroy resources after testing to avoid ongoing charges.**

---

## Production Considerations

**CPU allocation** — 0.25 vCPU causes slow Spring Boot startup (~40-60 seconds).
Production deployments use minimum 0.5 vCPU. More CPU means faster startup and
better request handling under load.

**Health check grace period** — must be set higher than the app startup time.
If the ALB starts health checking before Spring Boot finishes loading, tasks are
marked unhealthy and replaced in a loop. Set to at least 120 seconds for Spring Boot.

**Auto-deployment** — in production the pipeline includes an automatic ECS service
update step after pushing the image. No manual intervention needed.

**VPC endpoints** — adding ECR VPC endpoints eliminates NAT Gateway data transfer
costs by keeping ECR traffic inside the AWS network. Significant cost saving at scale.

**Multiple tasks** — production runs minimum 2 tasks across 2 availability zones
for high availability. If one AZ goes down, the other continues serving traffic.

**Auto scaling** — ECS Service Auto Scaling adjusts task count based on CPU or
request count metrics from CloudWatch. Handles traffic spikes automatically.

---

## Tech Stack

- **Java 17** — LTS version, stable and security-patched
- **Spring Boot 3.5** — Java web framework with embedded Tomcat
- **Maven 3.9** — Java build and dependency management tool
- **Docker** — two-stage containerisation
- **GitHub Actions** — CI/CD automation
- **AWS ECR** — private container image registry
- **AWS ECS Fargate** — serverless container runtime
- **AWS ALB** — Application Load Balancer
- **AWS CloudWatch** — container log aggregation