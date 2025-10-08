# Dockerfile Best Practices for Python REST API

This document covers best practices for writing Dockerfiles for Python REST APIs (using Flask in this case), Docker commands used in this milestone, and troubleshooting steps.

---

## 1. Dockerfile Best Practices

#### A. Use Multi-stage Builds
- Split your Dockerfile into build and runtime stages.
- Build dependencies and Python packages in a dedicated stage, then copy only the necessary artifacts to the final image.
- This reduces image size and avoids leaking build tools into runtime.

#### B. Choose Minimal Base Images
- Use lightweight images such as `python:3.10-alpine` to minimize the footprint.
- Alpine Linux-based images are small and secure, but make sure all dependencies (e.g., C libraries for psycopg2) are compatible.

#### C. Avoid Latest Tag
- Always tag your images using semantic versioning (e.g., `v1.0.0`), not `latest`.
- This improves reproducibility and tracking.

#### D. Remove Build Artifacts
- Delete cache, temp files, and `.pyc`/`__pycache__` to keep the image small.

#### E. Install Only What You Need
- Use `--no-cache` for package managers (`apk add --no-cache ...`).
- Avoid unnecessary tools/binaries.

#### F. Inject Environment Variables at Runtime
- Do **not** hardcode sensitive configs (like DB URLs) in the Dockerfile.
- Pass environment variables with `-e` or `--env-file` when running containers.

#### G. Use Non-root User (Optional for advanced security)
- For production, consider creating a non-root user to run your app.

#### H. Expose Only Required Ports
- Expose the port your app listens on, nothing else.

#### I. Use a Robust Entrypoint
- Prefer production WSGI servers (e.g., Gunicorn) to run Flask apps.

---

## 2. Docker Commands

#### Build the Docker Image
```bash
docker build -t student-api:v1.0.0 .
```
- `-t student-api:v1.0.0`: Tags the image (`v1.0.0` is semantic version, avoid `latest`).

#### Run the Container with Environment Variables
```bash
docker run -d \
  -p 5000:5000 \
  --name student-api \
  -e DATABASE_URL=postgresql://user:password@host:5432/dbname \
  -e FLASK_ENV=production \
  student-api:v1.0.0
```
- `-e`: Injects environment variables.
- `-p 5000:5000`: Maps host port to container port.

#### Run with an Env File
```bash
docker run -d \
  --env-file .env \
  -p 5000:5000 \
  student-api:v1.0.0
```
- `.env` file should contain key-value pairs for environment variables.

#### List Docker Images
```bash
docker images
```

#### Stop and Remove Containers
```bash
docker stop student-api
docker rm student-api
```

#### Remove Docker Images
```bash
docker rmi student-api:v1.0.0
```

---

## 3. Makefile Targets

```makefile
docker-build:
	docker build -t student-api:v1.0.0 .

docker-run:
	docker run -d -p 5000:5000 --name student-api --env-file .env student-api:v1.0.0

docker-stop:
	docker stop student-api && docker rm student-api

docker-clean:
	docker rmi student-api:v1.0.0
```

---

## 4. Troubleshooting Container Startup & Dependency Issues

### Common Issues
- **Missing Dependencies:** Alpine requires musl-based build tools. Install with `apk`.
- **App fails to start:** Check logs with `docker logs student-api`.
- **DB Connection Error:** Ensure `DATABASE_URL` is set and the DB is reachable from the container.
- **Port Binding Issues:** Make sure the host port is not in use.
- **Environment Variable Not Loaded:** Use `-e` or `--env-file` correctly.
---

### **1. Missing Dependencies (Alpine: musl-based build tools)**

**Explanation:**  
Alpine Linux uses musl-libc instead of glibc and has its own package manager (`apk`). Some Python packages (like `psycopg2` for PostgreSQL) require C build tools and specific libraries to compile.

**How to Fix:**  
Install the required build dependencies in your Dockerfile or interactively in the container.

**Dockerfile Example:**
```dockerfile
RUN apk add --no-cache gcc musl-dev postgresql-dev
```

