# REST API CI Pipeline Guide

This README explains the CI/CD requirements, best practices, and implementation for automating build, test, lint, and Docker image publishing for a Flask REST API using GitHub Actions and a self-hosted runner.

---

## 1. Overview of the Project Requirement for CI

- **Goal:**  
  Automate the process of building, testing, linting, and pushing Docker images of your REST API to a central registry (DockerHub or GitHub Container Registry).
- **Features:**  
  - CI pipeline with stages:
    - Build API with dependencies
    - Run automated tests
    - Perform code linting
    - Login to Docker registry
    - Build and push Docker image
  - Use `make` targets for build, test, and lint steps.
  - Pipeline runs on a self-hosted GitHub Actions runner.
  - Triggers only for changes in `app/` directory (not on docs/config/etc).
  - Manual trigger option (`workflow_dispatch`).
  - Only runs for relevant branches (`dev`, `main`).

---

## 2. CI Pipeline Best Practices in GitHub Actions

- **Consistency:**  
  Use the same pipeline steps for all environments (Dev, Staging, Prod) with only config changes.
- **Fail Fast:**  
  Stop pipeline immediately if lint or tests fail; don't build or deploy broken code.
- **Security:**  
  Store and use secrets using GitHub Secrets, never hardcode or echo them.
- **Speed:**  
  Optimize Docker builds, cache dependencies, and minimize pipeline execution time.
- **Idempotency:**  
  Pipelines should not cause duplicate deployments if re-run; builds must be reproducible.
- **Visibility:**  
  Keep logs clear, visible to the team, and publish test/lint results.
- **Manual & Automatic Triggers:**  
  Enable both event-based (push/PR) and manual triggers (`workflow_dispatch`) for flexibility.
- **Path Filtering:**  
  Use `paths` filter to run pipeline only for changes in relevant directories (e.g., `app/**`).
- **Short-lived Docker Images:**  
  Tag images using commit SHA (not `latest`), and clean up local images after push.
- **Self-hosted Runner Usage:**  
  Use your own runner for control over environment, caching, and resource allocation.

---

## 3. Explanation of the CI Pipeline Written

### **Trigger Configuration**
- Runs on push or pull request to `main` or `dev` branches, but only if files in `app/**` change.
- Can also be run manually.

### **Job Steps Breakdown**

| Step                       | Purpose                                                      |
|----------------------------|--------------------------------------------------------------|
| Checkout Code              | Pulls latest code from repo                                  |
| Setup Python               | Sets up Python runtime (v3.12)                               |
| Install Dependencies       | Installs all Python packages from `requirements.txt`         |
| Run Tests with SQLite      | Executes unit/integration tests (can be adapted for DB)      |
| Build Docker Image         | Builds image, tags with short commit SHA                     |
| Login to Docker Hub        | Authenticates to DockerHub using secrets                     |
| Push Docker Image          | Pushes image to registry, using tagged version               |
| Cleanup Local Images       | Removes built images to save disk space on runner            |

Example pipeline (`.github/workflows/rest-api-ci-pipeline.yml`):

```yaml
name: REST-API-CI-Pipeline

on:
  push:
    branches:
      - dev
      - main
    paths:
      - 'app/**'
  pull_request:
    branches:
      - dev
      - main
    paths:
      - 'app/**'
  workflow_dispatch:

jobs:
  build:
    runs-on: self-hosted
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.12'

      - name: Install Dependencies
        run: |
          cd app
          python3 -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run Tests with SQLite
        run: |
          pytest -v

      - name: Build Docker Image
        run: |
          cd app
          IMAGE_TAG=${GITHUB_SHA::7}  # short commit SHA
          echo "IMAGE_TAG=$IMAGE_TAG" >> $GITHUB_ENV
          docker build -t flask-app:$IMAGE_TAG .

      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_HUB_USERNAME }}
          password: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}

      - name: Push Docker Image to Docker Hub
        run: |
          docker tag flask-app:$IMAGE_TAG ${{ secrets.DOCKER_HUB_USERNAME }}/flask-app:$IMAGE_TAG
          docker push ${{ secrets.DOCKER_HUB_USERNAME }}/flask-app:$IMAGE_TAG

      - name: Cleanup Local Docker Images
        run: |
          docker rmi flask-app:$IMAGE_TAG || true
          docker rmi ${{ secrets.DOCKER_HUB_USERNAME }}/flask-app:$IMAGE_TAG || true
          docker system prune -f || true
```

