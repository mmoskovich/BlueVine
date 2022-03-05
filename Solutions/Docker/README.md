
# Centos VM running on VirtualBox


# Backend Application
## Application Python3 / Flask / ecs_logging

Python3 using modules:
* flask - A simple framework for building complex web applications.
* ecs_logging - Logging formatters for ECS (Elastic Common Schema) in Python


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
