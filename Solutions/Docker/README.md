
# ElasticSearch and Kibana in cloud
https://cloud.elastic.co/home

## Create Deployment
* Deployment Name - mmoskovich
* User:Password - "elastic:GCEtdirUef9gqkrg6LvOon9e" (Usually stored in keepass or similar password manager software)
* cloud.id - "mmoskovich:dXMtY2VudHJhbDEuZ2NwLmNsb3VkLmVzLmlvJDg2YTNhMjIxMGRiMDQ3OTlhZDJkMGExYjNlMjMyM2EzJGQyZTM1MjRhN2NhMTRjM2JiZjJjNzc0MTU4MWYxZTgy"
* ElasticSearch EndPoint - https://mmoskovich.es.us-central1.gcp.cloud.es.io:9243
* Kibana EndPoint - https://mmoskovich.kb.us-central1.gcp.cloud.es.io:9243

In the backend:
* The filebeat container will forward log entries to the "ElasticSearch EndPoint" using Two Factor Authentication (2FA) - "User:Password" and "cloud.id" (over https).
* The nginx container will redirect '/kibana' requests to the "Kibana EndPoint"


# Backend Solution - Centos VM running Docker
## Backend Application - web-app.py (Python3 / Flask / ecs_logging)

The application will use Python3 with the following modules:
* flask - A simple framework for building complex web applications
* ecs_logging - Logging formatters for ECS (Elastic Common Schema) in Python

When accessing the root path ('/') on port TCP/8080, the application will:
* Return 'Hello world!' message
* Store a log line (Elastic Common Schema format) in /var/log/web-app/web-app.log file
    * Notes:
        * The log file is stored on the local disk of the host itself - It is required for data consisty - to suppurt container restarts
        * The log file is used by Filebeat application to forward the messages to ElasticSearch


### File: /root/Home-Assignment/web-app/web-app.py
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
## Backend Application - Docker Image and Container

### Image installation
"python:3.8.2-alpine" is used as the base image

A dockerfile will be used to create a new 'web-app' image with the following requirments:
* Add the web-app.py script and run it when the container will start
* Install flask and ecs_logging python modules
* Expose port TCP/8080 - will be used to get requests from the nginx

#### File: /root/Home-Assignment/web-app/dockerfile
```
FROM python:3.8.2-alpine
ADD web-app.py /
RUN pip install flask
RUN pip install flask_restful
RUN pip install ecs_logging
EXPOSE 8080
CMD [ "python", "web-app.py"]
```
#### Image Installation Commands
```
cd /root/Home-Assignment/web-app/
docker pull python:3.8.2-alpine
docker build . -t web-app
```
### Container Installation and Runtime
#### Container Installation Commands
Run the commands below to:
* Create the shared log directory
* Create dedicated network for communicatiuons between the web-app and nginx applications on port TCP/8080.

```
mkdir -p /var/log/web-app/
docker network create internal
```
#### Container Runtime Command
```
docker run -d \
  --name=web-app \
  --network=internal \
  --volume="$(pwd)/web-app.py:/web-app.py:ro" \
  --mount type=bind,source=/var/log/web-app,target=/var/log/web-app \
  web-app
```
