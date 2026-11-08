# Clean Install: Docker, Minikube, and Jenkins on Ubuntu

A step-by-step guide to remove older installations and install the
latest stable releases of Docker Engine, Minikube, and Jenkins LTS on
Ubuntu.

> **Scope:** Ubuntu 22.04 LTS and 24.04 LTS, 64-bit (`amd64` or
> `arm64`).
>
> **Important:** This guide includes optional commands that permanently
> delete Docker images, containers, volumes, Minikube clusters, and
> Jenkins jobs/configuration. Back up anything you need before
> proceeding. Do not run the data-deletion commands on a machine that
> contains data you want to keep.

## 1. What you will install

-   **Docker Engine** from Docker's official APT repository.
-   **Minikube** from the official Kubernetes Minikube release download.
-   **Jenkins LTS** from the official Jenkins Debian repository.
-   **Java 21** (required by current Jenkins releases).

The commands install the latest stable package versions available from
the official repositories at installation time. They do not pin a
version number.

## 2. Check your Ubuntu version and architecture

``` bash
cat /etc/os-release
dpkg --print-architecture
uname -m
```

This guide is intended for Ubuntu 22.04/24.04 on `amd64` or `arm64`.

Update package metadata and install common utilities:

``` bash
sudo apt update
sudo apt install -y ca-certificates curl wget gnupg
```

## 3. Record existing versions (if installed)

Run these commands before removing anything:

``` bash
docker --version 2>/dev/null || true
docker compose version 2>/dev/null || true
minikube version 2>/dev/null || true
java -version 2>&1 || true
jenkins --version 2>/dev/null || true
```

You can also inspect installed packages:

``` bash
dpkg -l | grep -E 'docker|containerd|minikube|jenkins|openjdk' || true
```

## 4. Remove older installations

### 4.1 Remove old Docker packages

Stop Docker if it is running:

``` bash
sudo systemctl stop docker docker.socket containerd 2>/dev/null || true
```

Remove conflicting Ubuntu/Debian packages and old Docker Engine
packages:

``` bash
sudo apt remove -y docker.io docker-compose docker-compose-v2 docker-doc \
  podman-docker docker-buildx docker-ce docker-ce-cli \
  containerd containerd.io runc docker-buildx-plugin \
  docker-compose-plugin docker-ce-rootless-extras
```

Purge remaining official Docker packages, if present:

``` bash
sudo apt purge -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras
```

**Optional --- permanently delete Docker data.** This removes all local
containers, images, volumes, networks, and containerd data. Skip this if
you need to preserve them.

``` bash
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
```

Remove old Docker APT repository/key files (if present):

``` bash
sudo rm -f /etc/apt/sources.list.d/docker.list
sudo rm -f /etc/apt/sources.list.d/docker.sources
sudo rm -f /etc/apt/keyrings/docker.asc
sudo rm -f /etc/apt/keyrings/docker.gpg
```

### 4.2 Remove old Minikube

If the existing Minikube command is available, delete its clusters and
profiles:

``` bash
minikube delete --all --purge 2>/dev/null || true
```

Remove a package-based installation, if present:

``` bash
sudo apt purge -y minikube
```

Remove common manually installed binaries:

``` bash
sudo rm -f /usr/local/bin/minikube
sudo rm -f /usr/bin/minikube
```

**Optional --- permanently remove Minikube's local user data**
(profiles, cached images, and configuration):

``` bash
rm -rf "$HOME/.minikube"
```

Do not remove `$HOME/.kube` unless you intentionally want to delete
Kubernetes client configuration that may be used by other clusters.

### 4.3 Remove old Jenkins

Stop Jenkins and remove its package:

``` bash
sudo systemctl stop jenkins 2>/dev/null || true
sudo apt purge -y jenkins
```

Remove old Jenkins repository and key files so the current official
repository can be configured cleanly:

``` bash
sudo rm -f /etc/apt/sources.list.d/jenkins.list
sudo rm -f /etc/apt/keyrings/jenkins-keyring.asc
```

**Optional --- permanently delete Jenkins data.** This removes jobs,
build history, plugins, credentials, and configuration. Back up
`$JENKINS_HOME` first if you need any of it.

``` bash
sudo rm -rf /var/lib/jenkins
sudo rm -rf /var/cache/jenkins
sudo rm -rf /var/log/jenkins
```

### 4.4 Refresh APT

``` bash
sudo apt autoremove -y
sudo apt clean
sudo apt update
```

## 5. Install Docker Engine (official Docker repository)

### 5.1 Add Docker's official signing key and APT repository

``` bash
sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

### 5.2 Install the latest stable Docker packages

``` bash
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

Enable and start Docker:

``` bash
sudo systemctl enable --now docker
sudo systemctl status docker --no-pager
```

Verify the installation:

``` bash
sudo docker run hello-world
docker --version
docker compose version
```

### 5.3 (Optional) Run Docker without `sudo`

Add your current user to the `docker` group:

``` bash
sudo groupadd docker 2>/dev/null || true
sudo usermod -aG docker "$USER"
```

Log out and log back in (or reboot) for the group change to take effect.
Then verify:

``` bash
docker run hello-world
```

> **Security note:** Membership in the `docker` group grants root-level
> privileges on the host. Only add trusted users.

## 6. Install Minikube (official Kubernetes release)

### 6.1 Download and install the latest stable binary

The following selects the correct official Minikube binary for `amd64`
or `arm64`:

``` bash
ARCH=$(dpkg --print-architecture)

case "$ARCH" in
  amd64)
    MINIKUBE_ARCH="amd64"
    ;;
  arm64)
    MINIKUBE_ARCH="arm64"
    ;;
  *)
    echo "Unsupported architecture for this command: $ARCH"
    exit 1
    ;;
esac

curl -LO "https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-${MINIKUBE_ARCH}"

sudo install "minikube-linux-${MINIKUBE_ARCH}" /usr/local/bin/minikube
rm -f "minikube-linux-${MINIKUBE_ARCH}"
```

Verify:

``` bash
minikube version
```

### 6.2 Start a Kubernetes cluster using Docker

Make sure Docker is running and your user can access it. If you have not
configured Docker group access, run Minikube with `sudo` only if you
understand the resulting root-owned profile implications; preferably log
out/in after adding your user to the Docker group.

Start Minikube:

``` bash
minikube start --driver=docker
```

Check cluster status:

``` bash
minikube status
```

### 6.3 Use kubectl

Minikube can invoke a compatible `kubectl` without a separate
installation:

``` bash
minikube kubectl -- get nodes
minikube kubectl -- get pods -A
```

Optional: install the standalone `kubectl` using the official Kubernetes
instructions:
https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

## 7. Install Jenkins LTS (official Jenkins repository)

### 7.1 Install Java 21 first

Current Jenkins releases require Java 21 or later. Install the Java 21
runtime:

``` bash
sudo apt update
sudo apt install -y fontconfig openjdk-21-jre
java -version
```

### 7.2 Add the official Jenkins LTS repository

``` bash
sudo install -m 0755 -d /etc/apt/keyrings

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
```

### 7.3 Install Jenkins LTS

``` bash
sudo apt install -y jenkins
```

Enable and start the service:

``` bash
sudo systemctl enable --now jenkins
sudo systemctl status jenkins --no-pager
```

Jenkins listens on TCP port `8080` by default.

### 7.4 Unlock Jenkins in your browser

Open:

``` text
http://localhost:8080
```

Get the initial administrator password:

``` bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy the password into the Jenkins setup page, install the suggested
plugins, and create your first administrator account.

If Jenkins does not start, inspect its logs:

``` bash
sudo journalctl -u jenkins -n 100 --no-pager
```

If port 8080 is already in use, identify the process:

``` bash
sudo ss -ltnp | grep ':8080'
```

Resolve the port conflict or configure Jenkins to use another port
before restarting it.

## 8. Final verification

Run the following commands:

``` bash
docker --version
docker compose version
sudo systemctl is-active docker

minikube version
minikube status
minikube kubectl -- get nodes

java -version
jenkins --version
sudo systemctl is-active jenkins
```

Expected results: - Docker service reports `active`. - Minikube reports
its host, kubelet, and API server as running; the node is `Ready`. -
Java reports version 21 or later. - Jenkins service reports `active`;
the web interface is available at `http://localhost:8080`.

## 9. Useful commands

  Task                      Command
  ------------------------- ----------------------------------
  Stop Minikube             `minikube stop`
  Start Minikube            `minikube start --driver=docker`
  Delete Minikube cluster   `minikube delete`
  View Kubernetes nodes     `minikube kubectl -- get nodes`
  View Docker containers    `docker ps -a`
  View Docker images        `docker images`
  Restart Docker            `sudo systemctl restart docker`
  Restart Jenkins           `sudo systemctl restart jenkins`
  View Jenkins logs         `sudo journalctl -u jenkins -f`
  Check Jenkins status      `sudo systemctl status jenkins`

## 10. Troubleshooting

### Docker permission denied

If `docker ps` returns a permission error: 1. Confirm Docker is running:
`sudo systemctl status docker`. 2. If you added yourself to the `docker`
group, log out and log in again. 3. Test with `sudo docker ps` to
distinguish a permissions issue from a daemon issue.

### Minikube cannot connect to Docker

Check:

``` bash
sudo systemctl status docker
docker info
```

Confirm your user can run Docker commands without `sudo`, then retry:

``` bash
minikube start --driver=docker
```

### Minikube node is not Ready

``` bash
minikube status
minikube logs
minikube kubectl -- get nodes
```

### Jenkins is not starting

``` bash
sudo systemctl status jenkins --no-pager
sudo journalctl -u jenkins -n 100 --no-pager
java -version
sudo ss -ltnp | grep ':8080'
```

Confirm Java 21 or later is installed and port 8080 is available.

## 11. Official documentation and repositories

Always consult these pages for updated commands, supported Ubuntu
versions, release notes, and troubleshooting:

-   Docker Engine on Ubuntu:
    https://docs.docker.com/engine/install/ubuntu/
-   Docker Engine GitHub documentation: https://github.com/docker/docs
-   Minikube installation and quick start:
    https://minikube.sigs.k8s.io/docs/start/
-   Minikube releases: https://github.com/kubernetes/minikube/releases
-   Minikube delete command:
    https://minikube.sigs.k8s.io/docs/commands/delete/
-   Jenkins on Linux: https://www.jenkins.io/doc/book/installing/linux/
-   Jenkins Debian/Ubuntu packages:
    https://pkg.jenkins.io/debian-stable/

------------------------------------------------------------------------

**Maintenance:** These instructions intentionally use official
`latest`/stable package channels rather than hard-coded versions.
Re-check the official documentation before repeating the procedure,
because repository keys, supported Ubuntu releases, and prerequisites
can change.
