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
5.1 Docker
5.2 nginx
5.3 backend web application
5.4 Elasticsearch
5.5 Kibana
