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
1. nginx image creation
2. backend application image creation
3. Elasticsearch image creation
4. Kibana image creation
5. Installation and launch commands
- Docker
- nginx
- backend web application
- Elasticsearch
- Kibana
