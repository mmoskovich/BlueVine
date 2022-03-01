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
- nginx -> Kibana (http://Docker IP/kibana/)
- nginx -> backend (http://Docker IP/)
- backend -> Elasticsearch for logs
- Kibana -> Elasticsearch to present the logs

  
# Setup Files
1. Docker installation
2. nginx image creation and installation
3. backend application image creation and installation
4. Elasticsearch image creation and installation
5. Kibana image creation and installation
6. Run commands