**Key Points:**
- Only runs for changes in `app/**` (efficient, avoids unnecessary builds).
- Images are tagged with short commit SHA for traceability.
- Uses secrets for DockerHub login.
- Cleans up runner after push.

---

## 4. CI Pipeline Issues and Troubleshooting Guide

### 1. Python version mismatch in self-hosted runner

**Issue:**
  - Your workflow failed while installing click==8.2.1 because the runner’s default Python was 3.9:

**ERROR:**
  - No matching distribution found for click==8.2.1

**Cause:**
  - The package requires Python >=3.10, but the self-hosted runner had 3.9 installed.

**Resolution:**
  - Install the correct Python version on the runner (e.g., 3.12).
  - Use actions/setup-python in the workflow to explicitly set the version:
  
  ```yaml
  - name: Set up Python
    uses: actions/setup-python@v4
    with:
      python-version: '3.12'
  ```
---
### 2. Self-hosted runner not using the correct Python path

**Issue:**
  - Even after installing Python 3.12, python3 still pointed to 3.9.

**Cause:**
  - Self-hosted runners can have multiple Python versions installed; the PATH may not point to the newest.

**Resolution:**
  - Explicitly call the versioned binary if needed:
  ```sh
  python3.12 -m pip install --upgrade pip
  python3.12 -m pip install -r requirements.txt
  ```

  Or rely on actions/setup-python, which ensures the selected version is first in PATH.

---
### 3. Workflow YAML syntax errors

**Issue:**
  - Workflow failed with:
    - Invalid workflow file ... error in your yaml syntax on line 36

**Cause:**
  - Misaligned indentation (common in env or run blocks).
  - Example:

  ```yaml
  env:
    FLASK_ENV: testing
    DATABASE_URL: sqlite:///:memory
  run: pytest -v
  ```

**Resolution:**
  - Make sure run: is aligned with env:, not indented further:

  ```yaml
  - name: Run Tests with sqlite
  env:
    FLASK_ENV: testing
    DATABASE_URL: sqlite:///:memory
  run: pytest -v
  ```

---
### 4. Database connection errors in CI

**Issue:**
  - sqlalchemy.exc.OperationalError: connection to server at "localhost", port 5432 failed: Connection refused

**Cause:**
  - Your tests were trying to connect to PostgreSQL (psycopg2) instead of SQLite.
  - In local testing you used SQLite, but the app config in CI defaulted to PostgreSQL because create_app() was reading production config.

**Resolution:**
  - Make your create_app accept a config class:

  ```py
  def create_app(config_class=None):
    app = Flask(__name__)
    if config_class:
        app.config.from_object(config_class)
    else:
        app.config.from_object('app.config.Config')
  ```

  - Update conftest.py to pass the test config:
  ```py
  @pytest.fixture
    def client():
      app = create_app(config_class=TestConfig)
  ```

  - Use SQLite in-memory for tests:
  ```py
  SQLALCHEMY_DATABASE_URI = "sqlite:///:memory:"
  ```

---
### 5. GitHub Actions using wrong environment variables

**Issue:**
  - Even after adding .env-test, pipeline failed to use SQLite.

**Cause:**
  - Workflow was setting DATABASE_URL incorrectly.
  - Flask app wasn’t reading .env.test.

**Resolution:**
  - Ensure tests/config.py loads .env-test:
  ```py
  from dotenv import load_dotenv
  load_dotenv(dotenv_path=".env-test")
  ```
  - Set environment variables for pytest:
  ```yaml
  - name: Run Tests with sqlite
    env:
      FLASK_ENV: testing
      DATABASE_URL: sqlite:///:memory
    run: pytest -v
  ```

---
### 6. create_app() TypeError

**Issue:**
  - TypeError: create_app() got an unexpected keyword argument 'config_class'

**Cause:**

  - Your create_app() did not accept a parameter.
  - Tests were passing config_class.

**Resolution:**

  - Update create_app():

  ```py

  def create_app(config_class=None):
    app = Flask(__name__)
    if config_class:
        app.config.from_object(config_class)
    else:
        app.config.from_object('app.config.Config')
  ```

