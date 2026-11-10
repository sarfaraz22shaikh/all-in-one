# Docker, Docker Compose, Kubernetes & Docker Swarm Command Reference

Hands-on command reference and lab notes for containers and orchestration.

**Platform:** Ubuntu Linux

## Contents

1. [Docker](#1-docker)
2. [Volumes and Data Persistence](#2-volumes-and-data-persistence)
3. [Dockerfile and Custom Images](#3-dockerfile-and-custom-images)
4. [Docker Compose](#4-docker-compose)
5. [Kubernetes and Minikube](#5-kubernetes-and-minikube)
6. [Docker Swarm](#6-docker-swarm)
7. [Swarm Concepts and Limitations](#7-swarm-concepts-and-limitations)
8. [Quick Revision](#8-quick-revision)

## Prerequisites

- Ubuntu Linux with Docker installed and running
- User in the `docker` group (`sudo usermod -aG docker $USER`, then log out and in)
- Docker Compose plugin (`docker compose version`; if missing: `sudo apt install -y docker-compose-plugin`)
- Docker Hub account (only for the push lab)

---

## 1. Docker

### Images and containers

| Lab | Commands |
|---|---|
| Pull and run Nginx | `docker pull nginx`, `docker run -d --name mynginx nginx`, `docker ps`, `docker stop mynginx`, `docker rm mynginx`, `docker ps -a` |
| Image commands | `docker pull alpine`, `docker images`, `docker image ls`, `docker inspect alpine`, `docker rmi alpine`, `docker rmi -f alpine` |
| Interactive Ubuntu | `docker run -it --name ubuntu_container ubuntu bash` → `pwd`, `ls -la`, `whoami`, `cat /etc/os-release`, `exit` → `docker start ubuntu_container`, `docker exec -it ubuntu_container bash` |
| Rename container | `docker run -d --name web_server nginx`, `docker rename web_server my_server` |
| Tag images | `docker tag nginx mynginx:v1`, `docker tag nginx mynginx:latest`, `docker image inspect mynginx:v1`, `docker rmi mynginx:v1` |

Notes:
- Docker prevents removing an image used by a running container. Stop and remove the container first.
- `docker rmi -f` can untag an image but does not stop the running container.
- Removing one tag does not delete the image data if another tag references it.

### Push to Docker Hub

```bash
docker login
docker pull nginx
docker tag nginx YOUR_USERNAME/mynginx:v1
docker push YOUR_USERNAME/mynginx:v1
docker logout
```

Verify at `https://hub.docker.com/repositories/YOUR_USERNAME`.

### Networking and port mapping

```bash
docker run -d --name nginx_server -p 8080:80 nginx
# open http://localhost:8080
docker network ls
docker network inspect bridge
docker inspect nginx_server
docker stop nginx_server && docker rm nginx_server
```

Syntax: `-p HOST_PORT:CONTAINER_PORT`

---

## 2. Volumes and Data Persistence

### Named volume

```bash
docker volume create data_vol
docker volume ls
docker volume inspect data_vol
docker run -it --name volume_container -v data_vol:/data ubuntu bash
# inside: cd /data && echo "Docker volume persistence test" > test.txt && cat test.txt && exit
docker rm volume_container
docker run -it --name new_volume_container -v data_vol:/data ubuntu bash
# inside: cat /data/test.txt  (file persists)
docker rm new_volume_container
docker volume rm data_vol
```

### Bind mount

```bash
mkdir -p ~/docker-bind
echo "Hello from host machine" > ~/docker-bind/data.txt
docker run -it --name bind_container -v ~/docker-bind:/data ubuntu bash
# inside: cat /data/data.txt; echo "Created inside container" > /data/container.txt; exit
cat ~/docker-bind/container.txt
docker rm bind_container
```

| | Bind mount | Named volume |
|---|---|---|
| Location | Host directory chosen by user | Managed by Docker |
| Best for | Development | Persistent application data |
| Persists after container removal | Yes | Yes |

### MySQL with persistent volume

```bash
docker pull mysql
docker volume create mysql_data
docker run -d \
  --name mysql_container \
  -e MYSQL_ROOT_PASSWORD=root123 \
  -e MYSQL_DATABASE=testdb \
  -v mysql_data:/var/lib/mysql \
  -p 3306:3306 \
  mysql
docker logs -f mysql_container
docker exec -it mysql_container mysql -uroot -p
```

```sql
USE testdb;
CREATE TABLE students (id INT PRIMARY KEY, name VARCHAR(100));
INSERT INTO students VALUES (1, 'Rahul'), (2, 'Amit'), (3, 'John');
SELECT * FROM students;
```

Restart (`docker restart mysql_container`) and re-run `SELECT` to verify persistence.
Cleanup: `docker stop mysql_container && docker rm mysql_container`; remove data with `docker volume rm mysql_data` only when no longer needed.

### Shared volume between containers

```bash
docker volume create shared_vol
docker run -it --name container1 -v shared_vol:/data ubuntu bash   # write /data/shared.txt
docker run -it --name container2 -v shared_vol:/data ubuntu bash   # read and append
docker start container1 && docker exec -it container1 cat /data/shared.txt
docker rm -f container1 container2 && docker volume rm shared_vol
```

---

## 3. Dockerfile and Custom Images

All labs follow: create directory → write app + `Dockerfile` → build → run → logs → remove.

| Lab | Base image | Dockerfile essentials | Build / run |
|---|---|---|---|
| Python | `python:3.12-slim` | `WORKDIR /app`, `COPY app.py .`, `CMD ["python", "app.py"]` | `docker build -t python-app .` / `docker run --name python-container python-app` |
| Java | `eclipse-temurin:21-jdk` | `COPY Main.java .`, `RUN javac Main.java`, `CMD ["java", "Main"]` | `docker build -t java-app .` / `docker run --name java-container java-app` |
| Nginx site | `nginx:latest` | `COPY index.html /usr/share/nginx/html/index.html`, `EXPOSE 80` | `docker build -t custom-nginx .` / `docker run -d --name custom-nginx-container -p 8080:80 custom-nginx` |
| Shell script | `ubuntu:24.04` | `COPY script.sh .`, `RUN chmod +x script.sh`, `CMD ["./script.sh"]` | `docker build -t shell-app .` / `docker run --name shell-container shell-app` |

Example (Python):

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
CMD ["python", "app.py"]
```

---

## 4. Docker Compose

### Nginx static website

```yaml
services:
  nginx:
    image: nginx:latest
    container_name: compose-nginx
    ports:
      - "8080:80"
    volumes:
      - ./index.html:/usr/share/nginx/html/index.html:ro
    restart: unless-stopped
```

```bash
docker compose up -d
docker compose ps
docker compose logs
docker compose down
```

### Python script

```yaml
services:
  python-app:
    build: .
    container_name: python-compose-container
```

Run with `docker compose up --build`. The container exits when the script finishes.

### MySQL with persistent volume

```yaml
services:
  mysql:
    image: mysql:8.4
    container_name: compose-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: testdb
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
```

- `docker compose down` keeps the named volume.
- `docker compose down -v` removes containers and volume data.

### Three Nginx containers, one shared volume

Containers mounting the same volume at `/usr/share/nginx/html` cannot show different content. This lab uses one shared volume with a subdirectory per container: an `init` service (alpine) copies `nginx1/2/3/index.html` into `/shared/nginxN/`; each Nginx service waits on `init` (`condition: service_completed_successfully`), copies its own file to its web root, then starts Nginx.

| Service | Port |
|---|---|
| nginx1 | 8081 |
| nginx2 | 8082 |
| nginx3 | 8083 |

### Useful Compose commands

`up`, `up -d`, `up --build`, `ps`, `ps -a`, `logs`, `logs -f`, `restart`, `stop`, `start`, `down`, `down -v`, `pull`, `build` (all prefixed with `docker compose`).

---

## 5. Kubernetes and Minikube

```bash
# Prerequisites
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget apt-transport-https
sudo systemctl enable --now docker
sudo usermod -aG docker $USER      # log out and in afterwards

# Install (amd64 shown; use minikube-linux-arm64 on arm64)
curl -LO "https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64"
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
minikube version

# Run
minikube start --driver=docker
minikube status
minikube kubectl -- get nodes
minikube kubectl -- get pods -A
minikube kubectl -- cluster-info
minikube dashboard
```

| Action | Command |
|---|---|
| Stop / start / delete | `minikube stop` / `minikube start` / `minikube delete` |
| Profiles, IP, logs | `minikube profile list`, `minikube ip`, `minikube logs` |
| Addons | `minikube addons list`, `minikube addons enable dashboard`, `minikube addons enable metrics-server` |

Docs: https://minikube.sigs.k8s.io/docs/start/ · https://kubernetes.io/docs/home/

---

## 6. Docker Swarm

### Initialize

```bash
docker --version
docker swarm init
docker info
docker node ls
```

### Nginx service

```bash
docker service create \
  --name nginx-service \
  --replicas 2 \
  --publish published=8080,target=80 \
  nginx
docker service ls
docker service ps nginx-service
docker service inspect nginx-service
docker service logs nginx-service
# open http://localhost:8080
```

### Python worker service

```bash
docker build -t python-swarm:v1 .
docker service create --name python-service --replicas 3 python-swarm:v1
docker service logs -f python-service
docker ps --filter name=python-service
docker ps -q --filter name=python-service
docker logs CONTAINER_ID
```

`app.py` loops printing `Running in Swarm...` every 5 seconds; the Dockerfile uses `CMD ["python", "-u", "app.py"]`.

### Scale

```bash
docker service create --name nginx-service --replicas 2 nginx
docker service scale nginx-service=5
docker service scale nginx-service=1
docker service ls
docker service ps nginx-service
```

Scaling down stops and removes excess tasks.

### Rolling update and rollback

```bash
docker service create \
  --name nginx-service \
  --replicas 3 \
  --update-parallelism 1 \
  --update-delay 10s \
  nginx:1.25

docker service update --image nginx:latest nginx-service
docker service ps nginx-service
docker service rollback nginx-service
```

- `--update-parallelism 1`: update one task at a time.
- `--update-delay 10s`: wait 10 seconds between task updates.

### Cleanup

```bash
docker service rm nginx-service
docker service rm python-service
docker service ls
docker swarm leave --force
docker info
```

---

## 7. Swarm Concepts and Limitations

### Uses

| Capability | Description |
|---|---|
| Orchestration | Manages containers across multiple machines; maintains desired replica count; restarts failed tasks |
| Service deployment | Deploys apps as services with image and replica count |
| Scaling | `docker service scale web=5` |
| Load balancing | Built-in routing mesh across replicas via published ports |
| Rolling updates | Gradual updates with delay, parallelism, and rollback |
| High availability | Multiple replicas; manager availability via Raft consensus with multiple managers |
| Multi-node | Manager and worker nodes in one cluster |

### Limitations (compared with Kubernetes)

- Fewer advanced orchestration features
- Smaller ecosystem and community; less common in large production environments
- No native Horizontal Pod Autoscaling; scaling is manual or externally automated
- Basic overlay networking, with fewer policy and networking features
- Basic volume management; less advanced persistent storage orchestration
- Fewer monitoring, observability, and security integrations

---

## 8. Quick Revision

### Docker

| Task | Command |
|---|---|
| Pull / list / remove image | `docker pull IMAGE` / `docker images` / `docker rmi IMAGE` |
| Run (foreground / background / interactive) | `docker run IMAGE` / `docker run -d IMAGE` / `docker run -it IMAGE bash` |
| List running / all containers | `docker ps` / `docker ps -a` |
| Stop / start / restart | `docker stop NAME` / `docker start NAME` / `docker restart NAME` |
| Remove / force remove | `docker rm NAME` / `docker rm -f NAME` |
| Rename | `docker rename OLD NEW` |
| Exec / logs / inspect | `docker exec -it NAME bash` / `docker logs NAME` / `docker inspect NAME` |
| Volumes | `docker volume create NAME`, `docker volume ls`, `docker volume rm NAME` |
| Networks | `docker network ls` |
| Build / tag / push | `docker build -t NAME .` / `docker tag SOURCE TARGET` / `docker push IMAGE` |
| Login / logout | `docker login` / `docker logout` |

### Compose

`docker compose up -d` · `up --build` · `down` · `down -v` · `ps` · `logs` · `logs -f` · `restart` · `stop` · `start` · `pull` · `build`

### Swarm

`docker swarm init` · `docker swarm leave --force` · `docker node ls` · `docker service create|ls|ps|inspect|logs|scale|update|rollback|rm`

---

## Cleanup reminder

Remove containers, services, and volumes when a lab is finished. Use `-v` / `docker volume rm` only when the data is no longer needed.