**Interactive Command:**
```sh
docker-compose exec flask-app sh  # or docker exec -it <container> sh
apk add --no-cache gcc musl-dev postgresql-dev
```

---

### **2. App Fails to Start**

**Explanation:**  
If the container starts but the application inside crashes or never launches, you need to check container logs for error messages.

**How to Fix:**  
View the logs to diagnose the issue.

**Command:**
```sh
docker logs student-api
# or for docker-compose
docker-compose logs flask-app
```

**Typical Issues to Look For:**  
- Missing dependencies
- Syntax errors
- Unhandled exceptions
- Failed DB connections

---

### **3. DB Connection Error**

**Explanation:**  
The application cannot connect to the database, possibly due to a misconfigured connection string, DB not running, or network issues.

**How to Fix:**  
- Ensure the `DATABASE_URL` is set and correct.
- Confirm the DB container is healthy and accessible.

**Commands:**
```sh
# Check environment variable inside the app container
docker-compose exec flask-app env | grep DATABASE_URL

# Test DB connectivity from the app container
docker-compose exec flask-app sh
# Inside the shell (for Postgres):
pg_isready -h postgres -U <user> -d <db>
```

**Check DB Container Health:**
```sh
docker-compose ps
docker-compose logs postgres
```

---

### **4. Port Binding Issues**

**Explanation:**  
If the port you want to expose (e.g., 5000 for Flask, 5432 for Postgres) is already in use on your host, the container will fail to start.

**How to Fix:**  
Find and stop the process using the port, or use a different port mapping.

**Commands:**
```sh
# Find processes using the port
sudo lsof -i :5000
sudo lsof -i :5432

# Stop conflicting containers
docker ps
docker stop <conflicting_container>

# Restart your compose stack
docker-compose down
docker-compose up -d
```

---

### **5. Environment Variable Not Loaded**

**Explanation:**  
If your app depends on environment variables and they're not set, it may fail to start or behave incorrectly.

**How to Fix:**  
Pass the environment variables using `-e` flags or an `.env` file.

**Commands:**
```sh
# Using -e flag
docker run -e DATABASE_URL=postgresql://user:password@host:5432/dbname ...

# Using --env-file
docker run --env-file .env ...

# For docker-compose, ensure env_file is specified:
services:
  flask-app:
    env_file:
      - .env
```

**Check Loaded Variables:**
```sh
docker-compose exec flask-app env
```

---

## 5. Key Points to Remember

- Use multi-stage builds to minimize image size.
- Inject environment variables at runtime; never hardcode secrets.
- Tag images with semantic versioning.
- Use Makefile targets for reproducibility.
- Document all steps in README for clarity.
- Troubleshoot using logs, interactive shells, and Docker commands.
- Always keep security and minimalism in mind.

---

## References

