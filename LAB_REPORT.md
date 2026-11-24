COMPUTER SCIENCE DEPARTMENT • 5TH-YEAR SOFTWARE ENGINEERING
DEVOPS & CLOUD COMPUTING (LAB NO. 07)
Building an Automated CI/CD Pipeline with Jenkins, Docker, and GitHub

1. Deliverable 1: GitHub Repository & Project Structure
As mandated by Part 21, the source code, automated test suite, Dockerfile, and Jenkinsfile were prepared, initialized under Git version control, and published to a public GitHub repository. This enables Jenkins to automatically checkout and build the pipeline without requiring private access credentials.

SUBMISSION METADATA & GITHUB ACCESS
- Repository URL: (push this folder to GitHub, then replace this line with the clone URL)
- Default Branch: main
- Visibility: Public (HTTPS cloneable without authentication tokens)
- Tested Jenkins URL: http://localhost:8080
- Target Deployment URL: http://localhost:8081
- Internal Microservice Port: 8000 (Gunicorn WSGI)

1.1 Project Directory Hierarchy
The application follows a clean, modular structure separating application entry points, dependency manifests, container specifications, pipeline logic, and automated tests:

jenkins-flask-lab/
├── app.py                  # Flask web application exposing / and /health
├── requirements.txt        # Python dependency manifest (Flask, pytest, gunicorn)
├── Dockerfile              # Multi-layer Docker build specification (python:3.12-slim)
├── Jenkinsfile             # Declarative Jenkins CI/CD pipeline definition
├── pytest.ini              # Ensures pytest can import app.py
├── README.md               # Repository documentation and usage guidelines
└── tests/
    └── test_app.py         # Pytest test suite asserting status codes and JSON payloads

1.2 Source Code of All Project Artifacts
See the files in this repository: `app.py`, `requirements.txt`, `tests/test_app.py`, `Dockerfile`, `Jenkinsfile`.

2. Deliverable 2: Comprehensive Pipeline Report & Stage Descriptions
Continuous Integration and Continuous Delivery (CI/CD) represents the operational backbone of modern DevOps engineering. In this laboratory, we established a declarative Jenkins Pipeline that automates the transition of code commits from GitHub into verified, running Docker containers on the host machine. The pipeline orchestrates five sequential stages, each acting as a deterministic checkpoint.

2.1 System Architecture & Information Flow
Core Pipeline Topology: The architecture connects four primary components into a unified feedback loop:
1. GitHub SCM (Source Control): Serves as the Single Source of Truth (SSoT) containing application source code, Docker packaging recipes, test assertions, and the Jenkinsfile.
2. Jenkins Automation Server: Monitors Git commits, provisions ephemeral workspaces, reads the declarative Jenkinsfile, and executes stages sequentially on an agent equipped with Docker socket access.
3. Docker Engine: Builds reproducible, layered application images tagged dynamically with Jenkins build numbers, executes isolated unit tests in ephemeral containers, and deploys the production container.
4. Flask Application & Gunicorn WSGI: Lightweight Python microservice serving HTTP requests at port 8000 inside the container, exposed to the external host environment at http://localhost:8081.

2.2 In-Depth Stage-by-Stage Breakdown

Stage 1: Checkout (SCM Retrieval)
The Checkout stage ensures that the Jenkins execution workspace contains the exact commit revision triggered for the build. When a pipeline job is configured as "Pipeline script from SCM", Jenkins automatically clones the configured Git repository and checks out the target branch (main). The explicit stage in the Jenkinsfile provides operational visibility in the Jenkins Stage View UI and logs confirmation of workspace availability.

Stage 2: Build (Docker Image Creation)
Command: `docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .`

This stage packages the application into an immutable container image.
- Dynamic tagging: the image is tagged with `${BUILD_NUMBER}` (for example `jenkins-flask-lab:1`). This guarantees traceability between Jenkins builds and container images.
- Layer caching: the Dockerfile copies `requirements.txt` and installs packages before copying `app.py`, so later builds reuse the dependency layer unless dependencies change.
- Lightweight base image: `python:3.12-slim` reduces image size and attack surface.

Stage 3: Test (Automated Quality Gate)
Command: `docker run --rm ${IMAGE_NAME}:${BUILD_NUMBER} pytest -v`

Tests run inside the same container image that will be deployed. The `--rm` flag removes the test container after pytest finishes. If any assertion fails, pytest exits with code 1 and the pipeline aborts before Deploy.

Stage 4: Deploy (Container Replacement)
Commands:
```
docker rm -f ${CONTAINER_NAME} || true
docker run -d --name ${CONTAINER_NAME} -p ${APP_PORT}:8000 -e APP_VERSION=${BUILD_NUMBER} ${IMAGE_NAME}:${BUILD_NUMBER}
```

- Idempotent teardown: `docker rm -f ... || true` removes a previous container without failing on the first run.
- Port binding: host `8081` maps to container `8000`.
- Version injection: `APP_VERSION=${BUILD_NUMBER}` is shown on the home page.

Stage 5: Verify (Smoke Test & Health Check)
Commands:
```
curl --fail --retry 10 --retry-delay 2 http://localhost:${APP_PORT}/health
curl --fail http://localhost:${APP_PORT}/
```

`--fail` treats HTTP 4xx/5xx as errors. Retries give Gunicorn time to bind the socket.

Post-build hooks:
- `success`: logs pipeline completion.
- `failure`: logs that a stage failed and that diagnostics should be checked.

3. Deliverable 3: Screenshot of Successful Jenkins Pipeline
Capture Jenkins Pipeline Stage View for a successful build showing Checkout, Build, Test, Deploy, and Verify all green. Attach as Figure 1.

