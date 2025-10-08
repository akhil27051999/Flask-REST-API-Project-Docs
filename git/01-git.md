# Git & Version Control: Best Practices and Dev-Prod Parity

This README covers the essential principles for maintaining environment consistency, config management, and version control workflow using Git. It also lists common interview questions related to Git, environment parity, and best practices in version-controlled software projects.

---

## 1. Principles of Dev-Prod Parity

### **1. Environment Consistency**
- Use containers (Docker) or Infrastructure as Code (IaC) tools (Vagrant, Terraform) to ensure identical OS, runtime, and dependency versions.
- Always pin versions in dependency files (`requirements.txt`, `package.json`, etc.).
- Use the same database engine and versions across all environments.

### **2. Config Management**
- Never hardcode credentials or secrets.
- Store configs in environment-specific `.env` files (`.env.dev`, `.env.staging`, `.env.prod`).
- Pass secrets via environment variables or secret managers.
- Always add `.env*` files to `.gitignore` to keep secrets out of your repo.

### **3. Infrastructure as Code (IaC)**
- Automate provisioning (Docker Compose, ECS/K8s manifests).
- Use the same scripts/configs for both dev and prod environments.

### **4. Data Parity**
- Use database migrations or seed snapshots for consistent schemas.
- Anonymize production data for use in staging/dev where realistic testing is needed.

### **5. Dependency Parity**
- Avoid installing extra packages only in dev.
- Lock dependency versions (`requirements.txt`, `poetry.lock`, `package-lock.json`).
- Build/run your app inside containers to guarantee parity.

### **6. Process Parity**
- Run your app identically in dev and prod (e.g., both via `docker-compose up`).
- Use the same process manager (e.g., Gunicorn, Nginx).

### **7. CI/CD Pipelines**
- Separate pipelines for dev, staging, prod.
- Use identical build steps; only differ in branch triggers and deployment targets.

### **8. Observability**
- Ensure logs and monitoring are present in all environments.
- Use the same stack for metrics (Prometheus, Grafana).

### **9. Feature Flags**
- Use feature flags to toggle features, instead of dev-only code paths.
- Prevent codebase divergence.

### **10. Release Flow**
- Merge code through PRs (no direct pushes to prod).
- Branching strategy:
  - `feature/*` → `dev`
  - `dev` → `staging`
  - `staging` → `main` (production)

### **Golden Rule**
> If something works in dev, it should work in prod without code changes.  
> Only difference should be config & scaling, not app logic.

---

## 2. Git Workflow: Full Cycle

### **Local → Feature → Dev**
```sh
git clone git@github.com:org/repo.git
cd repo
git checkout dev
git pull origin dev
git checkout -b feature/your-feature
# Make changes
git add .
git commit -m "Describe your change"
git push origin feature/your-feature
# Create a PR: feature/your-feature → dev
```

### **Dev → Staging**
```sh
git checkout dev
git pull origin dev
git checkout staging
git merge dev
git push origin staging
# Create a PR: dev → staging
```

### **Staging → Main**
```sh
git checkout staging
git pull origin staging
git checkout main
git merge staging
git push origin main
# Create a PR: staging → main
```
---
## 3. Creating a Pull Request (PR) on GitHub

1. **Push Your Feature Branch**
   ```sh
   git push origin feature/my-new-feature
   ```
2. **Open GitHub Repository in Browser**
   - Go to your repository’s page on GitHub.
   - You’ll often see a "Compare & pull request" button for your branch.
   - Alternatively, click the "Pull requests" tab and "New pull request".

3. **Select Branches**
   - Set the base branch (e.g., `dev` or `main`).
   - Set the compare branch (your feature branch).

4. **Fill PR Details**
   - Enter a title and description explaining your changes.

5. **Create the Pull Request**
   - Click "Create pull request".

6. **Review & Merge**
   - Team reviews your PR, you can address comments.
   - Once approved, click "Merge pull request".

> **Tip:** Always use PRs for code review, CI testing, and safe merges. Never push directly to production branches.

---

### **How CI/CD Fits**
- **Dev branch** → triggers Dev pipeline → deploys to Dev server
- **Staging branch** → triggers Staging pipeline → deploys to Staging server
- **Main branch** → triggers Prod pipeline → deploys to Prod server

