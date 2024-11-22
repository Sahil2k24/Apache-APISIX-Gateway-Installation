# **My Journey with Apache APISIX: From Frustration to Success**

When I began my search for a reliable open-source API gateway, I narrowed my options to Kong and Apache APISIX. Initially, I decided to explore Kong, and while the experience was smooth, I discovered that its open-source version lacked certain critical features. This led me to Apache APISIX, which appeared promising in terms of functionality and flexibility. 

However, the installation process was unexpectedly challenging and frustrating. Following the official documentation, I tried to set it up using Docker. While most containers started successfully, the dashboard container, crucial for managing the gateway, failed to launch. The error consistently pointed to "etcd not connected." Determined to resolve this, I attempted various approaches—setting it up locally, trying it on an Ubuntu VM, and even on a Red Hat VM—but all yielded the same issue. 

I spent an entire day troubleshooting, combing through Stack Overflow, reading articles and blogs, and watching YouTube tutorials, yet the error persisted. The next day, with renewed effort, I explored more resources, but my attempts continued to be unsuccessful. I was on the verge of abandoning Apache APISIX and considering alternative API gateways when, at last, I stumbled upon a YouTube video. To my relief, it precisely addressed the issue and provided the solution I needed. 

Thanks to that video, I finally succeeded in setting up Apache APISIX and its dashboard. Reflecting on this journey, I realized that while the official documentation was comprehensive, it missed some vital details for configuring the dashboard. I now plan to document the process with clear and detailed instructions, referencing the official guide, to save others from encountering the same challenges.

---

## **Installing Apache APISIX with Docker Compose**

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
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

apisix:
  node_listen: 9080              # APISIX listening port
  enable_ipv6: false

  enable_control: true
  control:
    ip: "0.0.0.0"
    port: 9092

deployment:
  admin:
    allow_admin:               # https://nginx.org/en/docs/http/ngx_http_access_module.html#allow
      - 0.0.0.0/0              # We need to restrict ip access rules for security. 0.0.0.0/0 is for test.

    admin_key:
      - name: "admin"
        key: edd1c9f034335f136f87ad84b625c8f1
        role: admin                 # admin: manage all configuration data

      - name: "viewer"
        key: 4054f7cf07e344346cd3f287985e76a2
        role: viewer

  etcd:
    host:                           # it's possible to define multiple etcd hosts addresses of the same etcd cluster.
      - "http://etcd:2379"          # multiple etcd address
    prefix: "/apisix"               # apisix configurations prefix
    timeout: 30                     # 30 seconds

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
    host: 0.0.0.0     # `manager api` listening ip or host name
    port: 9000          # `manager api` listening port
  allow_list:           # If we don't set any IP list, then any IP access is allowed by default.
    - 0.0.0.0/0
  etcd:
    endpoints:          # supports defining multiple etcd host addresses for an etcd cluster
      - "http://etcd:2379"
                          # yamllint disable rule:comments-indentation
                          # etcd basic auth info
    # username: "root"    # ignore etcd username if not enable etcd auth
    # password: "123456"  # ignore etcd password if not enable etcd auth
    mtls:
      key_file: ""          # Path of your self-signed client side key
      cert_file: ""         # Path of your self-signed client side cert
      ca_file: ""           # Path of your self-signed ca cert, the CA is used to sign callers' certificates
    # prefix: /apisix     # apisix config's prefix in etcd, /apisix by default
  log:
    error_log:
      level: warn       # supports levels, lower to higher: debug, info, warn, error, panic, fatal
      file_path:
        logs/error.log  # supports relative path, absolute path, standard output
                        # such as: logs/error.log, /tmp/logs/error.log, /dev/stdout, /dev/stderr
    access_log:
      file_path:
        logs/access.log  # supports relative path, absolute path, standard output
                         # such as: logs/access.log, /tmp/logs/access.log, /dev/stdout, /dev/stderr
                         # log example: 2020-12-09T16:38:09.039+0800	INFO	filter/logging.go:46	/apisix/admin/routes/r1	{"status": 401, "host": "127.0.0.1:9000", "query": "asdfsafd=adf&a=a", "requestId": "3d50ecb8-758c-46d1-af5b-cd9d1c820156", "latency": 0, "remoteIP": "127.0.0.1", "method": "PUT", "errs": []}
  security:
      # access_control_allow_origin: "http://httpbin.org"
      # access_control_allow_credentials: true          # support using custom cors configration
      # access_control_allow_headers: "Authorization"
      # access_control-allow_methods: "*"
      # x_frame_options: "deny"
      content_security_policy: "default-src 'self'; script-src 'self' 'unsafe-eval' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; frame-src *"  # You can set frame-src to provide content for your grafana panel.

authentication:
  secret:
    secret              # secret for jwt token generation.
                        # NOTE: Highly recommended to modify this value to protect `manager api`.
                        # if it's default value, when `manager api` start, it will generate a random string to replace it.
  expire_time: 3600     # jwt token expire time, in second
  users:                # yamllint enable rule:comments-indentation
    - username: admin   # username and password for login `manager api`
      password: admin
    - username: user
      password: user

plugins:                          # plugin list (sorted in alphabetical order)
  - api-breaker
  - authz-keycloak
  - basic-auth
  - batch-requests
  - consumer-restriction
  - cors
  # - dubbo-proxy
  - echo
  # - error-log-logger
  # - example-plugin
  - fault-injection
  - grpc-transcode
  - hmac-auth
  - http-logger
  - ip-restriction
  - jwt-auth
  - kafka-logger
  - key-auth
  - limit-conn
  - limit-count
  - limit-req
  # - log-rotate
  # - node-status
  - openid-connect
  - prometheus
  - proxy-cache
  - proxy-mirror
  - proxy-rewrite
  - redirect
  - referer-restriction
  - request-id
  - request-validation
  - response-rewrite
  - serverless-post-function
  - serverless-pre-function
  # - skywalking
  - sls-logger
  - syslog
  - tcp-logger
  - udp-logger
  - uri-blocker
  - wolf-rbac
  - zipkin
  - server-info
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
    ##network_mode: host
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
    environment:
      - NGINX_PORT=80
    networks:
      apisix:

  web2:
    image: nginx:1.19.0-alpine
    restart: always
    volumes:
      - ./upstream/web2.conf:/etc/nginx/nginx.conf
    ports:
      - "9082:80/tcp"
    environment:
      - NGINX_PORT=80
    networks:
      apisix:

  prometheus:
    image: prom/prometheus:v2.25.0
    restart: always
    volumes:
      - ./prometheus_conf/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    networks:
      apisix:

  grafana:
    image: grafana/grafana:7.3.7
    restart: always
    ports:
      - "3000:3000"
    volumes:
      - "./grafana_conf/provisioning:/etc/grafana/provisioning"
      - "./grafana_conf/dashboards:/var/lib/grafana/dashboards"
      - "./grafana_conf/config/grafana.ini:/etc/grafana/grafana.ini"
    networks:
      apisix:


networks:
  apisix:
    driver: bridge

volumes:
  etcd_data:
    driver: local

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

<img width="960" alt="Dashboard" src="https://github.com/user-attachments/assets/cb7b408a-6266-4e1f-a881-47c305020002">