4. Deliverable 4: Screenshot of Application Running in Browser
Open http://localhost:8081 and capture the page showing "Jenkins CI/CD Lab" and "Version: 1" (or the current build number). Confirm `/health` returns `{"status":"ok"}`. Attach as Figure 2.

5. Deliverable 5: Screenshot of Failed Test Stage & Technical Analysis
To validate the quality gate, change the health assertion in `tests/test_app.py` from `"ok"` to `"incorrect"`:

```
assert response.get_json() == {"status": "incorrect"}
```

Commit, push, and run the pipeline. The Test stage fails. Deploy and Verify are skipped.

Why deployment did not occur:
1. Pytest assertion failure returns exit code 1 from `docker run ... pytest`.
2. Jenkins `sh` treats a non-zero exit as a fatal error and abort the pipeline (fail-fast).
3. Deploy never runs, so the previous healthy container on port 8081 keeps serving traffic.

Restore the assertion to `"ok"` after capturing Figure 3.

6. Deliverable 6 (Part 1): Answers to Section 19 — Blue-Green Deployment

Question 1: What is the difference between deployment and release?
Deployment is a technical action: installing, starting, and configuring a new build in an environment (for example launching a container on an internal port). Release is a business/operational action: routing live user traffic to that deployed version (for example updating a reverse proxy). Separating the two is the basis of zero-downtime strategies.

Question 2: Why is the new version deployed to Green before receiving production traffic?
Green is an idle environment identical to production. You can smoke-test, warm caches, and catch startup failures there without affecting users. If Green crashes, Blue continues serving traffic.

Question 3: What component can route user requests to Blue or Green?
A reverse proxy, Layer 7 load balancer, or API gateway: NGINX, HAProxy, AWS ALB, Google Cloud HTTP(S) Load Balancer, Kubernetes Ingress, or a service mesh (Envoy/Istio).

Question 4: Why should Blue remain available after switching traffic to Green?
Blue is a warm rollback target. If Green misbehaves under real load, traffic can be switched back in milliseconds without rebuilding or cold-starting the old version. Teams can also compare metrics side by side.

Question 5: How can Jenkins automate the deployment, testing and approval steps?
Jenkins builds and starts Green, runs smoke tests against it, then uses an `input` step for human approval. After approval it reloads the proxy to point at Green. After a soak period it can decommission Blue if no alerts fire.

Question 6: What is the main benefit of rollback in a Blue-Green deployment?
Near-zero MTTR with no user downtime: rollback is a routing switch to an already-running known-good environment, not a rebuild and restart.

7. Deliverable 6 (Part 2): Answers to Section 20 — CI/CD, Docker & Jenkins

Question 1: What is Jenkins, and what problem does it solve?
Jenkins is an open-source automation server for CI/CD. It reduces integration hell, manual deploy mistakes, and slow feedback by building and testing every commit.

Question 2: What is the purpose of a Jenkinsfile?
It is Pipeline as Code: a Groovy pipeline definition stored in Git with the application. The pipeline is versioned, reviewable, branch-aware, and portable if the Jenkins server is rebuilt.

Question 3: What is the difference between Continuous Integration and Continuous Delivery?
CI automatically builds and tests every commit. CD extends CI by packaging verified artifacts and keeping them ready to deploy (often to staging), typically with a manual production approval. Fully automated production release is Continuous Deployment.

Question 4: What is the role of GitHub in this lab?
GitHub is the SCM and single source of truth for the app, tests, Dockerfile, and Jenkinsfile. Jenkins checks out the repo to run the pipeline.

Question 5: Why is Docker used in the pipeline?
Docker gives environment parity, isolated test runs, and a single deployable unit that starts, stops, and rolls back consistently.

Question 6: Why are automated tests executed before deployment?
They are a quality gate (shift-left). Failed tests abort the pipeline so broken images never replace a healthy production container.

Question 7: What happens when a pipeline stage fails?
The stage stops, later stages are skipped, the build is FAILURE, the `post { failure }` block runs, and production is left unchanged.

Question 8: What is the purpose of the Verify stage?
It is a post-deploy smoke test. A container can start and still fail to serve HTTP. curl against `/health` and `/` proves the live process answers 200.

Question 9: Why should credentials not be stored directly in a Jenkinsfile?
The Jenkinsfile lives in Git history. Secrets would leak to clones and forks. Store them in the Jenkins Credentials Store or a vault and inject them with `withCredentials()`.

8. Optional Extension & Production Engineering Enhancements
- GitHub webhooks instead of polling (`http://jenkins-host:8080/github-webhook/`)
- Image scanning (Trivy) after Build
- SAST / coverage gates (SonarQube, Bandit)
- Prometheus/Grafana after deploy

9. Conclusion
This lab implements a declarative Jenkins pipeline that builds a Docker image, tests it, deploys it to port 8081, and verifies HTTP health. Failed tests skip deploy and protect the running service. Blue-green concepts and CI/CD theory are documented above.

9.1 Lab Assessment Criteria Fulfillment

| Criterion | Weight | Status |
| --- | --- | --- |
| GitHub repository and structure | 10% | app.py, requirements.txt, Dockerfile, Jenkinsfile, tests |
| Jenkins Pipeline configuration | 25% | 5-stage declarative Jenkinsfile with env and post hooks |
| Docker image build | 15% | python:3.12-slim, `${BUILD_NUMBER}` tags |
| Automated test execution | 20% | pytest in an ephemeral container |
| Successful deployment & verification | 15% | 8081 mapping, `/` and `/health` |
| Failure experiment and explanation | 10% | Test fail-fast analysis in section 5 |
| Report and conceptual questions | 5% | Sections 6 and 7 |
