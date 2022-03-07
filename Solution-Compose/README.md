


## File: ./web-app/web-app.py
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

## File: ./web-app/Dockerfile
```
FROM python:3.8.2-alpine
ADD web-app.py /
RUN pip install flask
RUN pip install flask_restful
RUN pip install ecs_logging
CMD [ "python", "web-app.py"]
```





## File: ./nginx/nginx.conf

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

## File: ./filebeat/filebeat.docker.yml
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

## File ./docker-compose.yml
```
version: "3"
services:
  web-app:
    container_name: "web-app"
    image: web-app:latest
    build: ./web-app
    networks:
      - backend
    volumes:
      - ./web-app/web-app.py:/web-app.py:ro
      - /var/log/web-app/web-app.log:/var/log/web-app/web-app.log:rw

  nginx:
    container_name: "nginx"
    image: nginx:alpine
    depends_on:
      - web-app
    networks:
      - backend
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro

  filebeat:
    container_name: "filebeat"
    image: docker.elastic.co/beats/filebeat:8.0.1
    user: root
    networks:
      - backend
    volumes:
      - ./filebeat/filebeat.docker.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /var/log/web-app/web-app.log:/var/log/web-app/web-app.log:rw
      - /usr/share/filebeat/data/registry:/usr/share/filebeat/data/registry:rw
    command: ["--strict.perms=false"]

networks:
  backend:
    name: "custome_backend"
```