---

## 4. Git Troubleshooting Guide

### 1. Wrong branch checkout / committed to wrong branch

`Problem`: You make changes in main instead of feature/....

`Fix:`

```sh
# create a new branch from current state
git checkout -b feature/fix-branch

# reset main back to remote state
git checkout main
git reset --hard origin/main
```
---

### 2. Forgot to pull latest changes before pushing (out of date branch)

`Problem`: 
```sh
git push origin dev
# error: failed to push some refs
```

`Fix`:
```sh
git pull --rebase origin dev   # pulls latest & reapplies your commits
git push origin dev
```
---

### 3. Merge conflicts

`Problem`: When merging PRs or running git pull, Git can’t auto-merge.

`Fix`:

```sh
# shows conflicting files
git status

# edit conflicting files manually (look for <<<<<<<, =======, >>>>>>>)
git add <resolved_file>
git commit
```
---

### 4. Accidentally committed secrets / large files

`Problem`: Committed .env file or large binaries.

`Fix`:
```sh
git rm --cached .env   # remove from Git index
echo ".env" >> .gitignore
git commit -m "remove secrets from repo"
git push
```

- For already pushed secrets → rotate keys immediately and use git filter-repo or BFG to remove from history.
---

### 5. Detached HEAD state

`Problem`: You’re on a commit, not a branch.

```sh
git checkout <commit_hash>
# now in "detached HEAD"
```

`Fix`:

```sh
git checkout -b feature/fix-detached
```
---

### 6. Need to undo last commit

`Case A`: Undo but keep changes staged

```sh
git reset --soft HEAD~1
```

`Case B`: Undo and discard changes
```sh
git reset --hard HEAD~1
```

`Case C`: Undo after push
```sh
git revert <commit_hash>   # safe for shared branches
```
---

### 7. Pushed to the wrong branch (e.g., pushed to main instead of dev)

`Fix`:

```sh
# create a new branch from that commit
git checkout -b feature/wrong-push

# reset main to remote state
git checkout main
git reset --hard origin/main
git push --force origin main
```
- Then open PR from feature/wrong-push → dev.
---

### 8. Branch name mistake / need to rename

```sh
git branch -m old-branch-name new-branch-name
git push origin :old-branch-name
git push origin new-branch-name
```
---

### 9. Delete local & remote branch

```sh
git branch -d feature/old-branch   # local
git push origin --delete feature/old-branch   # remote
```
---

### 10. CI/CD pipeline not triggering

`Causes`:

- Workflow file not in .github/workflows/.
- Wrong branch filter (on: push: branches:).
- YAML syntax errors.

`Fix`:

- Run yamllint <file> locally.
- Check GitHub Actions logs under Actions tab.

---

### 11. SSH vs HTTPS auth issues

`Problem`: Can’t push because wrong credentials.

`Fix`:

- switch to SSH

```sh
git remote set-url origin git@github.com:org/repo.git
```

- Check SSH connection:
```sh
ssh -T git@github.com
```
---

### 12. Rebasing gone wrong

`Problem`: Messed up history after git rebase.

`Fix`:
```sh
git rebase --abort    # cancel rebase
```

- Or reset back:
```sh
git reset --hard ORIG_HEAD
```

---

### 13. Someone force-pushed and broke history

`Problem`: Branch has diverged.

`Fix`:
```sh
git fetch origin
git checkout dev
git reset --hard origin/dev
```

---

### 14. Accidentally deleted branch

`Fix` (if still on remote):
```sh
git checkout -b dev origin/dev
```

- If deleted locally and remotely → use git reflog to recover.

---

### 15. Submodules not updating

`Fix`:

```sh
git submodule update --init --recursive
```

## 5. References

