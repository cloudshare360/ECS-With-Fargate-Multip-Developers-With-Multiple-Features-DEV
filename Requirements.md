
# Ephemeral environments for ECS Fargate with Terraform and GitLab CI/CD

This document analyzes your current setup and proposes a structured approach so each developer can spin up isolated containers per branch for independent feature testing, while preserving a clean path for shared integration testing. It is focused on a monolithic Java 17 Spring MVC app (JSP/Servlets) deploying to ECS Fargate across Dev/Test/Prod, with GitLab CI/CD invoking Terraform.

---

## Requirements

- **Isolated developer environments:** Each developer can deploy and test feature branches in their own container(s) without overriding others.
- **Branch-based deployments:** One container per branch (or feature), with predictable naming and automated lifecycle (create/update/destroy).
- **Shared integration testing:** A controlled environment to validate combined changes against external systems (payment gateway and downstream apps).
- **No Prod drift:** Prod account isolated; Test/Dev in shared account with guardrails to avoid accidental cross-environment changes.
- **Repeatable automation:** GitLab CI/CD + Terraform pipelines to create/update ECS services/tasks per branch and clean them up when merged/closed.
- **Manual option:** AWS CLI workflow for developers to create/update ephemeral services when needed (timeboxed and discoverable).
- **Minimal blast radius:** Quotas, autoscaling limits, and namespaces to prevent runaway costs or resource contention.
- **Observability:** Logs, metrics, and traceability aligned to branch/service identifiers; simple access patterns.
- **Routing and discovery:** Clear URLs or ports per branch to reach each container, with lightweight routing (ALB rules or per-service ALB).
- **Security & compliance:** Principle of least privilege on IAM; scoped access for developers; secrets management via AWS SSM Parameter Store or Secrets Manager.
- **Lifecycle hygiene:** Automatic teardown of ephemeral environments to control spend.

---

## Environment background and context

- **Accounts:** Prod in a dedicated AWS account; Dev and Test share a separate AWS account.
- **ECS topology today:** Single ECS cluster for Dev/Test. Dev currently uses one task/container; Test uses two tasks/containers on the same cluster.
- **Build pipeline:** GitLab CI/CD triggers Terraform to create the ECS cluster and deploy. A full run takes ~20 minutes.
- **Application:** Java 17 monolith (Spring MVC, JSP, Servlets) that integrates with external payment and downstream systems.
- **Developer pattern:** Multiple concurrent features per developer (e.g., d1: f1, f2, f3; d2: f3, f4, f5). Today’s pipeline overrides shared Dev tasks, causing contention and waiting.

---

## Problem statement

- **Contention in Dev:** A single Dev task/service is overwritten by each pipeline run, forcing developers to wait and re-run, slowing feedback.
- **Slow iteration:** ~20-minute pipeline cycles per change create friction for quick testing.
- **Limited isolation:** Test has more capacity (two tasks), but still insufficient for parallel feature validation.
- **Risk of drift:** Manual CLI updates can bypass the pipeline, risking inconsistent state and making cleanup difficult.
- **Integration testing gap:** No clear venue to validate multiple features together against external systems without stepping on each other.

---

## Solution overview

- **Ephemeral per-branch services:** For Dev, create an ECS service per branch on the shared cluster (or per developer namespace), with unique task definitions and ALB routing. Naming convention drives automation and discovery.
- **Stable shared environments:** Maintain “integration” (Test) as a stable shared environment with curated promotion rules from ephemeral Dev branches; Prod remains isolated.
- **Routing via ALB rules:** Use path-based routing or host-based routing to map branch services to distinct endpoints (e.g., /env/{branch} or {branch}.dev.company.com).
- **Terraform modules:** Parameterize modules to create ECS service, task definition, target group, ALB rule, log group, and SSM/Secrets bindings per branch.
- **CI/CD branches to services:** GitLab pipelines create/update on push, and destroy on branch deletion/merge. Developers can manually trigger ephemeral deploy/update jobs when needed.
- **Manual CLI safety rails:** Provide a documented CLI path for developers with guardrails (namespaces, tags, TTL, resource caps).
- **Observability tagging:** Standardize tags/labels (service=app, env=dev-branch, owner=developer, feature=fX) for CloudWatch logs, metrics, and cost tracking.
- **Integration testing promotion:** Merge or tag branches to promote selected artifact/images to Test; orchestrate external dependency stubs vs. real endpoints via configs.

