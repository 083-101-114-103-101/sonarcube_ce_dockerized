# sonarcube_ce_dockerized
This setup deploys a SonarQube Community Build instance (Version 2025.x) using PostgreSQL 15 as the backend database. It is configured to run on a Linux host that also houses a GitLab Runner, allowing for high-speed, internal container-to-container communication.


# SonarQube Community Build with PostgreSQL

This project provides a production-ready **SonarQube Community Build (2025.x)** environment using **PostgreSQL 15** as a database backend. It is optimized for high-speed, internal communication with a **GitLab Runner** located on the same host.

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

## 🚀 Step 2: Deployment with Docker Compose

Create a directory (e.g., `~/sonarqube`) and save the following content as `docker-compose.yml`.

### 2.1 The Docker Compose File

```yaml
networks:
  sonarqube_network:
    name: sonarqube_network
    driver: bridge

services:
  db:
    image: postgres:15
    container_name: sonarqube_db
    restart: always
    networks:
      - sonarqube_network
    environment:
      - POSTGRES_USER=sonar
      - POSTGRES_PASSWORD=sonar
      - POSTGRES_DB=sonar
    volumes:
      - postgresql_data:/var/lib/postgresql/data

  sonarqube:
    image: sonarqube:2025.1-community
    container_name: sonarqube_app
    depends_on:
      - db
    restart: always
    networks:
      - sonarqube_network
    ports:
      - "9000:9000"
    environment:
      - SONAR_JDBC_URL=jdbc:postgresql://db:5432/sonar
      - SONAR_JDBC_USERNAME=sonar
      - SONAR_JDBC_PASSWORD=sonar
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_logs:/opt/sonarqube/logs
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_temp:/opt/sonarqube/temp
    # Required for Elasticsearch 8.x stability
    tmpfs:
      - /tmp:size=256M,mode=1777

volumes:
  postgresql_data:
  sonarqube_data:
  sonarqube_logs:
  sonarqube_extensions:
  sonarqube_temp:
```

### 2.2 Start the Stack
Run:

```bash
docker compose up -d
```

Access the interface at **http://localhost:9000**.  
**Default Credentials:** `admin` / `admin` (change password immediately).

---

## 🦊 Step 3: GitLab Runner Integration

Since the GitLab Runner is on the same host, we connect it to the `sonarqube_network` to allow internal container-to-container communication.

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
* `SONAR_HOST_URL`: `http://sonarqube_app:9000` (Internal DNS name)
* `SONAR_TOKEN`: (Generate in SonarQube: *My Account > Security > Tokens*)

### 3.3 Example Pipeline (`.gitlab-ci.yml`)
Add this to your repository:

```yaml
sonarqube-check:
  stage: test
  image: 
    name: sonarsource/sonar-scanner-cli:latest
    entrypoint: [""]
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
    GIT_DEPTH: "0"
  cache:
    key: "${CI_JOB_NAME}"
    paths:
      - .sonar/cache
  script: 
    - sonar-scanner -Dsonar.projectKey=Your_Project_Key -Dsonar.sources=.
  allow_failure: true
```

---

## 🛠 Step 4: Troubleshooting & Maintenance

* **Check Logs:** `docker compose logs -f sonarqube`
* **Restart Services:** `docker compose restart`
* **Clean Removal (Data Loss!):** `docker compose down -v`
* **Internal Connectivity Test:** To verify the runner sees Sonar, run:
  `docker exec -it <runner_container_id> ping sonarqube_app`