- [Git Documentation](https://git-scm.com/doc)
- [GitHub Flow](https://docs.github.com/en/get-started/quickstart/github-flow)
- [Twelve-Factor App](https://12factor.net/)
- [Version Control Best Practices](https://www.atlassian.com/git/tutorials/comparing-workflows)

---

# Interview Questions (Git, Parity, Version Control)

## General Git & Version Control

#### **1: What is the difference between Git and GitHub?**
**A:** Git is a distributed version control system for tracking changes in source code. GitHub is a web-based platform for hosting Git repositories and collaborating on code.

---

#### **2: Why are branches important and how do you use them in a collaborative project?** 
**A:** Branches allow multiple developers to work in parallel, isolate features, and manage code integration. You typically work in a feature branch, then merge into dev, staging, and main via pull requests.

---

#### **3: What is a merge conflict? How do you resolve it?**
**A:** A merge conflict occurs when two branches have competing changes to the same line of code. You resolve it by manually editing the file, staging the changes, and committing the resolved file.

---

#### **4: Why should you use `.gitignore`?**
**A:** To prevent sensitive or irrelevant files (like `.env`, `__pycache__`, or build artifacts) from being committed to the repository.

---

#### **5: What is the advantage of using Pull Requests (PRs) over direct pushes to main?** 
**A:** PRs enable code review, automated testing, and safe integration, reducing the risk of bugs and maintaining code quality.

---

## Dev-Prod Parity & Environment Management

#### **6: What is dev-prod parity and why is it important?**
**A:** Dev-prod parity means development and production environments are as similar as possible, which helps prevent unexpected bugs and makes deployments predictable.

---

#### **7: What strategies can you use to manage configuration and secrets securely across environments?**
**A:** Use environment variables, `.env` files (ignored from Git), and secret managers like AWS Secrets Manager or Vault.

---

#### **8: How do containers (Docker) and IaC (Vagrant, Terraform) help with environment consistency?**
**A:** They allow you to define and replicate environments easily, ensuring all dependencies, OS, and configs are identical across machines and deployments.

---

#### **9: What is Infrastructure as Code (IaC) and how is it used in this project?**
**A:** IaC means defining your infrastructure (servers, networks, etc.) in code (e.g., Vagrantfile, Docker Compose). In this project, Vagrant provisions the VM and Docker Compose sets up the containers.

---

## CI/CD & Project Milestone-Specific Questions

#### **10: How does your CI pipeline ensure code quality and reproducibility?**
**A:** It runs linting, tests, builds a Docker image, and pushes to a registry—all automatically triggered on relevant branch pushes or PRs, using pinned dependency versions and environment isolation.

---

#### **11: How did you configure your CI pipeline to only run for code changes in a specific directory?**
**A:** By using the `paths` filter in the GitHub Actions workflow YAML, restricting runs to changes under `app/**`.

---

#### **12: How do you use Docker Compose for local development and production deployment?** 
**A:** Docker Compose defines all services (API, DB, Nginx) and their dependencies, enabling one-command multi-container orchestration for both dev and prod environments.

---

#### **13: Why did you use multiple API containers and Nginx in your deployment architecture?** 
**A:** Multiple API containers (replicas) increase availability and scalability; Nginx acts as a load balancer to distribute requests evenly.

---

#### **14: How do you handle database migrations in the pipeline and deployment?**
**A:** By running migration commands as part of the Makefile and/or CI pipeline before deploying or starting the API containers.

---

#### **15: What troubleshooting steps would you take if your Flask app cannot connect to PostgreSQL in Docker Compose?** 
**A:**  
- Check the health/status of both containers (`docker-compose ps`, `docker-compose logs postgres`).
- Verify environment variables for DB connection.
- Ensure healthchecks are passing.
- Use `docker-compose exec` to test DB connectivity manually.

---

#### **16: How did you ensure that your Docker images are small and efficient?** 
**A:** By using multi-stage builds, lightweight base images (Alpine), and cleaning up unused files and build cache in the Dockerfile.

---

#### **17: Describe your branching strategy and how it supports CI/CD.**
**A:** Feature branches merge into dev for integration and testing; dev merges into staging for QA; staging merges into main for production. Each branch triggers its environment-specific pipeline.

---

#### **18: How do you use Makefile targets to automate common tasks in your project?**
**A:** Makefile targets wrap Docker Compose commands for building images, running containers, applying migrations, checking logs, and cleaning up, enabling quick and reliable environment setup.

---

#### **19: What process do you follow for reviewing and merging code?** 
**A:**  
- Push changes to a feature branch.
- Open a pull request to the target branch (dev, staging, main).
- Undergo code review and CI checks.
- Merge after approval and successful tests.