---

## Architecture options

### Option A: Single ECS cluster with ALB routing per branch (recommended for Dev)
- **Pros:** Lower infra overhead; fast to spin per-branch services; simpler Terraform state; shared capacity with autoscaling.
- **Cons:** ALB rules scale with number of branches; requires diligent cleanup.

### Option B: Per-developer sub-clusters within the shared Dev account
- **Pros:** Strong isolation; easier resource caps per developer; fewer ALB rules if per-cluster ALB.
- **Cons:** More Terraform state and IAM complexity; higher cost and management overhead.

### Option C: Namespaced services with internal NLB and service discovery (advanced)
- **Pros:** Reduced ALB rule sprawl; service discovery via Cloud Map; good for microservices.
- **Cons:** More complex routing; monolith may not benefit; extra steps for external access.

> Quick verdict: Option A strikes the best balance for a monolithic app with many ephemeral branches. Use Option B selectively for heavy parallel work or noisy neighbors.

---

## Conventions and workflows

#### Naming conventions
- **Cluster:** ecs-dev-shared, ecs-test-shared, ecs-prod
- **Service (Dev ephemeral):** app-dev-{developer}-{branch} (e.g., app-dev-d1-f3)
- **Task definition family:** app-dev-{developer}-{branch}
- **ALB path rule:** /env/{developer}/{branch}
- **Log group:** /ecs/app/dev/{developer}/{branch}
- **Tags:** env=dev, scope=ephemeral, owner={developer}, branch={branch}, feature={fx}

