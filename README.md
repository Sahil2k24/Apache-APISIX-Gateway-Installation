#### **My Journey with Apache APISIX: From Frustration to Success**

When I began my search for a reliable open-source API gateway, I narrowed my options to Kong and Apache APISIX. Initially, I decided to explore Kong, and while the experience was smooth, I discovered that its open-source version lacked certain critical features. This led me to Apache APISIX, which appeared promising in terms of functionality and flexibility. 

However, the installation process was unexpectedly challenging and frustrating. Following the official documentation, I tried to set it up using Docker. While most containers started successfully, the dashboard container, crucial for managing the gateway, failed to launch. The error consistently pointed to "etcd not connected." Determined to resolve this, I attempted various approaches—setting it up locally, trying it on an Ubuntu VM, and even on a Red Hat VM—but all yielded the same issue. 

I spent an entire day troubleshooting, combing through Stack Overflow, reading articles and blogs, and watching YouTube tutorials, yet the error persisted. The next day, with renewed effort, I explored more resources, but my attempts continued to be unsuccessful. I was on the verge of abandoning Apache APISIX and considering alternative API gateways when, at last, I stumbled upon a YouTube video. To my relief, it precisely addressed the issue and provided the solution I needed. 

Thanks to that video, I finally succeeded in setting up Apache APISIX and its dashboard. Reflecting on this journey, I realized that while the official documentation was comprehensive, it missed some vital details for configuring the dashboard. I now plan to document the process with clear and detailed instructions, referencing the official guide, to save others from encountering the same challenges.

---

#### **Installing Apache APISIX with Docker Compose**

Now that you have an idea of my journey, here’s how you can set up Apache APISIX along with the dashboard using Docker Compose:

---

### **Prerequisites**

Before you begin, ensure the following software is installed on your machine:

1. **Docker**  
   Docker is required to run containers. Install Docker by following the instructions for your operating system:

   - For **Linux**: [Install Docker on Linux](https://docs.docker.com/engine/install/)
   - For **Mac**: [Install Docker on macOS](https://docs.docker.com/desktop/install/mac-install/)
   - For **Windows**: [Install Docker on Windows](https://docs.docker.com/desktop/install/windows-install/)

   You can verify the installation by running:
   ```bash
   docker --version
   ```

2. **Docker Compose**  
   Docker Compose is required to manage multi-container Docker applications. To install Docker Compose:

   - For **Linux**: [Install Docker Compose on Linux](https://docs.docker.com/compose/install/)
   - For **Mac** and **Windows**: Docker Compose is included with Docker Desktop, so no additional installation is needed.

   Verify the installation by running:
   ```bash
   docker-compose --version
   ```

3. **curl**  
   curl is used to download the necessary files. Install curl if it's not already installed:

   - On **Linux** (Debian-based):
     ```bash
     sudo apt update
     sudo apt install curl
     ```
   - On **macOS**:
     ```bash
     brew install curl
     ```
   - On **Windows**, you can download the executable from [curl official website](https://curl.se/windows/).

   You can verify curl by running:
   ```bash
   curl --version
   ```

---

### **Step-by-Step Guide to Set Up Apache APISIX**

#### 1. **Clone the APISIX Docker Repository**

First, clone the APISIX Docker repository to your local machine:

```bash
git clone https://github.com/apache/apisix-docker.git
cd apisix-docker/example
```

#### 2. **Prepare the Configuration Files**

Create the necessary configuration files for Apache APISIX and the Dashboard:

- **`apisix_conf/config.yaml`**

Create a directory called `apisix_conf` and add the following `config.yaml`:

```yaml
apisix:
  node_listen: 9080              # APISIX listening port
  enable_ipv6: false

  enable_control: true
  control:
    ip: "0.0.0.0"
    port: 9092

deployment:
  admin:
    allow_admin:
      - 0.0.0.0/0              # We need to restrict ip access rules for security. 0.0.0.0/0 is for test.

    admin_key:
      - name: "admin"
        key: edd1c9f034335f136f87ad84b625c8f1
        role: admin
      - name: "viewer"
        key: 4054f7cf07e344346cd3f287985e76a2
        role: viewer

  etcd:
    host:
      - "http://etcd:2379"
    prefix: "/apisix"
    timeout: 30

plugin_attr:
  prometheus:
    export_addr:
      ip: "0.0.0.0"
      port: 9091
```

- **`dashboard_conf/conf.yaml`**

Create a directory called `dashboard_conf` and add the following `conf.yaml`:

```yaml
conf:
  listen:
    host: 0.0.0.0
    port: 9000
  allow_list:
    - 0.0.0.0/0
  etcd:
    endpoints:
      - "http://etcd:2379"
    mtls:
      key_file: ""
      cert_file: ""
      ca_file: ""
  log:
    error_log:
      level: warn
      file_path:
        logs/error.log
    access_log:
      file_path:
        logs/access.log
  security:
    content_security_policy: "default-src 'self'; script-src 'self' 'unsafe-eval' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; frame-src *"
authentication:
  secret: secret
  expire_time: 3600
  users:
    - username: admin
      password: admin
    - username: user
      password: user

plugins:
  - api-breaker
  - authz-keycloak
  - basic-auth
  - batch-requests
  - consumer-restriction
  - cors
  - echo
  - fault-injection
  - grpc-transcode
  - hmac-auth
  - http-logger
  - ip-restriction
  - jwt-auth
  - key-auth
  - limit-conn
  - limit-count
  - limit-req
  - prometheus
  - proxy-cache
  - proxy-rewrite
  - redirect
  - request-id
  - request-validation
  - response-rewrite
  - traffic-split
```

#### 3. **Edit the `docker-compose.yml`**

Modify the `docker-compose.yml` file to define all necessary services (APISIX, Dashboard, etcd, Prometheus, etc.). Here's the complete `docker-compose.yml`:

```yaml
version: "3"

services:
  apisix-dashboard:
    image: apache/apisix-dashboard:3.0.0-alpine
    restart: always
    volumes:
      - ./dashboard_conf/conf.yaml:/usr/local/apisix-dashboard/conf/conf.yaml
    ports:
      - "9000:9000"
    networks:
      apisix:

  apisix:
    image: apache/apisix:${APISIX_IMAGE_TAG:-3.1.0-debian}
    restart: always
    volumes:
      - ./apisix_conf/config.yaml:/usr/local/apisix/conf/config.yaml:ro
    depends_on:
      - etcd
    ports:
      - "9180:9180/tcp"
      - "9080:9080/tcp"
      - "9091:9091/tcp"
      - "9443:9443/tcp"
      - "9092:9092/tcp"
    networks:
      apisix:

  etcd:
    image: bitnami/etcd:3.4.15
    restart: always
    volumes:
      - etcd_data:/bitnami/etcd
    environment:
      ETCD_ENABLE_V2: "true"
      ALLOW_NONE_AUTHENTICATION: "yes"
      ETCD_ADVERTISE_CLIENT_URLS: "http://etcd:2379"
      ETCD_LISTEN_CLIENT_URLS: "http://0.0.0.0:2379"
    ports:
      - "2379:2379/tcp"
    networks:
      apisix:

  web1:
    image: nginx:1.19.0-alpine
    restart: always
    volumes:
      - ./upstream/web1.conf:/etc/nginx/nginx.conf
    ports:
      - "9081:80/tcp"
    networks:
      apisix:

  web2:
    image: nginx:1.19.0-alpine
    restart: always
    volumes:
      - ./upstream/web2.conf:/etc/nginx/nginx.conf
    ports:
      - "9082:80/tcp"
    networks:
      apisix:

  prometheus:
    image: prom/prometheus:v2.25.0
    restart: always
    volumes:
      - ./prometheus_conf/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090

"
    networks:
      apisix:

  grafana:
    image: grafana/grafana:7.3.1
    restart: always
    ports:
      - "3000:3000"
    networks:
      apisix:

networks:
  apisix:
    driver: bridge

volumes:
  etcd_data:
```

#### 4. **Start the Docker Compose Setup**

To start all the services:

```bash
docker-compose up -d
```

This will launch Apache APISIX, the Dashboard, and the required services in detached mode.

#### 5. **Verify the Running Containers**

Once the containers are up and running, you can verify them by using the following command:

```bash
docker ps
```

This command will list all running containers. You should see containers for `apisix`, `apisix-dashboard`, `etcd`, `prometheus`, and `grafana` running, along with their respective ports.

#### 6. **Access the Dashboard**

Once the containers are running, you can access the Apache APISIX dashboard by navigating to:

```
http://localhost:9000
```

Use the credentials you configured in the `conf.yaml` file (`username: admin`, `password: admin`) to log in.

---

### **Conclusion**

By following these steps, you should be able to set up Apache APISIX with Docker and run the dashboard successfully. The key issue I faced—“etcd not connected”—was resolved thanks to the insights from the YouTube tutorial, and I hope this guide helps you avoid the frustration I encountered.
