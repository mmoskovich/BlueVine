# BlueVine - Home assignment

# Assigment
![Assigment](Assigment.JPG)


# Solution
The Solution that I choosed is docker based.

# Diagram

# Components
docker
  - container - nginx
  - container - backend web application
  - container - Elasticsearch
  - container - Kibana

# Flows
- nginx -> Kibana (http://<Docker IP>/kibana/)
- nginx -> backend (http://<Docker IP>/)
- backend -> Elasticsearch for logs
- Kibana -> Elasticsearch to present the logs
