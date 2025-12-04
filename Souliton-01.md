# Aws ecs fargate multi-tenant dev environment for java spring mvc monolith

This is a pragmatic, reusable guide to let each developer spin up isolated containers per feature branch on ECS Fargate, test independently, and then perform integration testing with shared dependencies. It aligns with your current GitLab CI/CD + Terraform setup and supports Prod/Test/Dev accounts.

---

## 1. Requirements

- **AWS accounts and environments:**  
  - **Prod:** Dedicated AWS account.  
  - **Dev/Test:** Shared AWS account, separated by namespaces/tags.  
- **Core services:**  
  - **ECS Fargate:** Cluster per environment; services per branch/feature.  
  - **ECR:** Container registry for images.  
  - **ALB + Target Groups:** Routing to branch-specific services.  
  - **CloudWatch Logs:** Aggregated logs per service/task.  
  - **VPC, subnets, SGs:** Private tasks + ALB in public subnets.  
  - **Parameters/Secrets:** SSM Parameter Store and Secrets Manager for configs.  
- **Pipeline & infra tooling:**  
  - **GitLab CI/CD:** Build/test/package/push/deploy.  
  - **Terraform:** Infra modules (network, ECS, ALB, ECR, IAM).  
- **Application stack:**  
  - **Java 17, Spring MVC, Servlets, JSP:** Monolithic app packaged into a Tomcat image.  
- **Access & permissions:**  
  - **AWS CLI:** Developers can launch/update their own branch services.  
  - **IAM:** Least-privilege roles for CI/CD and developer CLI with scoped resource tags.  
- **Naming convention:**  
  - Services and tasks are named by developer and branch:  
    - Example: app-dev-d1-f1, app-dev-d2-f3, app-test-featureX, app-int-shared.

---

## 2. Environment background and context

- **Current setup:**  
  - GitLab pipeline triggers Terraform to create an ECS cluster and deploy a single Dev service and two Test tasks on the same cluster. Dev deployments overwrite each other; Test has two tasks but still shared. Builds take ~20 minutes, serializing developer testing.
- **Constraints:**  
  - Shared Dev/Test account; Prod isolated.  
  - Monolith depends on payment gateway and downstream apps.  
  - Developers work on multiple features concurrently (e.g., d1: f1, f2, f3; d2: f3, f4, f5).  
- **Goal:**  
  - Each developer can deploy their own branch-specific container without blocking others.  
  - Support independent functional testing and collaborative integration testing.  
  - Optionally allow separate per-developer clusters, but prefer single cluster with many services/tasks for cost/control unless isolation is required.

---

## 3. Problem statement

- **Dev overwrite:** A single Dev service means each pipeline run replaces the current tasks; developers wait for each other.  
- **Long build cycles:** 20-minute builds delay feedback and increase contention.  
- **Integration pain:** Validating cross-feature integration requires merging into the shared Dev branch, causing churn and risk.  
- **Dependency coupling:** External services (payment gateway, downstream apps) complicate isolated testing and require controlled integration targets.  
- **Limited environment parallelism:** Lack of branch- or developer-scoped services prevents true multi-tenant Dev usage.

---

## 4. Solution

### High-level approach

- **Ephemeral, branch-scoped services on ECS Fargate:**
  - For each feature branch, create a dedicated ECS service and task definition, deployed to a shared Dev cluster.  
  - Naming: app-dev-<developer>-<branch> (sanitized).  
  - Each service registers to an ALB target group. Use path-based routing or subdomain routing (e.g., dev.example.com/d1/f1 or f1.d1.dev.example.com).

- **Shared integration environment:**
  - Maintain a persistent “integration” namespace (e.g., app-int) in Dev or Test for merged features or coordinated test sessions.  
  - Only merge feature branches into “integration” when ready to test interactions.

- **Terraform modules for repeatability:**
  - Network (VPC, subnets, SG).  
  - ECS cluster.  
  - ECR repository.  
  - ALB + target groups + listeners.  
  - ECS service module parameterized by service_name, image_tag, env vars, scaling.  
  - IAM roles with tag-based access control.

- **GitLab CI/CD pipelines for automation:**
  - Per-branch pipeline builds image and deploys branch service via Terraform with variables derived from branch name and developer ID.  
  - On merge or delete, tear down branch services automatically to control cost.  
  - Promotion jobs deploy to the integration environment, then Test, then Prod.