---
### 7. Docker Hub push failure

**Issue:**
  - unauthorized: access token has insufficient scopes

**Cause:**
  - GitHub Actions secret contained an old or insufficiently scoped token.

**Resolution:**
  - Go to Docker Hub → Account Settings → Security → New Access Token.
  - Assign write access for pushing images.
  - Update DOCKER_HUB_ACCESS_TOKEN in GitHub secrets.

  - Login step in workflow:
  ```yaml
  - name: Login to Docker Hub
    uses: docker/login-action@v2
    with:
      username: ${{ secrets.DOCKER_HUB_USERNAME }}
      password: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}
  ```

---
### 8. Using latest tag is not recommended

**Issue:**
  - Tagging images with latest can cause deployment of unintended versions.

**Resolution:**
  - Use immutable tags like short commit SHA:

  ```sh
  IMAGE_TAG=${GITHUB_SHA::7}
  docker tag flask-app:$IMAGE_TAG ${{ secrets.DOCKER_HUB_USERNAME }}/flask-app:$IMAGE_TAG
  docker push ${{ secrets.DOCKER_HUB_USERNAME }}/flask-app:$IMAGE_TAG
  ```

  - Optional: tag with branch name if needed, but avoid latest for production.

---
### 9. Cleanup of Docker images

**Issue:**
  - Runner storage gets full if images are not removed.

**Resolution:**
  - Clean up after push:
  ```yaml
  - name: Cleanup Local Docker Images
    run: |
      docker rmi flask-app:$IMAGE_TAG || true
      docker rmi ${{ secrets.DOCKER_HUB_USERNAME }}/flask-app:$IMAGE_TAG || true
      docker system prune -f || true
  ```

---
#### Final Recommendations

- Always specify Python version explicitly in workflows.
- For unit tests, always use SQLite in-memory or a separate test DB.
- Ensure create_app() supports test config injection.
- Keep Docker Hub tokens in GitHub secrets and refresh them after editing.
- Use commit-based Docker tags instead of latest.
- Clean up local images to save runner storage.
- Indentation in YAML is critical—always check env + run blocks.
- Always test your workflow locally with pytest before CI runs.

---

# CI/CD and GitHub Actions Interview Questions

This README explains the CI/CD most asked interview questions and answers in Github Actions.

---

#### **1: What is the difference between CI and CD?**
**A:** CI (Continuous Integration) automates building, testing, and validation of code on every commit. CD (Continuous Delivery/Deployment) automates deployment of validated code to staging or production environments.

---

#### **2: How do you trigger a GitHub Actions workflow only when specific files or directories change?**
**A:** Use the `paths` field under `on.push` and `on.pull_request` in the workflow YAML.

---

#### **3: How do you securely manage secrets in GitHub Actions?**
**A:** Store secrets in GitHub repository settings, and reference them using `${{ secrets.SECRET_NAME }}` in workflows.

---

#### **4: What is the purpose of the `workflow_dispatch` trigger?**
**A:** It allows manual triggering of workflows via the GitHub UI or API.

---

#### **5: How can you ensure that Docker images are traceable and not overwritten?**
**A:** Tag images with commit SHA or semantic versioning, and avoid using the `latest` tag for production images.

---

#### **6: How do you use a self-hosted runner in GitHub Actions?**
**A:** Set up a runner on your machine/server and specify `runs-on: self-hosted` in the workflow.

---

#### **7: Why is code linting important in CI pipelines?**
**A:** Linting catches style and simple code errors early, preventing issues before running tests or building images.

---

#### **8: How do you cache dependencies in GitHub Actions for faster builds?**
**A:** Use the `actions/cache` action to cache files or directories (e.g., pip cache, node_modules).

---

#### **9: How is idempotency achieved in CI/CD pipelines?**
**A:** By ensuring that running the same pipeline twice produces the same result and does not cause duplicate deployments or builds.

---

#### **10: What are common stages in a CI pipeline for a containerized app?**
**A:** Checkout, setup runtime, install dependencies, linting, testing, build Docker image, push to registry, (optional) deploy.

---

## References

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [DockerHub Login Action](https://github.com/docker/login-action)
- [CI/CD Best Practices](https://www.atlassian.com/continuous-delivery/ci-vs-cd)
