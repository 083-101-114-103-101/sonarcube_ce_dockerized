# SonarQube Community Build with PostgreSQL

This project provides a production-ready **SonarQube Community Build (2025.x)** environment using **PostgreSQL 15**. It is optimized for internal communication with a **GitLab Runner** located on the same host.

---

## 📋 Step 1: Host System Preparation (Mandatory)

SonarQube's embedded Elasticsearch engine requires specific kernel limits. These **must** be set on the host machine (not inside the container).

### 1.1 Apply Changes Immediately
Run these commands with root privileges:

```bash
# Increase memory map areas
sudo sysctl -w vm.max_map_count=524288

# Increase file descriptor limits
sudo sysctl -w fs.file-max=131072
```

### 1.2 Make Changes Permanent
To ensure these settings survive a reboot, edit `/etc/sysctl.conf`:

```bash
sudo nano /etc/sysctl.conf
```

Add these lines to the bottom:

```text
# SonarQube / Elasticsearch requirements
vm.max_map_count=524288
fs.file-max=131072
```

Apply the file: `sudo sysctl -p`

---

## 🚀 Step 2: Deployment

**Repo:** [083-101-114-103-101/sonarcube_ce_dockerized](https://github.com/083-101-114-103-101/sonarcube_ce_dockerized)

### 2.1 Clone and Prepare
```bash
# Clone the repository
git clone https://github.com/083-101-114-103-101/sonarcube_ce_dockerized.git

# Enter the directory
cd sonarcube_ce_dockerized

# Ensure you are on the correct revision
git checkout 90200c46e2e5695f60e54bbc187cbcc0cef270bd
```

### 2.2 Start the Stack
The configuration uses this [docker-compose.yml](https://github.com/083-101-114-103-101/sonarcube_ce_dockerized/blob/90200c46e2e5695f60e54bbc187cbcc0cef270bd/docker-compose.yml) to orchestrate SonarQube and PostgreSQL.

Run:
```bash
docker compose up -d
```

Access the interface at **http://localhost:9000**.  
**Default Credentials:** `admin` / `admin` (change password immediately).

---

## 🦊 Step 3: GitLab Runner Integration

Since the GitLab Runner is on the same host, we connect it to the internal network defined in the compose file (`sonarqube_network`).

### 3.1 Update GitLab Runner Config
Edit your `/etc/gitlab-runner/config.toml`:

```toml
[[runners]]
  name = "Your Runner Name"
  executor = "docker"
  [runners.docker]
    # Connect the runner to the SonarQube network
    network_mode = "sonarqube_network"
```

Restart the runner: `sudo gitlab-runner restart`

### 3.2 Configure GitLab CI/CD Variables
In your GitLab project (**Settings > CI/CD > Variables**), add:
* `SONAR_HOST_URL`: `http://sonarqube:9000` (Internal DNS name from the compose file)
* `SONAR_TOKEN`: (Generate in SonarQube: *My Account > Security > Tokens*)

### 3.3 Example Pipeline (`.gitlab-ci.yml`)
```yaml
sonarqube-check:
  stage: test
  image: 
    name: sonarsource/sonar-scanner-cli:latest
    entrypoint: [""]
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
    GIT_DEPTH: "0"
  script: 
    - sonar-scanner -Dsonar.projectKey=Your_Project_Key -Dsonar.sources=.
  allow_failure: true
```

---

## 🛠 Step 4: Maintenance

* **Check Logs:** `docker compose logs -f sonarqube`
* **Stop Services:** `docker compose stop`
* **Internal Connectivity Test:** `docker exec -it <runner_container_id> ping sonarqube`

