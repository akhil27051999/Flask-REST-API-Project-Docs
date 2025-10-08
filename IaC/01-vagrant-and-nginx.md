# Production Deployment with Vagrant, Docker Compose, and Nginx Load Balancer

This README guides you through setting up a production-like environment for your REST API using Vagrant, Docker Compose, and Nginx for load balancing between multiple API containers.

---

## 1. Overview

- **Goal:**  
  Deploy a scalable REST API stack with:
  - **2 API containers** (for load balancing)
  - **1 PostgreSQL DB container**
  - **1 Nginx container** (as a load balancer for the APIs)
- **Environment:**  
  - Vagrant box (AWS or local VM)
  - Docker Compose for container orchestration
  - Bash bootstrap script for dependency installation
  - Nginx configured for round-robin load balancing

---

## 2. Vagrant Installation & Setup

### **Step 1: Install Vagrant**

On Ubuntu/Debian:
```sh
sudo apt update
sudo apt install vagrant -y
```

Check version:
```sh
vagrant --version
```

### **Step 2: Install AWS CLI**
```sh
sudo apt install awscli -y
```

Configure your AWS credentials:

```sh
aws configure
```
- AWS Access Key ID → <your access key>
- AWS Secret Access Key → <your secret key>
- Default region name → us-east-1 (or your region)
- Default output format → json

### **Step 3: Install the Vagrant AWS Plugin**
```sh
vagrant plugin install vagrant-aws
```

### **Step 4: Add a Dummy Box**
```sh
vagrant box add dummy https://github.com/mitchellh/vagrant-aws/raw/master/dummy.box --provider=aws
```

### **Step 5: Create a Vagrant Project**
```sh
mkdir ~/Flask-REST-API/vagrant
cd ~/Flask-REST-API/vagrant
```

### **Step 6: Create a .env file for credentials**
```sh
cat > .env <<EOL
AWS_ACCESS_KEY_ID="<AWS_ACCESS_ID>"
AWS_SECRET_ACCESS_KEY="<PASSWORD>
EOL
```

Add the .pem file to the project directory and set correct permissions.
```sh
cp ~/api-server.pem ~/vagrant-setup/
chmod 400 api-server.pem
```

### **Step 7: Write the Vagrantfile**
Create the Vagrantfile with the configuration for the EC2 instance.  
Find and use a valid Ubuntu 22.04 LTS AMI ID for your region!

```sh
vi Vagrantfile
```

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'vagrant-aws'

Vagrant.configure("2") do |config|
  config.vm.box = "dummy"

  config.vm.provider "aws" do |aws, override|
    # AWS credentials
    aws.access_key_id = ENV['AWS_ACCESS_KEY_ID']
    aws.secret_access_key = ENV['AWS_SECRET_ACCESS_KEY']

    # AWS region and AMI
    aws.region = "us-east-1"
    aws.ami = "ami-054d6a336762e438e"

    # Instance type
    aws.instance_type = "t3.small"

    # Key pair for SSH
    aws.keypair_name = "api-server"
    override.ssh.username = "ubuntu"
    override.ssh.private_key_path = File.expand_path("api-server.pem", __dir__)

    # Network settings
    aws.subnet_id = "subnet-0a03e0dbddc93b6aa"
    aws.associate_public_ip = true
    aws.security_groups = ["sg-09d4c5f5679711dd2"]

    # Do not terminate instance on shutdown
    aws.terminate_on_shutdown = false
  end
end
```

### **Step 8: Launch the EC2 Instance**

- Load the environment variables:
```sh
source .env
```

- Launch the instance:
```sh
AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY vagrant up --provider=aws
```

### **Step 9: SSH into the Instance**

```sh
vagrant ssh
# Or directly:
ssh -i api-server.pem ubuntu@<PUBLIC_IP>
```

### **Common Vagrant Commands**
- `vagrant ssh`: Connect to the running EC2 instance via SSH.
- `vagrant halt`: Gracefully shuts down the EC2 instance.
- `vagrant destroy`: Permanently terminates the EC2 instance and all associated resources, removing the infrastructure to prevent further charges.

---

## 3. Bootstrap Bash Script

- **Purpose:**  
  Automates installation of all required dependencies (Docker, Docker Compose, PostgreSQL client, Kubernetes tools) in the VM.

- **bootstrap.sh**

```sh
  #!/usr/bin/env bash

set -e  # exit on error
set -u  # treat unset vars as errors
set -o pipefail # catch errors in pipelines

GREEN='\033[0;32m'
NC='\033[0m'

log() {
    echo -e "${GREEN}[+] $1${NC}"
}

update_system() {
    log "Updating system packages..."
    sudo apt-get update -y
    sudo apt-get upgrade -y
}

install_basic_tools() {
    log "Installing basic tools (curl, wget, unzip, git, make, python3, pip, venv)..."
    sudo apt-get install -y \
        curl wget unzip git make \
        python3 python3-pip python3-venv \
        software-properties-common apt-transport-https ca-certificates gnupg lsb-release
}

install_postgres_client() {
    log "Installing PostgreSQL client..."
    sudo apt-get install -y postgresql-client
}

