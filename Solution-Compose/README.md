# Solution Diagram

![image](https://user-images.githubusercontent.com/100310547/157263394-54cfa6e6-52a2-4343-a651-015d0b79db43.png)

# ElasticSearch and Kibana Application hosted on Elastic Cloud

https://cloud.elastic.co/home

## Create Deployment
* Deployment Name: mmoskovich
* User: "elastic"
* Password: "GCEtdirUef9gqkrg6LvOon9e" (Usually stored in keepass or similar password manager software)
* cloud.id: "mmoskovich:dXMtY2VudHJhbDEuZ2NwLmNsb3VkLmVzLmlvJDg2YTNhMjIxMGRiMDQ3OTlhZDJkMGExYjNlMjMyM2EzJGQyZTM1MjRhN2NhMTRjM2JiZjJjNzc0MTU4MWYxZTgy"
* ElasticSearch EndPoint: https://mmoskovich.es.us-central1.gcp.cloud.es.io:9243
* Kibana EndPoint: https://mmoskovich.kb.us-central1.gcp.cloud.es.io:9243

In the backend:
* The filebeat container will forward log entries to the "ElasticSearch EndPoint" using Two Factor Authentication (2FA) - "User:Password" and "cloud.id" (over https)
* The nginx container will redirect '/kibana' requests to the "Kibana EndPoint"

## Index and Dashboard

### Index and Data View
The backend (filebeat component) will upload data to the ElasticSearch which will store it on ".filebeat..." index

The ".filebeat..." index will be used to create the "filebeat-\*" Data View

### "Hello world!" Dashbaord
The Dashboard will be created based on the "filebeat-\*" Data View

The dashboard will have 2 charts:
* Vertical Bar Chart - based on timestamp
* Count Chart

![image](https://user-images.githubusercontent.com/100310547/157265690-be839efe-9dee-4ab8-ac27-535da2216a9c.png)




# Backend Solution - Linux (Centos) hosted on AWS EC2

## Backend Web Application

### Backend Web Application - web-app.py (Python3)

The web application will be based on Python3 with the following modules:
* flask - A simple framework for building complex web applications
* ecs_logging - Logging formatters for ECS (Elastic Common Schema)

When accessing the root path ('/') on port TCP/8080, the web application will:
* Return 'Hello world!' message
* Store a log line in Elastic Common Schema (ECS) format in /var/log/web-app/web-app.log log file:
  * The log file is stored on the local disk of the Linux host itself - It is required for data consisty - to suppurt container restarts
  * The log file is monitored by the filebeat application to forward messages to ElasticSearch


#### File: ./web-app/web-app.py
```
#!/usr/bin/env python3

from flask import Flask
import logging
import ecs_logging
import time

handler = logging.FileHandler('/var/log/web-app/web-app.log')
handler.setFormatter(ecs_logging.StdlibFormatter())

logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)
logger.addHandler(handler)

app = Flask(__name__)

@app.route('/')
def index():
    logger.info("Someone just accessed '/' and got 'Hello world!' message")

    return 'Hello world!'

app.run(host='0.0.0.0', port=8080)
```

### Backend Web Application - Docker Image

"python:3.8.2-alpine" will be used as the base image

Dockerfile will be used to create a new 'web-app' image with the following requirments:
* Add the web-app.py script and run it when the container will start
* Install flask and ecs_logging python modules

#### File: ./web-app/Dockerfile
```
FROM python:3.8.2-alpine
ADD web-app.py /
RUN pip install flask
RUN pip install flask_restful
RUN pip install ecs_logging
CMD [ "python", "web-app.py"]
```



## Proxy Application

### Proxy Application - nginx

nginx application will be used as proxy application

The nginx application will listen on port TCP/80 for incoming HTTP requests and will:
* Forward '/' requests to the web application on port TCP/8080 (HTTP)
* Redirect '/kibana' requests to the Kibana EP on Elastic cloud (HTTPS)

#### File: ./nginx/nginx.conf

```
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    access_log off;
    error_log off;

    server {
        listen 80;

        location = / {
            proxy_pass http://web-app:8080;
        }

        location = /kibana {
            return 301 https://mmoskovich.kb.us-central1.gcp.cloud.es.io:9243;
        }
    }

    sendfile        on;
    #tcp_nopush     on;
    keepalive_timeout  5;
    #gzip  on;
}
```

## Log Forwarder Application

### Log Forwarder Application - filebeat

filebeat application will be used to forward logs to ElasticSearch

The filebeat application will:
* Monitor the log file (/var/log/web-app/web-app.log) for new entries
* Forward the logs to ElasticSearch
* Update the registry file for the new transactions:
  * Based on the registry file, the filebeat "knows" what was the last log line that it handled
  * The registry file is stored on the local disk of the linux host under the registry directory (/usr/share/filebeat/data/registry) - It is required for data consisty - to suppurt container restarts and avoid from sending the same data twice

#### File: ./filebeat/filebeat.docker.yml
```
filebeat.config:
  modules:
    path: ${path.config}/modules.d/*.yml
    reload.enabled: false

processors:
- add_cloud_metadata: ~

output.elasticsearch:
  hosts: ['https://mmoskovich.es.us-central1.gcp.cloud.es.io:9243']

cloud:
  id: "mmoskovich:dXMtY2VudHJhbDEuZ2NwLmNsb3VkLmVzLmlvJDg2YTNhMjIxMGRiMDQ3OTlhZDJkMGExYjNlMjMyM2EzJGQyZTM1MjRhN2NhMTRjM2JiZjJjNzc0MTU4MWYxZTgy"
  auth: "elastic:GCEtdirUef9gqkrg6LvOon9e"

filebeat.inputs:
- type: log
  enabled: true
  paths:
    - "/var/log/web-app/web-app.log"
```

## Solution Deployment

### Solution Deployment - docker-compose
The various componsnts will be installed using the docker-compose utility

#### File ./docker-compose.yml
```
version: "3"
services:
  web-app:
    container_name: "web-app"
    image: web-app:latest
    build: ./web-app
    networks:
      - internal
    volumes:
      - ./web-app/web-app.py:/web-app.py:ro
      - /var/log/web-app:/var/log/web-app:rw

  nginx:
    container_name: "nginx"
    image: nginx:alpine
    depends_on:
      - web-app
    networks:
      - internal
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro

  filebeat:
    container_name: "filebeat"
    image: docker.elastic.co/beats/filebeat:8.0.1
    user: root
    networks:
      - internal
    volumes:
      - ./filebeat/filebeat.docker.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /var/log/web-app:/var/log/web-app
      - /usr/share/filebeat/data/registry:/usr/share/filebeat/data/registry:rw
    command: ["--strict.perms=false"]

networks:
  internal:
    name: "internal"
```


### Solution Deployment - Installation
#### Prerequisites
* AWS VPC:
  * Security Group blocking all incoming traffic except of SSH (TCP/22) and HTTP (TCP/80)
  * EC2 instance up and running with basic Centos7 version, configured with public IP and private key
  * Notes - The common practice is not to expose the EC2 server with a public IP - it should be behind NLB, etc
* docker-ce 20.10.X
* docker-compose 3.5+ (1.29.X)
* Connect to the EC2 instance and copy the attached solution-compose.tar file to any directory under the server (for example, ~/HA/)


```
# Docker installation
###########################################
sudo systemctl stop docker

sudo yum remove docker docker-client docker-client-latest docker-common docker-latest
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum-config-manager --enable docker-ce-nightly
sudo yum install docker-ce docker-ce-cli containerd.io

sudo docker --version

sudo systemctl start docker
sudo systemctl status docker


# Docker-Compose installation
###########################################
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

sudo rm -rf /usr/bin/docker-compose
sudo rm -rf /bin/docker-compose

sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

sudo docker-compose --version

# Create working directory
###########################################
mkdir ~/HA
cd ~/HA
cp /tmp/solution-compose.tar .
tar -xvf solution-compose.tar
```

#### Installation
Run the following commands:
```
# Files:
# ./web-app/web-app.py
# ./web-app/Dockerfile
# ./nginx/nginx.conf
# ./filebeat/filebeat.docker.yml
# ./docker-compose.yml

cd ~/HA

sudo docker-compose up -d
```

#### Verifications

##### Images and Containers
On the Linux host, verify that the images exist and the containers are up and running:
```
sudo docker images

sudo docker ps
```

##### Nginx connectivity
Open a browser and test the reverse proxy setup to the web application through the nginx
('Hello world!' message should appear)
```
http://<LINUX_HOST_IP>:80/
```

Open another browser and test the redirect setup to the Kibana application through the nginx
```
http://<LINUX_HOST_IP>:80/kibana
```
Look for the new entries on the Kibana/ES UI


##### Restart Containers
Check that the filebeat application doesn't send the entire logs upon restart
Confirm using the Kibana UI
```
sudo docker stop filebeat

sudo docker start filebeat

sudo docker ps | grep filebeat
```

##### E2E Tests
From the Linux host, run the following commands and confirm the results using the Kibana UI:
```
i=1
repeat=10000
while true && [ $i -lt $repeat ] ; do 
  curl -s http://127.0.0.1:80/
  echo " #$i $(date -u +"%H:%M:00")"
  ((i=i+1))
done > transactions-details.txt

cat transactions-details.txt | awk '{print $NF}' | uniq -c
```

##### Verification tool
Verification tool (python) can be provided if needed:
* Send GET requests to nginx ('/')
* Send GET requests to ES to get the number of requests that arrived
* Compare between the results
