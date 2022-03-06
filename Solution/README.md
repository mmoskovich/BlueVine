# Solution Diagram

![image](https://user-images.githubusercontent.com/100310547/156922450-01a6d543-dac2-4c17-b403-4c239e26e11e.png)



# ElasticSearch and Kibana Application hosted on Elastic Cloud

https://cloud.elastic.co/home

## Create Deployment
* Deployment Name - mmoskovich
* User: "elastic"
* Password: "GCEtdirUef9gqkrg6LvOon9e" (Usually stored in keepass or similar password manager software)
* cloud.id: "mmoskovich:dXMtY2VudHJhbDEuZ2NwLmNsb3VkLmVzLmlvJDg2YTNhMjIxMGRiMDQ3OTlhZDJkMGExYjNlMjMyM2EzJGQyZTM1MjRhN2NhMTRjM2JiZjJjNzc0MTU4MWYxZTgy"
* ElasticSearch EndPoint: https://mmoskovich.es.us-central1.gcp.cloud.es.io:9243
* Kibana EndPoint: https://mmoskovich.kb.us-central1.gcp.cloud.es.io:9243

In the backend:
* The filebeat container will forward log entries to the "ElasticSearch EndPoint" using Two Factor Authentication (2FA) - "User:Password" and "cloud.id" (over https)
* The nginx container will redirect '/kibana' requests to the "Kibana EndPoint"



# Backend Solution - Linux (Centos) VM host running Docker Containers




## Backend Web Application
### Backend Web Application - web-app.py (Python3)
The web application will be based on Python3 with the following modules:
* flask - A simple framework for building complex web applications
* ecs_logging - Logging formatters for ECS (Elastic Common Schema)

When accessing the root path ('/') on port TCP/8080, the web application will:
* Return 'Hello world!' message
* Store a log line (Elastic Common Schema format) in /var/log/web-app/web-app.log log file:
  * The log file is stored on the local disk of the Linux host itself - It is required for data consisty - to suppurt container restarts
  * The log file is used by Filebeat application to forward the messages to ElasticSearch


#### File: /root/Home-Assignment/web-app/web-app.py
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

### Backend Web Application - Docker Image and Container

#### Image installation
"python:3.8.2-alpine" will be used as the base image

A dockerfile will be used to create a new 'web-app' image with the following requirments:
* Add the web-app.py script and run it when the container will start
* Install flask and ecs_logging python modules
* Expose port TCP/8080 - will be used to get requests from the nginx

##### File: /root/Home-Assignment/web-app/dockerfile
```
FROM python:3.8.2-alpine
ADD web-app.py /
RUN pip install flask
RUN pip install flask_restful
RUN pip install ecs_logging
EXPOSE 8080
CMD [ "python", "web-app.py"]
```

##### Image Installation Commands
```
cd /root/Home-Assignment/web-app/

docker pull python:3.8.2-alpine

docker build . -t web-app
```
#### Container Installation and Runtime Commands
Run the commands below to:
* Create new network, named 'internal', for communicatiuons between the nginx and web-app applications (on port TCP/8080)
* Create a shared log directory (/var/log/web-app)
* Create the container for the first time and run it (detached mode):
  * Bind the new 'internal' network
  * Add the web application (web-app.py) under the root directory
  * Mount the log (/var/log/web-app) directory
  
```
mkdir -p /var/log/web-app/

docker network create internal

docker run \
  -d \
  --name=web-app \
  --expose 8080 \
  --network=internal \
  --volume="$(pwd)/web-app.py:/web-app.py:ro" \
  --mount type=bind,source=/var/log/web-app,target=/var/log/web-app \
  web-app
```


##### Sanity Tests
```
# From the Linux host:
docker images | grep web-app
docker ps | grep web-app
docker inspect web-app
```




## Proxy Application
### Proxy Application - nginx
nginx application will be used as proxy application

The nginx application will listen on port TCP/80 for incoming HTTP requests and will:
* Forward '/' requests to the web application on port TCP/8080 (HTTP)
* Redirect '/kibana' requests to the Kibana EP on Elastic cloud (HTTPS)

#### File: /root/Home-Assignment/nginx/nginx.conf
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



### Proxy Application - Docker Image and Container

#### Image installation
"nginx:alpine" will be used as the base image

```
docker pull nginx:alpine
```

#### Container Installation and Runtime Commands
Run the commands below to:
* Create the container for the first time and run it (detached mode):
  * Port mapping between the linux host and the applicaton on port TCP/80
  * bind the new 'internal' network
  * Replace the nginx configration file (/etc/nginx/nginx.conf) with the above nginx configuration file

```
cd /root/Home-Assignment/nginx/

docker run \
  -d \
  --name nginx \
  -p 80:80 \
  --network=internal \
  --volume="$(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro" \
  nginx:alpine
```


##### Sanity Tests
```
# From the Linux host:
docker images | grep nginx
docker ps | grep nginx
docker inspect nginx

# Open a browser and test the redirect setup to Kibana:
http://<LINUX_VM_IP>:80/kibana

# Open a browser and test the reverse proxy setup to the web application:
http://<LINUX_VM_IP>:80/

# Check the web applcaition log file:
cat /var/log/web-app/web-app.log

# Log file verification - Restart the web-app container and verify that the web-app.log file on the Linux host was not override
docker stop web-app
docker start web-app
docker ps | grep web-app

cat /var/log/web-app/web-app.log

```


##### ECS Format Log Examples (/var/log/web-app/web-app.log)
```
{"@timestamp":"2022-03-05T22:34:11.574Z","log.level":"info","message":"Someone just accessed '/' and got 'Hello world!' message","ecs":{"version":"1.6.0"},"log":{"logger":"werkzeug","origin":{"file":{"line":23,"name":"web-app.py"},"function":"index"},"original":"Someone just accessed '/' and got 'Hello world!' message"},"process":{"name":"MainProcess","pid":24456,"thread":{"id":139743951750912,"name":"Thread-1"}}}
{"@timestamp":"2022-03-05T22:34:42.417Z","log.level":"info","message":"Someone just accessed '/' and got 'Hello world!' message","ecs":{"version":"1.6.0"},"log":{"logger":"__main__","origin":{"file":{"line":23,"name":"web-app.py"},"function":"index"},"original":"Someone just accessed '/' and got 'Hello world!' message"},"process":{"name":"MainProcess","pid":24463,"thread":{"id":139721875691264,"name":"Thread-1"}}}
```




## Log Forwarder Application



### Log Forwarder Application - filebeat
filebeat application will be used to forward logs to ES (on cloud)

The filebeat application will:
* Monitor the log file (/var/log/web-app/web-app.log) for new entries
* Forward the logs to ES
* Update the registry file for the new transactions:
  * Based on the registry file, the filebeat "knows" what was the last log line that it handled
  * The registry file is stored on the local disk of the linux host under the registry directory (/usr/share/filebeat/data/registry) - on the next restart it will continue from the last processing point

#### File: /root/Home-Assignment/filebeat/filebeat.docker.yml
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


### Log Forwarder Application - Docker Image and Container

#### Image installation
"docker.elastic.co/beats/filebeat:8.0.1" will be used as the base image

```
docker pull docker.elastic.co/beats/filebeat:8.0.1
```

#### Container Installation and Runtime Commands
Run the commands below to:
* Create the consist registry directory (/usr/share/filebeat/data/registry) - required 
* Create the container for the first time and run it (detached mode) as use root:
  * Replace the /usr/share/filebeat/filebeat.yml configuration file with the above filebeat.docker.yml file
  * Mount the log (/var/log/web-app) and registry (/usr/share/filebeat/data/registry) directories

```
mkdir -p /usr/share/filebeat/data/registry

cd /root/Home-Assignment/filebeat

docker run \
  -d \
  -u root \
  --name=filebeat \
  --volume="$(pwd)/filebeat.docker.yml:/usr/share/filebeat/filebeat.yml:ro" \
  --volume="/var/lib/docker/containers:/var/lib/docker/containers:ro" \
  --volume="/var/run/docker.sock:/var/run/docker.sock:ro" \
  --mount type=bind,source=/var/log/web-app,target=/var/log/web-app \
  --mount type=bind,source=/usr/share/filebeat/data/registry,target=/usr/share/filebeat/data/registry \
  -e --strict.perms=false \
  docker.elastic.co/beats/filebeat:8.0.1 filebeat

```


##### Sanity Tests
```
# From the linux VM:
docker images | grep filebeat
docker ps | grep filebeat
docker inspect filebeat

# Open a browser and test the reverse proxy setup to the web application through the nginx:
http://<LINUX_VM_IP>:80/

# Look for new entries in the web applcaition log file:
cat /var/log/web-app/web-app.log

# Look for the new entries on the Kibana/ES UI

# Registry verification - Restart the filebeat container and confirm, on the Kibana/ES UI, that the filebeat didn't send the entire log lines again
docker stop filebeat
docker start filebeat
docker ps | grep filebeat

# From the Linux host, run the following commands to stress the system and compare the number of transactions with the Kibana UI:
i=1
repeat=10000
while true && [ $i -lt $repeat ] ; do 
  curl -s http://127.0.0.1:80/
  echo " #$i $(date -u +"%H:%M:00")"
  ((i=i+1))
done > transactions-details.txt

cat transactions-details.txt | awk '{print $NF}' | uniq -c

```
