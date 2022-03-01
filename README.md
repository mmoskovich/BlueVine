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
- nginx -> Kibana (http://Docker_IP/kibana/)
- nginx -> backend (http://Docker_IP/)
- backend -> Elasticsearch for logs
- Kibana -> Elasticsearch to present the logs

  
# Setup Files
- nginx image creation
- backend application image creation
- Elasticsearch image creation
- Kibana image creation
- Installation and launch commands
  - Docker
  - nginx
  - backend web application
  - Elasticsearch
  - Kibana