#### Image tags
- **Pattern:** {branch}-{shortsha} for ephemeral; release-{version} for Test/Prod
- **Registry:** ECR repositories per app (e.g., ecr://app-monolith)

#### Secrets and configs
- **Dev ephemeral:** SSM parameters namespaced by branch: /app/dev/{developer}/{branch}/{key}
- **Test:** /app/test/{key} with controlled overrides
- **Prod:** /app/prod/{key} locked down

#### Lifecycles
- **Create/update:** On push to feature branches, ephemeral service is created/updated.
- **Destroy:** On branch deletion or merge to development/main, ephemeral teardown runs.
- **TTL sweep:** Nightly job tags and tears down stale services (e.g., >7 days without commits).

---

## Developer manual approach (AWS CLI guarded workflow)

- **Prereqs:** IAM role with limited permissions to create/update ECS services only under app-dev-{developer}-*; access to ECR; SSM read for dev namespace; CloudWatch Logs creation.
- **Build and push:**
  - **Action:** Build Docker locally for the monolith; tag with {branch}-{shortsha}; push to ECR.
- **Register task definition:**
  - **Action:** Create a task definition family app-dev-{developer}-{branch} referencing the image, CPU/memory, env vars, secrets (SSM), and log configuration.
- **Create or update service:**
  - **Action:** Upsert ECS service app-dev-{developer}-{branch} on ecs-dev-shared with desiredCount=1 (or more if needed), attach to target group.
- **Add ALB rule:**
  - **Action:** Create a path-based rule /env/{developer}/{branch} to the service’s target group; ensure health checks match the app’s path.
- **Observe:**
  - **Action:** Tail CloudWatch logs for the log group; check target group health; hit the routed URL for smoke tests.
- **Cleanup:**
  - **Action:** Delete service and task definition revisions; remove ALB rule and target group when done.

> Guardrails: Enforce resource tags, set per-developer quotas, and restrict IAM to only app-dev-{developer}-namespaced resources. Provide a one-command wrapper to avoid drift.

---

## Automation approach (GitLab CI/CD and Terraform)

#### Terraform modules
- **Core module:** ECS service + task definition + target group + ALB rule + log group + SSM/Secrets wiring.
- **Inputs:** developer, branch, image tag, CPU/memory, environment, desiredCount, health check path, VPC/subnets/security groups.
- **Outputs:** URL endpoint, log group, service ARN, task family.

#### Pipelines
- **Feature branch pipeline (Dev ephemeral):**
  - **Stages:** build → unit-test → image-push → plan → apply → smoke-test
  - **Behavior:** Creates/updates app-dev-{developer}-{branch}; posts endpoint in job summary.
- **Destroy pipeline (on branch delete/merge):**
  - **Stages:** plan-destroy → destroy
  - **Behavior:** Tears down service, ALB rule, target group, and orphaned task definitions.
- **Integration pipeline (Test):**
  - **Trigger:** Merge to development or tagged integration release.
  - **Behavior:** Builds release image, applies Test services (two tasks), runs integration tests against payment/downstream with environment toggles.
- **Prod pipeline:**
  - **Trigger:** Release/tag from Test; manual approval gates.
  - **Behavior:** Deploys to isolated Prod account via cross-account role; immutable images; blue/green or rolling.

#### State management
- **Remote backend:** Terraform state per environment and per developer-branch workspace (e.g., tfstate key dev/{developer}/{branch}.tfstate).
- **Locks:** Enforce state locking to prevent collisions.

#### Observability and reporting
- **Logs and metrics:** CloudWatch with tags; GitLab job annotations include endpoint, ALB rule ID, and log group.
- **Cost control:** Scheduled cleanup; budgets/alerts; desiredCount defaults to 1 in Dev.

---

## Integration testing strategy

- **Promotion model:** Only selected branches or the development branch produce artifacts deployed to Test; ephemeral Dev branches do not hit shared Test directly.
- **Config switches:** Inject environment-specific endpoints (payment gateway sandboxes, downstream mocks vs. real) via SSM parameters per environment.
- **Contract tests:** Validate integrations at the boundary before merging to development; smoke tests run post-deploy in Test.
- **Data isolation:** Use distinct test datasets or tenant identifiers per test run to avoid interference.
- **Gatekeeping:** Require passing ephemeral Dev checks and contract tests before integration deploy; provide rollback hooks.

---

## Risks and mitigations

- **ALB rule sprawl:** Limit concurrent ephemeral services per developer; nightly cleanup; consider host-based routing if path rules grow.
- **State drift via CLI:** Wrap CLI with scripts that call the same Terraform module or enforce tags/names; audit changes.
- **Secrets exposure:** Centralize in SSM/Secrets Manager; disallow hardcoded secrets; scope IAM to environment namespaces.
- **Costs:** Enforce TTL, quotas, and desiredCount caps; monitor with tags and budgets.
- **External dependencies:** Use sandbox endpoints and resilient test data to prevent side effects.

---

## Missing inputs to finalize implementation

- **VPC details:** VPC ID, private/public subnets, route setup, security groups for Dev/Test/Prod.
- **ALB setup:** Existing ALB ARN(s) per environment, listener ports, certificate ARNs if using HTTPS.
- **Service discovery choice:** Path vs. host routing; desired DNS patterns (e.g., {branch}.dev.company.com).
- **Resource sizing:** CPU/memory per task, autoscaling policies, expected concurrency.
- **Secrets layout:** Exact SSM/Secrets parameter names and hierarchy for Dev/Test/Prod.
- **IAM model:** Roles and boundaries for developers, CI/CD, and Terraform; cross-account role ARNs for Prod.
- **Artifact registry:** ECR repository names and lifecycle policies.
- **Test harness:** Integration test suites, stubs/mocks availability, data requirements.
- **Cleanup policy:** TTL duration, conditions for auto-destroy, exemptions.

If you provide these details, I can turn this analysis into concrete Terraform modules, CI/CD jobs, and a safe CLI wrapper.