- [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Python on Docker](https://pythonspeed.com/docker/)
- [Multi-stage Builds](https://docs.docker.com/develop/develop-images/multistage-build/)
- [Docker Compose Environment Variables](https://docs.docker.com/compose/environment-variables/)

---

# Docker Image Optimization: Best Practices & Techniques

This explains how to drastically reduce build time and image size for Dockerized Python applications (e.g., Flask REST APIs). It covers practical strategies, example Dockerfiles, and tips for efficient Docker builds.

---

## Why Optimize Docker Images?

- **Smaller images** = faster uploads/downloads, quicker CI/CD, and lower resource usage.
- **Faster builds** = efficient developer workflows and speedy deployments.

---

## 1. Improve Build Time with Layer Caching

### **How Docker Builds Work**
- Docker builds images in layers: each instruction in the Dockerfile creates a new layer.
- Layers are cached if their contents (hashes) do not change.
- Changing a layer (e.g., modifying code) invalidates all layers after it, forcing Docker to rebuild them.

### **Optimization Strategy**
- Place frequently-changing operations **lower** in the Dockerfile.
- Place rarely-changing operations (like installing system or Python dependencies) **higher**.

### **Example Dockerfile (Build Time Optimized)**
```dockerfile
FROM python:3.12

# Install OS dependencies first
RUN apt-get update && apt-get install -y <build_dependencies>

# Copy only requirements.txt and install Python packages early
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Set working directory and copy project files last
WORKDIR /app
COPY . .

EXPOSE 5000
CMD flask run --host=0.0.0.0 --port=5000
```
**Result:**  
If code changes but `requirements.txt` does not, Docker reuses the dependency installation layer, saving build time.

---

## 2. Reduce Docker Image Size

### **Strategies**
- Use lightweight base images (e.g., `python:3.10-alpine` vs. default Debian images).
- Remove unnecessary files (bytecode, cache, logs).
- Install only what you need.
- Use **multi-stage builds** to separate build tools from runtime.

### **Multi-Stage Builds Example**
```dockerfile
# Stage 1: Build dependencies
FROM python:3.10-alpine AS build
WORKDIR /api/app
RUN apk add --no-cache gcc musl-dev postgresql-dev
COPY requirements.txt ./
RUN pip install --user --no-cache-dir -r requirements.txt \
    && find /root/.local -name '*.pyc' -delete \
    && find /root/.local -name '__pycache__' -delete

# Stage 2: Runtime image
FROM python:3.10-alpine AS main
WORKDIR /api
RUN apk add --no-cache postgresql-client
COPY --from=build /root/.local /root/.local
COPY . ./app
ENV PATH=/root/.local/bin:$PATH
ENV FLASK_APP=app/wsgi.py
EXPOSE 5000
CMD ["gunicorn", "app.wsgi:app"]
```

### Implementation

```sh
ubuntu@ip-10-0-6-246:~/Flask-REST-API/app$ docker build -t flask-app:7.0.0 .
[+] Building 0.4s (15/15) FINISHED                                                                                                                                      docker:default
 => [internal] load build definition from Dockerfile                                                                                                                              0.0s
 => => transferring dockerfile: 927B                                                                                                                                              0.0s
 => [internal] load metadata for docker.io/library/python:3.10-alpine                                                                                                             0.1s
 => [auth] library/python:pull token for registry-1.docker.io                                                                                                                     0.0s
 => [internal] load .dockerignore                                                                                                                                                 0.0s
 => => transferring context: 2B                                                                                                                                                   0.0s
 => [internal] load build context                                                                                                                                                 0.0s
 => => transferring context: 2.55kB                                                                                                                                               0.0s
 => [build 1/5] FROM docker.io/library/python:3.10-alpine@sha256:24cab748bf7bd8e3d2f9bb4e5771f17b628417527a4e1f2c59c370c2a8a27f1c                                                 0.0s
 => CACHED [main 2/5] WORKDIR /api                                                                                                                                                0.0s
 => CACHED [main 3/5] RUN apk add --no-cache postgresql-client                                                                                                                    0.0s
 => CACHED [build 2/5] WORKDIR /api/app                                                                                                                                           0.0s
 => CACHED [build 3/5] RUN apk add --no-cache gcc musl-dev postgresql-dev                                                                                                         0.0s
 => CACHED [build 4/5] COPY requirements.txt ./                                                                                                                                   0.0s
 => CACHED [build 5/5] RUN pip install --user --no-cache-dir -r requirements.txt     && find /root/.local -name '*.pyc' -delete     && find /root/.local -name '__pycache__' -de  0.0s
 => CACHED [main 4/5] COPY --from=build /root/.local /root/.local                                                                                                                 0.0s
 => [main 5/5] COPY . ./app                                                                                                                                                       0.1s
 => exporting to image                                                                                                                                                            0.1s
 => => exporting layers                                                                                                                                                           0.0s
 => => writing image sha256:6c928ff24bb26018336ee58bd826f3abaf64ac10f28caa1c11a437464f65c520                                                                                      0.0s
 => => naming to docker.io/library/flask-app:7.0.0
                                                                                                                        0.0s
ubuntu@ip-10-0-6-246:~/Flask-REST-API/app$ docker images
REPOSITORY                    TAG       IMAGE ID       CREATED         SIZE
flask-app                     7.0.0     6c928ff24bb2   8 seconds ago   90.2MB
flask-app                     1.0.0     433b8919ecd2   6 days ago      90.2MB
akhilthyadi/flask-app         7.0.0     fb8b7b968025   2 weeks ago     90.2MB
gcr.io/k8s-minikube/kicbase   v0.0.48   c6b5532e987b   3 weeks ago     1.31GB
postgres                      15        b3d48187cc8a   3 weeks ago     445MB
nginx                         alpine    4a86014ec699   7 weeks ago     52.5MB

ubuntu@ip-10-0-6-246:~/Flask-REST-API/app$ docker run -p 5000:5000 flask-app:7.0.0   gunicorn --bind 0.0.0.0:5000 --workers 2 app.wsgi:app
[2025-10-03 13:07:24 +0000] [1] [INFO] Starting gunicorn 23.0.0
[2025-10-03 13:07:24 +0000] [1] [INFO] Listening at: http://0.0.0.0:5000 (1)
[2025-10-03 13:07:24 +0000] [1] [INFO] Using worker: sync
[2025-10-03 13:07:24 +0000] [6] [INFO] Booting worker with pid: 6
[2025-10-03 13:07:24 +0000] [7] [INFO] Booting worker with pid: 7

ubuntu@ip-10-0-6-246:~/Flask-REST-API$ docker ps -a
CONTAINER ID   IMAGE                                 COMMAND                  CREATED          STATUS                      PORTS                                         NAMES
91ee92303b6a   flask-app:7.0.0                       "gunicorn --bind 0.0…"   49 seconds ago   Up 48 seconds               0.0.0.0:5000->5000/tcp, [::]:5000->5000/tcp   wizardly_rhodes
8c11379be88c   gcr.io/k8s-minikube/kicbase:v0.0.48   "/usr/local/bin/entr…"   9 days ago       Exited (130) 18 hours ago                                                 minikube-m03
6df8ab455d80   gcr.io/k8s-minikube/kicbase:v0.0.48   "/usr/local/bin/entr…"   9 days ago       Exited (130) 18 hours ago                                                 minikube-m02
d0a9042a8160   gcr.io/k8s-minikube/kicbase:v0.0.48   "/usr/local/bin/entr…"   9 days ago       Exited (130) 18 hours ago                                                 minikube
```

**Result:**  
- Only essential runtime files are included; build tools and caches are discarded.
- Image size drops from over 1GB to less than 110MB (over 90% reduction).

---

## 3. Use `.dockerignore` to Exclude Unwanted Files

Create a `.dockerignore` file to prevent sensitive and unnecessary files from being copied into your build.
Example:
```
# Ignore virtual environments
/venv/*
# Ignore env files
.env
.env.test
# Ignore Python bytecode
app/__pycache__/*
__pycache__/
*.pyc
# Ignore log files
*.log
```
**Benefits:**  
- Prevents accidental exposure of secrets.
- Keeps image size minimal.

---

## 4. Real Results

- **Original Image:** 1.26GB (≈1290MB)
- **Optimized Image:** 110MB
- **Reduction:** ~91%

**Build Time:**  
- Layer caching and staged builds result in much faster rebuilds (20s+ saved per build).

---

## 5. Summary of Key Recommendations

- **Copy `requirements.txt` and install dependencies before copying source code.**
- **Use Alpine images for minimal footprint.**
- **Adopt multi-stage builds to separate builder and runtime environments.**
- **Clean up caches, bytecode, and logs after installation.**
- **Use `.dockerignore` to exclude unneeded files.**
- **Tag images with semantic versioning, not `latest`.**
- **Expose only necessary ports and inject environment variables at runtime.**

---

## References

- [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Python on Docker](https://pythonspeed.com/docker/)
- [Multi-stage Builds](https://docs.docker.com/develop/develop-images/multistage-build/)
- [.dockerignore Reference](https://docs.docker.com/engine/reference/builder/#dockerignore-file)

---
# Dockerfile Interview Questions & Answers

This document provides common interview questions related to Dockerfiles, Docker image building, and best practices, along with sample answers.

---

#### **1. What is a Dockerfile?**
**Answer:**  
A Dockerfile is a text file that contains instructions for building a Docker image. Each instruction in the Dockerfile creates a layer in the image, specifying how the environment should be set up, which files to include, and how to run the application.

---

#### **2. What are multi-stage builds in Docker, and why are they important?**
**Answer:**  
Multi-stage builds allow you to use multiple `FROM` statements in your Dockerfile, letting you separate your build environment from your runtime environment. This helps create smaller, more secure images by copying only the necessary artifacts from the build stage to the final image and leaving behind unnecessary build tools and dependencies.

---

#### **3. Why should you avoid using the `latest` tag for Docker images in production?**
**Answer:**  
The `latest` tag is ambiguous and can lead to unpredictable deployments since it may point to different versions over time. Using explicit semantic version tags (e.g., `v1.0.0`) ensures reproducibility and traceability.

---

#### **4. How do you inject environment variables into a Docker container?**
**Answer:**  
You can inject environment variables at runtime using the `-e` flag:
```bash
docker run -e VAR_NAME=value image:tag
```
Or by using an environment file:
```bash
docker run --env-file .env image:tag
```

---

#### **5. What are some best practices for writing secure and efficient Dockerfiles?**
**Answer:**  
- Use minimal base images (e.g., Alpine).
- Implement multi-stage builds to reduce image size.
- Avoid hardcoding secrets/configs; use environment variables.
- Remove build artifacts and cache files.
- Run applications as non-root users when possible.
- Expose only necessary ports.
- Use official images and pin versions.

---

#### **6. How can you reduce the size of your Docker image?**
**Answer:**  
- Use lightweight base images like `python:3.10-alpine`.
- Remove unnecessary files and caches after installation.
- Only install required dependencies.
- Use multi-stage builds to separate build and runtime.
- Clean up temporary files and `.pyc` caches.

---

#### **7. What is the difference between `CMD` and `ENTRYPOINT` in a Dockerfile?**
**Answer:**  
- `CMD` sets default commands or parameters that can be overridden at runtime.
- `ENTRYPOINT` specifies a command that will always run, making the container behave like the specified executable.
- They can be used together, where `ENTRYPOINT` sets the main command and `CMD` provides default arguments.

---

#### **8. How do you troubleshoot a Docker container that won’t start?**
**Answer:**  
- Check container logs using `docker logs <container_name>`.
- Run the container interactively (`docker run -it ...`) to inspect the environment.
- Verify environment variables and exposed ports.
- Ensure dependencies and services (like databases) are accessible.

---

#### **9. Why is it important to explicitly declare dependencies in your project and Dockerfile?**
**Answer:**  
Explicitly declaring dependencies (e.g., in `requirements.txt`) ensures the environment is reproducible. It avoids "it works on my machine" problems and makes it easier to manage updates and security patches.

---

#### **10. What command lists all Docker images on your host?**
**Answer:**  
```bash
docker images
```

---

#### **11. Explain the difference between `COPY` and `ADD` instructions in Dockerfile.**
**Answer:**  
- `COPY` simply copies files/directories from source to destination.
- `ADD` does the same but can also extract local tar archives and supports remote URLs (discouraged for security).

---

#### **12. How do you clean up unused containers and images to free disk space?**
**Answer:**  
```bash
docker system prune -a
```
This removes all unused containers, networks, images, and cache.

---

#### **13. What is the purpose of exposing a port in a Dockerfile?**
**Answer:**  
`EXPOSE` informs Docker that the container listens on the specified network port at runtime. It's a documentation feature and does not actually publish the port.

---

#### 14. **How do you run your Flask app in production using Docker?**
**Answer:**  
Use a robust WSGI server like Gunicorn in the Dockerfile:
```dockerfile
CMD ["gunicorn", "app.wsgi:app"]
```
Expose the app port and pass necessary environment variables.

---

#### **15. What is the difference between a Docker image and a Docker container?**
**Answer:**  
- A Docker **image** is a snapshot/template with everything needed to run an application.
- A Docker **container** is a running instance of an image.

---

## References

- [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Docker Documentation](https://docs.docker.com/)

---
