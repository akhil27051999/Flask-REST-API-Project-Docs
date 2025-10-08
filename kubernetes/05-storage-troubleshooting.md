## Storage Troubleshooting Guide

**This guide addresses storage-related issues encountered while running Docker, Minikube, and EC2 instances for development and Kubernetes clusters.**

### 1. Check Disk Usage

#### Commands:
```sh
df -h           # View disk usage per filesystem
sudo du -xh / | sort -rh | head -50  # Largest directories/files
```

#### Common issue:
- / or /var is at 100% → Minikube, Docker, or OS cannot write new data.

#### Resolution:
- Identify largest directories with du and remove unnecessary files.

### 2. Docker Storage Issues

#### 2.1 Docker Images and Containers Fill Disk

#### Symptoms:

- Docker is out of disk space!
- Minikube fails to start because Docker storage is full.

#### Checks:
```sh
docker system df   # Show Docker usage
docker images      # List all images
docker ps -a       # List all containers
docker volume ls   # List volumes
```

#### Resolution:
- Remove unused images, containers, and networks:
```sh
docker system prune -a -f
docker volume prune -f
docker network prune -f
```

- Stop and remove containers using target images (if docker rmi fails):
```sh
docker ps -a
docker stop <container_id>
docker rm <container_id>
docker rmi <image_id>
```
- Use cleanup scripts (cleanup.sh) to automate container/image removal.

### 2.2 Overlay2 Storage

#### Cause:
- /var/lib/docker/overlay2 can grow large due to container layers, especially for Minikube multi-node clusters.

#### Resolution:
- Prune unused containers, volumes, and images.
- For stubborn data, manually remove old overlay directories:
```sh
sudo rm -rf /var/lib/docker/overlay2/<id>
```

### 3. Minikube Storage Issues

#### Symptoms:
- Minikube cannot start multiple nodes.
- Logs show:
  - write lastStart.txt: no space left on device
- Resolution:
  - Delete old clusters:
  ```sh
  minikube delete --all
  ```
  - Clear Minikube cache:
  ```sh
  rm -rf ~/.minikube/cache
  rm -rf ~/.minikube/machines
  ```

- Check volume sizes:
  ```sh
  docker volume ls
  docker volume rm <volume_name>
  ```
  - Reduce cluster node resources (memory/CPUs) to fit system.

### 4. EC2 Instance Disk Issues

#### Symptoms:
- Root disk / fills up quickly.
- Cannot start Minikube nodes or containers.
- Checks:
  ```sh
  df -h       # Check disk usage
  lsblk       # Check block devices
  ```

- Resolution:
  - Increase EBS volume size in AWS:
  - Navigate to EC2 → Volumes → Modify Volume
  - Increase from 8GB → 30GB (or more depending on cluster needs)
  - Resize the filesystem on Ubuntu:
  ```sh
  sudo growpart /dev/nvme0n1 1
  sudo resize2fs /dev/nvme0n1p1
  df -h
  ```

- Remove unnecessary files from /var, /home, or /usr:
  ```sh
  sudo apt-get clean
  sudo apt-get autoremove -y
  ```

- Move large caches (like Minikube .minikube/cache) to a bigger volume if needed.

### 5. Recommendations for Multi-Node Minikube Cluster

- Each node requires storage for:
  - Docker images
  - Overlay2 layers
  - Minikube VM files
  - Minimum disk recommendation: 30GB root volume
- For limited space, reduce Minikube node count or prune old data frequently.
- Use separate volumes for Docker and Minikube caches if possible.

### 6. Quick Commands Summary

#### Disk usage
```sh
df -h
sudo du -xh / | sort -rh | head -50
```
#### Docker cleanup
```sh
docker system prune -a -f
docker volume prune -f
docker network prune -f
docker stop $(docker ps -aq)
docker rm $(docker ps -aq)
```

#### Minikube cleanup
```sh
minikube delete --all
rm -rf ~/.minikube/cache
rm -rf ~/.minikube/machines
```
#### EC2 volume resize
```sh
sudo growpart /dev/nvme0n1 1
sudo resize2fs /dev/nvme0n1p1
```