- **Manual AWS CLI fallback:**
  - Developers can build locally, push to ECR, register a task definition, and create/update their own ECS service using a well-defined naming convention.  
  - IAM policies restrict them to manage only resources tagged with their developer ID.

- **Configuration isolation:**
  - Use SSM parameters per service to inject environment-specific URLs, API keys, and feature flags.  
  - For external dependencies, support “sandbox” endpoints during branch testing; switch to shared integration endpoints when needed.

---

## 5. Step-by-step instructions: Manual approach using AWS CLI

### Prerequisites

- **Installed tools:** AWS CLI v2, Docker, Java 17, Maven/Gradle.  
- **AWS credentials:** Access to Dev/Test account with policy allowing ECR, ECS, ALB target registration, CloudWatch Logs; scoped via tags.  
- **Environment variables:**
  - AWS_ACCOUNT_ID, AWS_REGION, DEV_ID (e.g., d1), BRANCH (e.g., f1), APP_NAME=app, CLUSTER_NAME=dev-cluster.
  - BASE_DOMAIN=dev.example.com (optional if using path-based routing).

### 1. Build and push the image to ECR

- **Create ECR repo (once per app):**
  - aws ecr create-repository --repository-name app --image-scanning-configuration scanOnPush=true
- **Authenticate Docker to ECR:**
  - aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
- **Build image:**
  - docker build -t app:$DEV_ID-$BRANCH .
- **Tag for ECR:**
  - docker tag app:$DEV_ID-$BRANCH $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/app:$DEV_ID-$BRANCH
- **Push:**
  - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/app:$DEV_ID-$BRANCH

### 2. Prepare task definition JSON

- **task-def.json (Tomcat + Java 17):**
  - Use image: $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/app:$DEV_ID-$BRANCH
  - CPU/memory: 0.5 vCPU, 1–2 GB (adjust as needed).
  - Env vars: SPRING_PROFILES_ACTIVE=dev-$DEV_ID-$BRANCH, DEPENDENCY_URLS, etc.
  - Log configuration: awslogs group /ecs/$APP_NAME-$DEV_ID-$BRANCH.

Example container definition snippet:
```json
{
  "family": "app-dev-d1-f1",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::<account>:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::<account>:role/appTaskRole",
  "containerDefinitions": [{
    "name": "app",
    "image": "<account>.dkr.ecr.<region>.amazonaws.com/app:d1-f1",
    "portMappings": [{ "containerPort": 8080, "protocol": "tcp" }],
    "essential": true,
    "environment": [
      { "name": "SPRING_PROFILES_ACTIVE", "value": "dev-d1-f1" },
      { "name": "PAYMENT_API_BASE", "value": "https://sandbox-payments.example.com" }
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/app-dev-d1-f1",
        "awslogs-region": "<region>",
        "awslogs-stream-prefix": "ecs"
      }
    }
  }]
}
```

### 3. Register the task definition

- aws ecs register-task-definition --cli-input-json file://task-def.json

### 4. Create target group for ALB and routing rule

- **Create TG:**
  - aws elbv2 create-target-group --name app-dev-$DEV_ID-$BRANCH --protocol HTTP --port 8080 --vpc-id <vpc-id> --target-type ip --health-check-path /health
