
# Centos VM running on VirtualBox


# Backend Application
## Application Python3 / Flask / ecs_logging

Python3 using modules:
* flask - A simple framework for building complex web applications.
* ecs_logging - Logging formatters for ECS (Elastic Common Schema) in Python

When accessing the root path ('/') on port TCP/8080, the application will:
* Return 'Hello world!' message.
* Store a log line (Elastic Common Schema format) in /var/log/web-app/web-app.log file.
    * Notes:
        * The file is stored on the local path of the host itself
        * It is required for data consisty - to suppurt container restarts
        * The file is used by Filebeat application to forward the messages to ElasticSearch



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
## Docker Image and Container
### Image installation
#### File: /root/Home-Assignment/web-app/dockerfile
The dockerfile is required to create new image that is based on the "python:3.8.2-alpine" image:
* Add the web-app.py script and run it when the container start.
* Install flask and ecs_logging python modules.
* Expose port TCP/8080 - will be used to get requests from the nginx.
```
FROM python:3.8.2-alpine
ADD web-app.py /
RUN pip install flask
RUN pip install flask_restful
RUN pip install ecs_logging
EXPOSE 8080
CMD [ "python", "web-app.py"]
```
#### Commands
```
cd /root/Home-Assignment/web-app/
docker pull python:3.8.2-alpine
docker build . -t web-app
```
### Container Installations and Runtime
#### Container Environment Setup
The nginx and the web-app containers will comunicate on seperated network named 'internal'.
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
