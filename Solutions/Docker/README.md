
# Centos VM running on VirtualBox


# Backend Application
## Application Python3 / Flask / ecs_logging


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
    logger.info("Someone just accessed and got Hello World! :)")

    return 'Hello world!'

app.run(host='0.0.0.0', port=8080)

```
