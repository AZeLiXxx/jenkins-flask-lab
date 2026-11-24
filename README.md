# Jenkins CI/CD Lab

Automated CI/CD pipeline for a Flask microservice using Jenkins, Docker, and GitHub.

## Application

- `/` — HTML page with the lab title and `APP_VERSION`
- `/health` — JSON health check `{ "status": "ok" }`
- Internal port: `8000` (Gunicorn)
- Host mapping: `http://localhost:8081`

## Local tests

```bash
pip install -r requirements.txt
pytest -v
```

## Docker

```bash
docker build -t jenkins-flask-lab:1 .
docker run --rm jenkins-flask-lab:1 pytest -v
docker rm -f jenkins-flask-app || true
docker run -d --name jenkins-flask-app -p 8081:8000 -e APP_VERSION=1 jenkins-flask-lab:1
curl --fail http://localhost:8081/health
```

## Jenkins

Create a Pipeline job using **Pipeline script from SCM**:

- SCM: Git
- Branch: `main`
- Script path: `Jenkinsfile`

Jenkins must be able to run Docker (`docker` on the agent PATH, typically with the Docker socket available).
