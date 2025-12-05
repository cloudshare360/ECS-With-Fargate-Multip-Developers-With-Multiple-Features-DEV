# Requirements

- **Developer isolation:** Each developer can deploy and test their own changes concurrently without overwriting others’ work.  
- **Branch-level environments:** One container (ECS service) per feature branch, with predictable naming, routing, and cleanup.  
- **Integration testing support:** Ability to run multiple developer features against shared downstream services (e.g., payment gateway) with configurable endpoints and test data.  
- **Environment alignment:** Maintain Dev and Test in a shared AWS account and Prod in a separate account, while ensuring consistent patterns across environments.  
- **Tooling continuity:** Keep GitLab CI/CD, Terraform, ECS with Fargate, Java 17 (Spring MVC, Servlets, JSP), and Tomcat as the delivery stack.  
- **Manual and automated paths:** Support AWS CLI-based manual spins for fast local testing and a CI pipeline for repeatability and auditability.  
- **Speed and feedback:** Reduce blocking and waiting caused by the current ~20-minute build/test cycle; enable parallel deployments.  
- **Governance and controls:** Enforce IAM least privilege, resource quotas, tagging, and cost controls for ephemeral branch environments.  
- **Consistency and reproducibility:** Standardize naming conventions, environment variables, image tagging strategy, and Terraform state layout to avoid drift.  
- **Routing and service discovery:** Provide unique endpoints per branch/service, DNS or path-based routing, and clear discovery for testers.  
- **Lifecycle management:** Automatic cleanup of branch environments on merge or inactivity to contain costs.  

---

# Environment background and context

- **Accounts and environments:** Prod is in a separate AWS account. Dev and Test share one AWS account. A single ECS cluster is defined per non-prod, with Dev deploying one task/service and Test deploying two tasks/services.  
- **Current pipeline:** GitLab CI/CD invokes Terraform to provision the ECS cluster and deploy the application. The monolithic Java 17 Spring MVC app (Servlets/JSP on Tomcat) is packaged and pushed. Each pipeline run takes around 20 minutes to build and test.  
- **Deployment behavior:** Dev pipeline deploys to one ECS task/service. Test pipeline deploys two tasks/services on the same cluster. Branch changes are applied to these shared services, causing overwrites.  
- **Developer workflow:** Developers have AWS CLI access. To avoid waiting, they manually create tasks/services, build Docker images, push to ECR, and update task definitions to test independently.  
- **Dependencies:** The application is monolithic but integrates with external systems like payment gateways and downstream applications, which must be accessible and configurable in Dev/Test.  
- **Feature concurrency:** Multiple developers may work on overlapping features (e.g., D1 on F1/F2/F3; D2 on F3/F4/F5), which demands parallel, isolated environments and controlled integration validation.  

---

# Problem statement

- **Overwrites and blocking:** A single shared Dev service means Developer 1’s deployment overwrites Developer 2’s environment, forcing sequential testing and creating delays.  
- **Slow feedback loop:** The ~20-minute pipeline run time magnifies blocking; developers wait for builds and for others to finish validation.  
- **Inconsistent manual paths:** Direct AWS CLI deployments bypass CI/CD and Terraform, causing configuration drift, inconsistent naming, and hard-to-reproduce environments.  
- **Lack of isolation:** Current Dev/Test service topology (one service in Dev, two in Test) cannot support multiple concurrent branch-level tests without interference.  
- **Integration friction:** No clear model to test multiple features against external services concurrently, with predictable routing, test data, and endpoint configuration.  
- **Governance gaps:** No automated lifecycle for ephemeral branch environments leads to orphaned resources, cost sprawl, and permissions risks.  
- **State and tagging issues:** Image tagging (e.g., “latest”) and shared Terraform state contribute to accidental overwrites and non-deterministic deployments across branches.  

---

# Explain the problem statement

## Detailed explanation

- **Shared mutable target causes contention:** The Dev pipeline deploys a single ECS service/task, so any branch deployment replaces the running container. This design assumes serial testing, which breaks in parallel feature development. Without per-branch services or namespaces, developers overwrite each other’s changes.  
- **Pipeline structure couples branches to one environment:** Terraform/CD define static services bound to Dev/Test, not to branches. When a branch builds, it targets the same service name and task definition family. Even with multiple tasks, they represent instances of the same service version, not separate branch-specific environments.  
- **Tagging and immutability gaps:** Using non-unique image tags (e.g., “latest” or shared version tags) and task definition names lets one branch silently redeploy over another. Lack of per-branch task definition revisions and distinct ECS services eliminates isolation.  
- **Manual CLI bypass introduces drift:** Developers create ad hoc tasks/services via CLI to gain isolation, but those resources aren’t tracked by Terraform. Over time, configuration diverges (CPU/memory, env vars, IAM, networking), making failures hard to debug and cleanup unreliable.  
- **Routing discovery is unclear:** Even if multiple services exist, without unique DNS names, paths, or service discovery records, testers can’t easily find or distinguish environments. This blocks parallel testing and increases coordination costs.  
- **Integration testing needs structured access:** External dependencies (payment gateways, downstream apps) require configurable endpoints and test data. Multiple branch environments hitting shared dependencies demand clear segmentation (sandbox credentials, tenant headers, mock/stub toggles) to avoid cross-test interference and flaky results.  
- **Lifecycle and cost are unmanaged:** Ephemeral environments must be created quickly and torn down on merge/PR close or inactivity. Without automation, resources linger—raising costs and complicating Terraform state.  
- **Permissions and policy not aligned:** Broad CLI permissions plus unmanaged ephemeral resources increase the risk of misconfigurations, leaked secrets, and unpredictable behavior. Least-privilege roles and guardrails are needed to enable speed without compromising control.  

## Summarized explanation

- **Root issue:** A single shared ECS service per environment means branch deployments overwrite each other, forcing serial testing.  
- **Consequences:** Long pipeline times, blocking between developers, manual bypasses causing drift, and unreliable integration tests.  
- **What’s missing:** Per-branch isolated services with unique image tags, routing, and lifecycle automation; controlled integration endpoints; and governance (IAM, tagging, cleanup).  
- **Outcome needed:** Ephemeral, branch-scoped environments enabling parallel developer testing and reliable integration validation, without configuration drift or cost sprawl.
