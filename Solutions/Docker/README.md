
# Centos VM running on VirtualBox


# Backend Application
## Application Python3 / Flask / ecs_logging

Python3 using modules:
* flask - A simple framework for building complex web applications.
* ecs_logging - Logging formatters for ECS (Elastic Common Schema) in Python
When accessing the root path ('/') on port 8080, the application will:
* Return 'Hello world!' message.
* Store a message in Elastic Common Schema (ECS) format in /var/log/web-app/web-app.log file.
** Notes:
*** The file is stored on the local path of the host itself.
*** It is required for data consisty and suppurt container restarts



### File: /root/Home-Assignment/web-app/web-app.py
```
#!/usr/bin/env python3

from flask import Flask
import logging
import ecs_logging
import time


handler = logging.FileHandler('/var/log/web-app/web-app.log')
handler.setFormatter(ecs_logging.StdlibFormatter())

logger = logging.getLogger('werkzeug')
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
#### dockerfile
The dockerfile is required to create new image that is based on the "python:3.8.2-alpine" image.
It will add the python script, install flask and ecs_logging modules and expose port 8080 to access the 

```
docker pull python:3.8.2-alpine


```

### Container Installation