- **Add listener rule (path-based):**
  - aws elbv2 create-rule --listener-arn <listener-arn> --priority <unique> --conditions Field=path-pattern,Values=/$DEV_ID/$BRANCH/* --actions Type=forward,TargetGroupArn=<tg-arn>

Alternatively, for subdomain routing, create a DNS record (CNAME) and an ALB listener rule matching host header: $BRANCH.$DEV_ID.$BASE_DOMAIN.

### 5. Create ECS service

- aws ecs create-service \
  --cluster $CLUSTER_NAME \
  --service-name $APP_NAME-dev-$DEV_ID-$BRANCH \
  --task-definition $APP_NAME-dev-$DEV_ID-$BRANCH \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[<private-subnet-ids>],securityGroups=[<sg-id>],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=<tg-arn>,containerName=app,containerPort=8080" \
  --tags "key=owner,value=$DEV_ID" "key=branch,value=$BRANCH" "key=env,value=dev"

### 6. Test

- **URL (path-based):** https://$BASE_DOMAIN/$DEV_ID/$BRANCH/  
- **Logs:** CloudWatch Logs group /ecs/app-dev-$DEV_ID-$BRANCH.  
- **Iterate:** Rebuild/push and run aws ecs update-service --force-new-deployment to refresh tasks.

### 7. Teardown when done

- aws ecs delete-service --cluster $CLUSTER_NAME --service $APP_NAME-dev-$DEV_ID-$BRANCH --force
- aws elbv2 delete-rule --rule-arn <rule-arn>
- aws elbv2 delete-target-group --target-group-arn <tg-arn>
- Optionally deregister old task definitions or let them age out.

---

## 6. Automation using gitlab cicd and terraform for ecs fargate

### Terraform structure

- **Modules:**
  - **network/**: vpc, subnets, route tables, security groups.  
  - **ecr/**: repo and lifecycle rules.  
  - **alb/**: ALB, listeners (80/443), ACM, target groups, rules.  
  - **ecs/**: cluster, task execution role, task role, service.  
- **ecs_service module variables:**
  - service_name, cluster_arn, task_cpu, task_memory, image, env_vars, desired_count, subnets, security_groups, target_group_arn, tags.

Example ecs_service module usage:
```hcl
module "ecs_service_branch" {
  source            = "./modules/ecs_service"
  service_name      = "app-dev-${var.dev_id}-${var.branch}"
  cluster_arn       = aws_ecs_cluster.dev.arn
  image             = "${var.account_id}.dkr.ecr.${var.region}.amazonaws.com/app:${var.dev_id}-${var.branch}"
  task_cpu          = 512
  task_memory       = 1024
  env_vars = {
    SPRING_PROFILES_ACTIVE = "dev-${var.dev_id}-${var.branch}"
    PAYMENT_API_BASE       = var.payment_api_base
  }
  desired_count     = 1
  subnets           = var.private_subnet_ids
  security_groups   = [aws_security_group.app.id]
  target_group_arn  = module.alb_branch.tg_arn
  tags = {
    owner  = var.dev_id
    branch = var.branch
    env    = "dev"
  }
}
```

Example ALB rule module:
```hcl
module "alb_branch" {
  source        = "./modules/alb_rule"
  alb_arn       = aws_lb.dev.arn
  listener_arn  = aws_lb_listener.http.arn
  path_pattern  = "/${var.dev_id}/${var.branch}/*"
  priority      = var.priority
  vpc_id        = aws_vpc.main.id
  target_port   = 8080
}
```

- **IAM with tag-based permissions:**
  - Developers can manage only ECS services, target groups, rules, and task definitions tagged owner=<their_id>.  
  - CI/CD role can manage all for dev/test; restricted in prod.

### GitLab CI/CD pipeline

- **Variables and naming:**
  - DEV_ID derived from user or protected variable.  
  - BRANCH from CI_COMMIT_REF_SLUG.  
  - ENV=dev/test/prod based on branch/protected merges.  
  - IMAGE_TAG="${DEV_ID}-${BRANCH}".

- **.gitlab-ci.yml (simplified):**
```yaml
stages:
  - build
  - test
  - package
  - deploy
  - cleanup

variables:
  AWS_REGION: us-east-1
  APP_REPO: "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/app"

before_script:
  - export DEV_ID="${DEV_ID:-$GITLAB_USER_LOGIN}"
  - export BRANCH="${CI_COMMIT_REF_SLUG}"
  - echo "DEV_ID=$DEV_ID BRANCH=$BRANCH"

build:
  stage: build
  script:
    - docker build -t app:$DEV_ID-$BRANCH .
    - aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
    - docker tag app:$DEV_ID-$BRANCH $APP_REPO:$DEV_ID-$BRANCH
    - docker push $APP_REPO:$DEV_ID-$BRANCH
  artifacts:
    expire_in: 1 week

deploy_dev_branch:
  stage: deploy
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^feature\/.+$/'
  script:
    - terraform -chdir=infra init
    - terraform -chdir=infra apply -auto-approve \
        -var="env=dev" \
        -var="dev_id=$DEV_ID" \
        -var="branch=$BRANCH" \
        -var="image_tag=$DEV_ID-$BRANCH"
  environment:
    name: "dev/$DEV_ID/$BRANCH"
    url: "https://dev.example.com/$DEV_ID/$BRANCH/"

integration_deploy:
  stage: deploy
  needs: ["deploy_dev_branch"]
  rules:
    - if: '$CI_PIPELINE_SOURCE == "pipeline" && $INTEGRATE == "true"'
  script:
    - terraform -chdir=infra apply -auto-approve \
        -var="env=integration" \
        -var="service_name=app-int" \
        -var="image_tag=$DEV_ID-$BRANCH"
  environment:
    name: "integration"
    url: "https://int.dev.example.com/"

cleanup_branch:
  stage: cleanup
  rules:
    - if: '$CI_PIPELINE_SOURCE == "push"'
      when: manual
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: never
  script:
    - terraform -chdir=infra destroy -auto-approve \
        -var="env=dev" \
        -var="dev_id=$DEV_ID" \
        -var="branch=$BRANCH"
```

- **Test and Prod promotions:**
  - Use protected branches/tags (e.g., merge to develop triggers Test deploy with 2 tasks; merge to main triggers Prod deploy with change approval gates).  
  - In Test, allow multiple tasks per service for basic scaling or dual instances for parallel test sessions.  
  - In Prod, immutable image tags and explicit blue/green or rolling deployment via ECS deployment controller.

### Integration testing workflow

- **Branch validation:** Developers deploy branch services and test independently with sandbox dependencies.  
- **Integration session:**  
  - Merge feature branches into an “integration” branch or trigger “integration_deploy” with specific pairs of branch images (controlled via variables).  
  - Point integration services to shared integration endpoints for payment/downstream.  
  - Run end-to-end tests; capture logs and metrics.  
- **Tear down:** Stop integration services when complete to save cost.

---

## Sample dockerfile and app notes

- **Dockerfile (Tomcat + JSPs/Servlets):**
```dockerfile
FROM eclipse-temurin:17-jdk
RUN apt-get update && apt-get install -y wget && rm -rf /var/lib/apt/lists/*
ENV CATALINA_HOME=/usr/local/tomcat
RUN wget -q https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.82/bin/apache-tomcat-9.0.82.tar.gz -O /tmp/tomcat.tar.gz \
 && tar -xzf /tmp/tomcat.tar.gz -C /usr/local \
 && mv /usr/local/apache-tomcat-9.0.82 $CATALINA_HOME \
 && rm /tmp/tomcat.tar.gz
COPY target/app.war $CATALINA_HOME/webapps/ROOT.war
EXPOSE 8080
CMD ["/usr/local/tomcat/bin/catalina.sh", "run"]
```

- **Spring MVC Hello World:** Map “/” to JSP view, add “/health” controller returning 200 OK.

---

## Operational considerations

- **Cost control:** Auto-destroy branch services after inactivity (e.g., nightly job for branches older than N days).  
- **Security:** Restrict developer IAM with resource tags; require TLS on ALB.  
- **Observability:**  
  - CloudWatch Logs per service/task, with log retention policy.  
  - ALB access logs to S3.  
  - Optional X-Ray for downstream tracing.  
- **Limits and quotas:** Ensure ALB rule/target group counts are within quotas; implement path-based routing to avoid subdomain explosion.  
- **Naming and sanitization:** Replace non-alphanumeric characters in branch names with dashes; cap length to avoid AWS name limits.  
- **Rollback:** Keep previous image tags; use update-service with force-new-deployment to revert quickly.

---

## Inputs that may be missing or needing confirmation

- **Domains and certificates:** Do you have dev.example.com with ACM cert in the Dev/Test account?  
- **VPC/Subnet details:** IDs for private subnets and desired networking rules.  
- **Security groups:** Inbound rules for ALB to reach tasks; egress rules for tasks to reach dependencies.  
- **Dependency endpoints:** Sandbox vs integration URLs for payment gateway and downstream apps.  
- **IAM strategy:** Will you enforce tag-based RBAC for developers? Provide developer IDs list and mapping.  
- **GitLab runners:** Where do runners execute (Docker-in-Docker, shared runners)? Permissions to assume AWS roles?  
- **Terraform state:** Backend configuration (S3/DynamoDB) per environment.  
- **Priority allocation:** ALB listener rule priority scheme for branch routing (auto-increment or hash-based).

If you share these, I can tailor the Terraform and CI snippets to your exact environment and naming.