install_docker() {
    log "Installing Docker CE..."
    sudo apt-get remove -y docker docker-engine docker.io containerd runc || true

    # Add Docker’s official GPG key
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker.gpg

    # Set up stable repository
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
      $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    sudo apt-get update -y
    sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

    sudo systemctl enable docker
    sudo usermod -aG docker vagrant || true
}

install_kubernetes_tools() {
    log "Installing Kubernetes tools (kubectl, minikube)..."

    # Install kubectl
    curl -LO "https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    chmod +x kubectl
    sudo mv kubectl /usr/local/bin/

    # Install minikube
    curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
    sudo install minikube-linux-amd64 /usr/local/bin/minikube
    rm -f minikube-linux-amd64
}

install_kubernetes_tools() {
    log "Installing Kubernetes tools (kubectl, minikube, kind)..."
    # Install kubectl
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
    rm kubectl

    # Install minikube
    curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
    sudo install minikube-linux-amd64 /usr/local/bin/minikube
    rm minikube-linux-amd64

}

main() {
    update_system
    install_basic_tools
    install_postgres_client
    install_docker
    install_kubernetes_tools

    log "Bootstrap completed!"
    log "Please log out and log back in (or run 'newgrp docker') to use Docker without sudo."
    log "Run 'minikube start --driver=docker --nodes 3' to start your Kubernetes cluster."
    log "After that, you can label nodes with:"
    echo "kubectl label node minikube type=application"
    echo "kubectl label node minikube-m02 type=database"
    echo "kubectl label node minikube-m03 type=dependent_services"
}

main "$@"
```

- **Run script inside your Vagrant box:**
  ```sh
  chmod +x bootstrap.sh
  ./bootstrap.sh
  ```

- **Script highlights:**
  - Updates system packages
  - Installs basic tools (`curl`, `wget`, `git`, `make`, `python3`, etc.)
  - Installs Docker CE and Docker Compose plugin
  - Installs PostgreSQL client
  - Installs Kubernetes tools (optional for future use)
  - Adds user to Docker group

---

## 4. Docker Compose Deployment

- **Compose file includes:**  
  - **flask-app**: 2 replicas of the API server
  - **postgres**: single DB container
  - **nginx**: reverse proxy and load balancer

- **Example docker-compose.yml:**
    ```yaml
    version: '3.8'
    services:
      flask-app:
        image: flask-app:1.0.0
        deploy:
          replicas: 2
        env_file:
          - .env
        depends_on:
          - postgres
        ports:
          - "5000"
        networks:
          - backend

      postgres:
        image: postgres:15
        env_file:
          - .env
        volumes:
          - pgdata:/var/lib/postgresql/data
        ports:
          - "5432:5432"
        networks:
          - backend

      nginx:
        image: nginx:alpine
        ports:
          - "8080:80"
        volumes:
          - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
        depends_on:
          - flask-app
        networks:
          - backend

    volumes:
      pgdata:

    networks:
      backend:
    ```

- **Deploy stack:**
  ```sh
  docker compose up -d
  ```

---

## 5. Nginx Configuration for Load Balancing

- **nginx.conf** (committed to repo):

  ```nginx
  events {}

  http {
      upstream flask_backend {
          server flask-app:5000;
          server flask-app:5001; # If mapped differently
      }

      server {
          listen 80;

          location / {
              proxy_pass http://flask_backend;
              proxy_set_header Host $host;
              proxy_set_header X-Real-IP $remote_addr;
              proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          }
      }
  }
  ```

- **How it works:**  
  Nginx forwards incoming requests on port 8080 (`host:8080`) to the two API containers, distributing load in round-robin fashion.

---

## 6. Makefile Integration

- **Recommended targets:**
  - `make build`: Build Docker images
  - `make up`: Start containers via Docker Compose
  - `make down`: Stop and remove containers
  - `make logs`: View logs
  - `make status`: Show status

- **Example:**
  ```makefile
  build:
      docker compose build

  up:
      docker compose up -d

  down:
      docker compose down

  logs:
      docker compose logs

  status:
      docker compose ps
  ```

---

## 7. Testing & Verification

- **API endpoints:**  
  - Test with Postman collection or `curl`.
  - All endpoints should return status code 200.

  ```sh
  curl -i http://localhost:8080/healthcheck
  curl -i http://localhost:8080/api/v1/students
  ```

- **Check container status:**
  ```sh
  docker compose ps
  ```

- **Nginx logs (for load balancing):**
  ```sh
  docker compose logs nginx
  ```

---

## 8. Troubleshooting

- **If containers fail to start:**  
  - Check logs with `docker compose logs <service>`
  - Ensure ports are not already in use
  - Verify `.env` variables are correct

- **If Nginx does not load balance:**  
  - Check upstream configuration in `nginx.conf`
  - Ensure both API containers are running

- **Common commands:**
  ```sh
  docker compose logs
  docker compose exec nginx nginx -t
  docker compose exec flask-app curl localhost:5000/healthcheck
  ```

---

## 9. References

- [Vagrant Documentation](https://www.vagrantup.com/docs)
- [Docker Compose](https://docs.docker.com/compose/)
- [Nginx Load Balancing Guide](https://nginx.org/en/docs/http/load_balancing.html)
- [Makefile Guide](https://www.gnu.org/software/make/manual/make.html)

---
**Deploying with Vagrant + Docker Compose + Nginx is a robust way to simulate production for your REST API stack!**
