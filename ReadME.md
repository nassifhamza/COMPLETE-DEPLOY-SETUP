# Complete DevOps Environment Setup Guide

This document provides a comprehensive guide to setting up a complete DevOps environment using Docker Compose. It covers the configuration of various services, including Jenkins, SonarQube, Nexus, NGINX, PostgreSQL, and MongoDB, along with detailed instructions on how to run and configure them.

## 1. Prerequisites

Before you begin, ensure you have the following installed on your system:

*   **Docker Desktop:** For Windows and macOS, Docker Desktop provides the Docker Engine, Docker CLI client, Docker Compose, and Kubernetes. For Linux, you can install Docker Engine and Docker Compose separately.
    *   [Install Docker Desktop](https://www.docker.com/products/docker-desktop)
*   **Git:** For cloning the example project and managing your configuration files.
    *   [Install Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
*   **Text Editor:** A code editor like VS Code, Sublime Text, or Notepad++ for editing configuration files.

## 2. Project Structure

It is recommended to organize your project files in a structured manner. Create a main directory (e.g., `devops-environment`) and place your `docker-compose.yml` and `nginx` configuration directory within it:

```
devops-environment/
├── docker-compose.yml
└── nginx/
    ├── nginx.conf
    ├── conf.d/
    │   ├── jenkins.conf
    │   ├── sonarqube.conf
    │   └── nexus.conf
    └── certs/
        ├── nginx.crt
        └── nginx.key
```

## 3. `docker-compose.yml` Explained

The `docker-compose.yml` file defines all the services, networks, and volumes required for your DevOps environment. Each service is configured with its image, ports, volumes, environment variables, and dependencies.

```yaml
version: '3.8'

services:
  jenkins:
    container_name: jenkins
    image: jenkins/jenkins:lts
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock # Mount Docker socket for Jenkins to build/run Docker images
    environment:
      - JAVA_OPTS=-Xmx1024m -Xms512m
    networks:
      - devops-network
    # Explanation:
    # - container_name: Assigns a fixed name to the container for easy identification.
    # - image: Specifies the Jenkins LTS (Long Term Support) image.
    # - ports: Maps host ports to container ports. 8080 for Jenkins UI, 50000 for Jenkins agents.
    # - volumes: Persists Jenkins data to a named volume (jenkins_home) and mounts the Docker socket
    #            from the host to allow Jenkins to execute Docker commands (e.g., build images).
    # - environment: Sets Java options for Jenkins, controlling memory allocation.
    # - networks: Connects Jenkins to the custom devops-network.

  sonarqube:
    container_name: sonarqube
    image: sonarqube:lts-community
    ports:
      - "9000:9000"
    environment:
      - SONARQUBE_JDBC_URL=jdbc:postgresql://sonarqube_db:5432/sonarqube
      - SONARQUBE_JDBC_USERNAME=sonar
      - SONARQUBE_JDBC_PASSWORD=sonar
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
    networks:
      - devops-network
    depends_on:
      - sonarqube_db
    # Explanation:
    # - container_name: Fixed name for the SonarQube container.
    # - image: Uses the SonarQube LTS Community Edition image.
    # - ports: Maps host port 9000 to container port 9000 for SonarQube UI.
    # - environment: Configures SonarQube to use the PostgreSQL database (sonarqube_db) with specified credentials.
    # - volumes: Persists SonarQube data, extensions, and logs to named volumes.
    # - networks: Connects SonarQube to the custom devops-network.
    # - depends_on: Ensures sonarqube_db starts before SonarQube.

  sonarqube_db:
    container_name: sonarqube_db
    image: postgres:13
    environment:
      - POSTGRES_DB=sonarqube
      - POSTGRES_USER=sonar
      - POSTGRES_PASSWORD=sonar
    volumes:
      - sonarqube_db_data:/var/lib/postgresql/data
    networks:
      - devops-network
    # Explanation:
    # - container_name: Fixed name for the SonarQube database container.
    # - image: Uses PostgreSQL version 13.
    # - environment: Sets the database name, user, and password for SonarQube's database.
    # - volumes: Persists the database data to a named volume.
    # - networks: Connects the database to the custom devops-network.

  nexus:
    container_name: nexus
    image: sonatype/nexus3
    ports:
      - "8081:8081"
    volumes:
      - nexus_data:/nexus-data
    networks:
      - devops-network
    # Explanation:
    # - container_name: Fixed name for the Nexus container.
    # - image: Uses the Sonatype Nexus Repository Manager 3 image.
    # - ports: Maps host port 8081 to container port 8081 for Nexus UI.
    # - volumes: Persists Nexus data to a named volume.
    # - networks: Connects Nexus to the custom devops-network.

  nginx:
    container_name: nginx
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    networks:
      - devops-network
    depends_on:
      - jenkins
      - sonarqube
      - nexus
    # Explanation:
    # - container_name: Fixed name for the NGINX container.
    # - image: Uses the latest NGINX image.
    # - ports: Maps host ports 80 (HTTP) and 443 (HTTPS) to container ports for NGINX to serve web traffic.
    # - volumes: Mounts NGINX configuration files and SSL certificates from the host to the container as read-only.
    # - networks: Connects NGINX to the custom devops-network.
    # - depends_on: Ensures Jenkins, SonarQube, and Nexus start before NGINX, as NGINX proxies them.

  postgresql:
    container_name: postgresql
    image: postgres:13
    environment:
      - POSTGRES_DB=mydatabase
      - POSTGRES_USER=myuser
      - POSTGRES_PASSWORD=mypassword
    volumes:
      - postgresql_data:/var/lib/postgresql/data
    networks:
      - devops-network
    # Explanation:
    # - container_name: Fixed name for the general PostgreSQL database container.
    # - image: Uses PostgreSQL version 13.
    # - environment: Sets the database name, user, and password for your application's database.
    # - volumes: Persists the database data to a named volume.
    # - networks: Connects the database to the custom devops-network.

  mongodb:
    container_name: mongodb
    image: mongo:latest
    ports:
      - "27017:27017"
    environment:
      - MONGO_INITDB_ROOT_USERNAME=mongoadmin
      - MONGO_INITDB_ROOT_PASSWORD=mongopassword
    volumes:
      - mongodb_data:/data/db
    networks:
      - devops-network
    # Explanation:
    # - container_name: Fixed name for the MongoDB database container.
    # - image: Uses the latest MongoDB image.
    # - ports: Maps host port 27017 to container port 27017 for MongoDB access.
    # - environment: Sets the root username and password for MongoDB initialization.
    # - volumes: Persists the database data to a named volume.
    # - networks: Connects the database to the custom devops-network.

volumes:
  jenkins_home:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:
  sonarqube_db_data:
  nexus_data:
  postgresql_data:
  mongodb_data:
  # Explanation:
  # - Defines named volumes for persistent storage of data for each service. This ensures that data
  #   is not lost when containers are stopped, removed, or updated.

networks:
  devops-network:
    driver: bridge
    # Explanation:
    # - Defines a custom bridge network named 'devops-network'. All services are connected to this
    #   network, allowing them to communicate with each other using their service names as hostnames.
    #   This isolates the DevOps environment from other Docker networks on your host.
```

## 4. NGINX Configuration

NGINX acts as a reverse proxy and load balancer for your services, allowing you to access them via custom domain names (e.g., `jenkins.devops.local`) and providing SSL/TLS encryption. You need to create an `nginx` directory with the following structure:

```
devops-environment/
├── docker-compose.yml
└── nginx/
    ├── nginx.conf
    ├── conf.d/
    │   ├── jenkins.conf
    │   ├── sonarqube.conf
    │   └── nexus.conf
    └── certs/
        ├── nginx.crt
        └── nginx.key
```

### 4.1. Generate SSL Certificates

To enable HTTPS access for your services, you need self-signed SSL certificates. These are suitable for local development. Run the following command in your `devops-environment` directory to generate them:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout nginx/certs/nginx.key -out nginx/certs/nginx.crt -subj "/CN=devops.local"
```
*   **Explanation:** This command generates a self-signed SSL certificate (`nginx.crt`) and its private key (`nginx.key`) valid for 365 days. The `CN=devops.local` sets the Common Name, which is important for your browser to recognize the certificate for your `*.devops.local` domains.

### 4.2. `nginx.conf`

This is the main NGINX configuration file. Create it at `nginx/nginx.conf`:

```nginx
user nginx;
worker_processes auto;

error_log /var/log/nginx/error.log notice;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main 
              '$remote_addr - $remote_user [$time_local] "$request" '
              '$status $body_bytes_sent "$http_referer" '
              '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    include /etc/nginx/conf.d/*.conf;
}
```
*   **Explanation:** This file sets up basic NGINX worker processes, logging, and includes all `.conf` files from the `conf.d` directory, which will contain your service-specific configurations.

### 4.3. Service-Specific NGINX Configurations

Create individual configuration files for each service in the `nginx/conf.d/` directory. These files define how NGINX proxies requests to each Docker service.

#### `nginx/conf.d/jenkins.conf`

```nginx
server {
    listen 80;
    listen 443 ssl;
    server_name jenkins.devops.local;

    ssl_certificate /etc/nginx/certs/nginx.crt;
    ssl_certificate_key /etc/nginx/certs/nginx.key;

    location / {
        proxy_pass http://jenkins:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 600;
        proxy_send_timeout 600;
        proxy_read_timeout 600;
        send_timeout 600;
    }
}
```
*   **Explanation:** This block configures NGINX to listen on ports 80 (HTTP) and 443 (HTTPS) for requests to `jenkins.devops.local`. It uses the generated SSL certificates. `proxy_pass http://jenkins:8080;` forwards requests to the Jenkins container (which is accessible via its service name `jenkins` and internal port `8080` within the Docker network). The `proxy_set_header` directives ensure that Jenkins receives the correct host, client IP, and protocol information.

#### `nginx/conf.d/sonarqube.conf`

```nginx
server {
    listen 80;
    listen 443 ssl;
    server_name sonarqube.devops.local;

    ssl_certificate /etc/nginx/certs/nginx.crt;
    ssl_certificate_key /etc/nginx/certs/nginx.key;

    location / {
        proxy_pass http://sonarqube:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 600;
        proxy_send_timeout 600;
        proxy_read_timeout 600;
        send_timeout 600;
    }
}
```
*   **Explanation:** Similar to Jenkins, this configures NGINX to proxy requests for `sonarqube.devops.local` to the SonarQube container on internal port `9000`.

#### `nginx/conf.d/nexus.conf`

```nginx
server {
    listen 80;
    listen 443 ssl;
    server_name nexus.devops.local;

    ssl_certificate /etc/nginx/certs/nginx.crt;
    ssl_certificate_key /etc/nginx/certs/nginx.key;

    location / {
        proxy_pass http://nexus:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 600;
        proxy_send_timeout 600;
        proxy_read_timeout 600;
        send_timeout 600;
    }
}
```
*   **Explanation:** This configures NGINX to proxy requests for `nexus.devops.local` to the Nexus container on internal port `8081`.

## 5. How to Run the Environment

Once all your `docker-compose.yml` and NGINX configuration files are in place, you can start your entire DevOps environment with a single command.

1.  **Navigate to your `devops-environment` directory** (the one containing `docker-compose.yml` and the `nginx` folder) in your terminal.
2.  **Run the following command:**
    ```bash
docker compose up -d --build
    ```
*   **Explanation:**
    *   `docker compose up`: This command reads your `docker-compose.yml` file and starts all the services defined within it.
    *   `-d`: (detached mode) Runs the containers in the background, so your terminal remains free.
    *   `--build`: Forces Docker Compose to rebuild images if there are changes in the `Dockerfile`s (though in this setup, we are using pre-built images, it's good practice for local development if you were to add custom Dockerfiles).

This command will pull the necessary Docker images (if not already present), create the containers, set up the `devops-network`, and start all services in the correct order based on `depends_on` directives.

## 6. Initial Configuration and Access for Each Service

This section provides detailed steps for accessing and performing initial setup for each service.

### 6.1. Jenkins

*   **Purpose:** Jenkins is an open-source automation server that enables developers to reliably build, test, and deploy their software.
*   **Access:** Open your web browser and navigate to `https://jenkins.devops.local`.
    *   **Note on SSL:** Your browser will likely show a warning about the self-signed SSL certificate. You can safely bypass this for local development by proceeding to the website.
*   **Initial Admin Password:** When Jenkins starts for the first time, it generates an initial administrator password. To retrieve it, run the following command in your terminal:
    ```bash
    docker logs jenkins | grep "initialAdminPassword"
    ```
    Copy the alphanumeric password displayed and paste it into the Jenkins setup wizard in your browser.
*   **Install Plugins:** During the setup, choose "Install suggested plugins." This will install a common set of plugins necessary for most CI/CD tasks.
*   **Create Admin User:** Follow the prompts to create your first admin user (username and password). Remember these credentials for future logins.
*   **Requirements to Run:** Jenkins requires Java to run. The `jenkins/jenkins:lts` Docker image comes with Java pre-installed. It also needs persistent storage for its configuration, jobs, and build history, which is handled by the `jenkins_home` volume.

### 6.2. SonarQube

*   **Purpose:** SonarQube is an open-source platform for continuous inspection of code quality to perform static code analysis, identify bugs, code smells, and security vulnerabilities.
*   **Access:** Open your web browser and navigate to `https://sonarqube.devops.local`.
*   **Default Credentials:** The default login is `admin` with password `admin`. You will be prompted to change this upon first login. For direct use in pipelines, you can keep these defaults for initial setup, but it is highly recommended to change them in a production environment.
*   **SonarQube Admin Token (for Jenkins Integration):** After initial login, it's best practice to generate a user token for the `admin` user to use for Jenkins integration. Go to `My Account` > `Security` > `Generate Tokens`, create a new token, and copy its value (e.g., `sqp_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0`). This token will be used in Jenkins credentials.
*   **Requirements to Run:** SonarQube requires a database (PostgreSQL in this setup) to store its analysis results and configuration. It also needs persistent storage for its data, extensions, and logs, handled by `sonarqube_data`, `sonarqube_extensions`, and `sonarqube_logs` volumes.

### 6.3. Nexus Repository

*   **Purpose:** Nexus Repository Manager is a universal artifact repository that supports various formats (Maven, npm, Docker, NuGet, etc.). It's used to store and manage build artifacts, dependencies, and Docker images.
*   **Access:** Open your web browser and navigate to `https://nexus.devops.local`.
*   **Default Credentials:** The default login is `admin` with password `admin123`. You will be prompted to change this upon first login. For direct use in pipelines, you can keep these defaults for initial setup, but it is highly recommended to change them in a production environment.
*   **Requirements to Run:** Nexus requires persistent storage for its repositories and configuration, handled by the `nexus_data` volume.

### 6.4. PostgreSQL Database

*   **Purpose:** PostgreSQL is a powerful, open-source relational database system. In this setup, it serves as the backend database for SonarQube and can be used by your application (e.g., the FastAPI example project).
*   **Access:** You can connect to the PostgreSQL database from other containers within the `devops-network` using the hostname `postgresql` and port `5432`. From your host machine, you can connect using `localhost:5432` (if you map the port in `docker-compose.yml`, which is not explicitly done for this service as it's primarily for internal use).
*   **Credentials:**
    *   Database: `mydatabase`
    *   User: `myuser`
    *   Password: `mypassword`
*   **Management:** You can use a PostgreSQL client (e.g., `psql` from the command line, or a GUI tool like DBeaver or pgAdmin) to connect and manage databases, users, and permissions. For example, to connect from your host using `psql` (if installed):
    ```bash
    psql -h localhost -p 5432 -U myuser -d mydatabase
    ```
*   **Requirements to Run:** PostgreSQL requires persistent storage for its data, handled by the `postgresql_data` volume.

### 6.5. MongoDB Database

*   **Purpose:** MongoDB is a popular NoSQL document database. It can be used by applications that require a flexible, scalable, and high-performance database.
*   **Access:** You can connect to the MongoDB database from other containers within the `devops-network` using the hostname `mongodb` and port `27017`. From your host machine, you can connect using `localhost:27017`.
*   **Credentials:**
    *   Username: `mongoadmin`
    *   Password: `mongopassword`
*   **Management:** You can use a MongoDB client (e.g., `mongosh` from the command line, or a GUI tool like MongoDB Compass) to connect and manage collections and documents. For example, to connect from your host using `mongosh` (if installed):
    ```bash
    mongosh "mongodb://mongoadmin:mongopassword@localhost:27017/admin"
    ```
*   **Requirements to Run:** MongoDB requires persistent storage for its data, handled by the `mongodb_data` volume.




## 7. Detailed Jenkins Pipeline Tutorial

This section guides you through setting up a basic CI/CD pipeline in Jenkins for the `fastapi/full-stack-fastapi-template` project. This pipeline will demonstrate fetching code, building, running tests, performing SonarQube analysis, and pushing artifacts to Nexus.

### 7.1. Access Jenkins and Install Plugins

1.  **Access Jenkins:** Ensure your Jenkins container is running and accessible via `https://jenkins.devops.local`.
2.  **Initial Setup:** If this is your first time, complete the initial setup by providing the admin password (found via `docker logs jenkins | grep "initialAdminPassword"`) and installing suggested plugins.
3.  **Install Additional Plugins:** Go to `Manage Jenkins` > `Manage Plugins` > `Available plugins`. Search for and install the following plugins if they are not already installed:
    *   **Git** (usually pre-installed)
    *   **Pipeline** (usually pre-installed)
    *   **Docker** (for building Docker images, if needed, though Docker socket mount handles much of this)
    *   **SonarQube Scanner** (for SonarQube integration)
    *   **Nexus Artifact Uploader** (for Nexus integration)

    After selecting, click `Install without restart` or `Download now and install after restart`.

### 7.2. Create a New Jenkins Pipeline Job

1.  From the Jenkins dashboard, click on `New Item`.
2.  Enter an item name (e.g., `fastapi-devops-pipeline`).
3.  Select `Pipeline` and click `OK`.

### 7.3. Configure the Pipeline Job

On the configuration page:

1.  **General:** (Optional) Add a description for your pipeline.
2.  **Build Triggers:** For now, you can leave this as default. Later, you might configure `GitHub hook trigger for GITScm polling` or `Poll SCM` for automatic builds on code changes.
3.  **Pipeline:**
    *   Select `Pipeline script from SCM`.
    *   **SCM:** Choose `Git`.
    *   **Repository URL:** Enter `https://github.com/fastapi/full-stack-fastapi-template.git`.
    *   **Credentials:** Leave as `- none -` for public repositories. If you fork the repository and make it private, you'll need to add Jenkins credentials (e.g., `Username with password` for GitHub).
    *   **Branches to build:** Leave as `*/master` or change to `*/main` if the default branch is `main`.
    *   **Script Path:** Enter `Jenkinsfile`. This tells Jenkins to look for a file named `Jenkinsfile` in the root of your repository for the pipeline definition.

4.  Click `Save`.



### 7.4. Next Steps for Integration (Jenkins Credentials and Tools)

To make the SonarQube Analysis and Nexus Push stages work, you need to configure credentials and tools within Jenkins:

*   **SonarQube Integration:**
    1.  **Configure SonarQube Server:** In Jenkins, go to `Manage Jenkins` > `Configure System`. Scroll down to `SonarQube servers` and click `Add SonarQube`. Provide a `Name` (e.g., `My SonarQube Server`) and the `Server URL` (`http://sonarqube:9000`).
    2.  **Generate SonarQube Token:** Log into your SonarQube instance (`https://sonarqube.devops.local`). Go to `My Account` > `Security` > `Generate Tokens`. Create a new token and copy its value.
    3.  **Add SonarQube Credential in Jenkins:** In Jenkins, go to `Manage Jenkins` > `Manage Credentials` > `System` > `Global credentials (unrestricted)` > `Add Credentials`. Select `Secret text` as the `Kind`. Paste the SonarQube token into the `Secret` field. Set the `ID` to `sonarqube-token` (this *must* match `SONAR_TOKEN_ID` in the `Jenkinsfile`). Add a descriptive `Description`.
    4.  **Configure SonarQube Scanner Tool:** In Jenkins, go to `Manage Jenkins` > `Global Tool Configuration`. Scroll down to `SonarQube Scanners`. Click `Add SonarQube Scanner`. Set the `Name` to `SonarQube` (this *must* match `installationName` in the `Jenkinsfile`). Check `Install automatically` and select the latest `Version`.

*   **Nexus Integration:**
    1.  **Create Nexus Repositories:** Log into your Nexus instance (`https://nexus.devops.local`). Create a new `maven2 (hosted)` repository for your artifacts and a `docker (hosted)` repository for Docker images if you plan to push them.
    2.  **Create Nexus User (Optional but Recommended):** Create a dedicated user in Nexus with appropriate deployment permissions instead of using `admin` for pipeline pushes.
    3.  **Add Nexus Credential in Jenkins:** In Jenkins, go to `Manage Jenkins` > `Manage Credentials` > `System` > `Global credentials (unrestricted)` > `Add Credentials`. Select `Username with password` as the `Kind`. Enter the Nexus username (e.g., `admin` or your dedicated user) and password. Set the `ID` to `nexus-user` (this *must* match `NEXUS_CREDENTIAL_ID` in the `Jenkinsfile`). Add a descriptive `Description`.

## 8. Troubleshooting Common Issues

This section addresses common problems you might encounter during setup and operation.

*   **Container not starting:**
    *   **Check Docker Logs:** The first step is always to check the logs of the failing container. Run `docker logs <container_name>` (e.g., `docker logs jenkins`) to see error messages.
    *   **Port Conflicts:** Ensure no other application on your host machine is using the ports mapped in `docker-compose.yml` (e.g., 80, 443, 8080, 9000, 8081, 5432, 27017). Use `netstat -ano | findstr :<port>` (Windows) or `sudo lsof -i :<port>` (Linux/macOS) to identify conflicting processes.
    *   **Resource Limits:** Ensure Docker Desktop or your Docker daemon has enough allocated resources (CPU, memory) to run all services.

*   **Service not accessible via `*.devops.local` URLs:**
    *   **`hosts` File Configuration:** This is the most common issue. You *must* add entries to your host machine's `hosts` file to map the custom domains to your localhost IP address. The `hosts` file location varies by OS:
        *   **Windows:** `C:\Windows\System32\drivers\etc\hosts`
        *   **Linux/macOS:** `/etc/hosts`
        Add the following lines:
        ```
        127.0.0.1 jenkins.devops.local
        127.0.0.1 sonarqube.devops.local
        127.0.0.1 nexus.devops.local
        ```
    *   **Flush DNS Cache:** After modifying the `hosts` file, flush your DNS cache to ensure changes take effect immediately:
        *   **Windows:** `ipconfig /flushdns` (in Admin Command Prompt)
        *   **macOS:** `sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder` (in Terminal)
        *   **Linux:** `sudo systemctl restart NetworkManager` or `sudo /etc/init.d/nscd restart` (commands vary by distribution)
    *   **NGINX Running:** Ensure the `nginx` container is running and its logs show no errors.
    *   **XAMPP/Other Web Servers:** Confirm that XAMPP Apache or any other web server is completely stopped and not using ports 80 or 443.

*   **Jenkins `docker.sock` Mounting Error (`not a directory: unknown`):**
    *   This error indicates a problem with Docker Desktop's setup on Windows, specifically how it exposes the Docker daemon socket to WSL2 or the host. The `docker-compose.yml` already includes the correct volume mount (`- /var/run/docker.sock:/var/run/docker.sock`).
    *   **Ensure Docker Desktop is running and healthy.**
    *   **Reset Docker Desktop to factory defaults.** This is often the most effective solution for persistent Docker Desktop issues. Go to `Settings` > `Troubleshoot` > `Reset to factory defaults`. (This will remove all your local Docker images, containers, and volumes).
    *   **Verify WSL2 is correctly set up and integrated with Docker Desktop.** Ensure Windows features are enabled, the kernel update is installed, and Docker Desktop's WSL integration is enabled for your distributions.
    *   **Run Docker Compose from Windows Terminal:** Always run `docker compose up -d --build` from a standard Windows Command Prompt or PowerShell, not directly from within a WSL2 terminal if your project files are on the Windows filesystem.

*   **Jenkins `ERROR: Could not find or load main class org.sonar.runner.Main`:**
    *   This means the SonarQube Scanner is not correctly configured in Jenkins.
    *   **Install SonarQube Scanner Plugin:** In Jenkins, go to `Manage Jenkins` > `Manage Plugins` > `Available plugins` and install `SonarQube Scanner`.
    *   **Configure SonarQube Scanner Tool:** In Jenkins, go to `Manage Jenkins` > `Global Tool Configuration` > `SonarQube Scanners`. Add a new scanner with `Name: SonarQube` (matching `installationName` in `Jenkinsfile`) and check `Install automatically`.
    *   **Configure SonarQube Server:** In Jenkins, go to `Manage Jenkins` > `Configure System` > `SonarQube servers`. Add your SonarQube instance URL (`http://sonarqube:9000`) and link a `Secret text` credential (e.g., `sonarqube-token`) containing a SonarQube user token.

*   **Jenkins `ERROR: nexus-password` or similar credential errors:**
    *   This indicates a mismatch between the credential ID used in the `Jenkinsfile` and the ID configured in Jenkins.
    *   **Verify Credential ID:** In Jenkins, go to `Manage Jenkins` > `Manage Credentials` > `System` > `Global credentials (unrestricted)`. Ensure you have a `Username with password` credential with `ID: nexus-user` (or whatever `NEXUS_CREDENTIAL_ID` is set to in your `Jenkinsfile`) and the correct username/password for Nexus.
    *   **Update `Jenkinsfile`:** Ensure the `Jenkinsfile` uses the correct credential ID and the `withCredentials` block as shown in the example `Jenkinsfile` in section 7.4.

This comprehensive troubleshooting guide should help you identify and resolve most issues you might encounter. Always refer to the official documentation of each tool for more in-depth information.




## 9. Default Credentials for Services

For your convenience, here are the default credentials for the services in this DevOps environment. **It is highly recommended to change these default credentials in a production environment for security reasons.**

*   **Jenkins:**
    *   Initial Admin Password: Retrieved via `docker logs jenkins | grep "initialAdminPassword"`
    *   After initial setup, you will create your own admin user.

*   **SonarQube:**
    *   Username: `admin`
    *   Password: `admin`

*   **Nexus Repository:**
    *   Username: `admin`
    *   Password: `admin123`

*   **PostgreSQL Database:**
    *   Database Name: `mydatabase`
    *   User: `myuser`
    *   Password: `mypassword`

*   **MongoDB Database:**
    *   Root Username: `mongoadmin`
    *   Root Password: `mongopassword`

These credentials are used in the `docker-compose.yml` and are referenced in the `Jenkinsfile` example for integration purposes.



