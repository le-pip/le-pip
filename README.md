# le-pip
## ELK demo with syslog collection and a reverse proxy

## Network diagram
```
+------------------+ +------------------+
|      ELK NW      | |   Managment NW   |
|------------------| |------------------|
|      Elastic     | |                  |
|                  | |                  |
|  +-> Logstash    | |                  |
|  |               | |                  |
|  |   Kibana  <------+  Nginx      <--------+ gateway:5602
|  |               | ||                 |
|  |   Cerebro <------+             <--------+ gateway:9001
|  |               | |                  |
+--|---------------+ +------------------+
   |
   +-----------------------------------------+ all:1514
```

* Elastic is accessible via Logstash/Kibana/Cerebro only. No direct access.
* Logstash is listening for syslog at all hosts at port 1514
* Kibana and Cerebro are accessible at the gateway node via Nginix reverse proxy with basic authentication


## Assumptions
* *docker-ce* is installed at all target servers
* *sshd* is running at all the target servers
* Same user and password or ssh key is used to connect to all target servers

## Security limitations

* No host level security 
* No host level TLS
* No TLS between Docker swarm nodes
* No TLS between ELK components
* Ansible configured to silence SSH security warnings
* No data encryption at rest
* No audit
* Logstash unable to bind host's 127.0.0.1 in swarm mode which leaves it exposed at target nodes [[#32299](https://github.com/moby/moby/issues/32299)]

## ELK cluster configuration

Due to the limit of 4 Elastic nodes, 3 of them are set to be both 'data' and 'master' while the 4th node is set to be 'ingest' (becoming single point of failure, but with better restart policy).  

Kibana and Cerebro are set to communicate with Elastic cluster via the ingest node.

Elastic Index Lifecycle Management is enabled after the deployment. Seven days retention policy is set for syslog indices. Syslog template is updated accordingly. This eliminates the need in Curator cron tasks and allows easy API based management or retention, rolling and hot-warm policies.

Logstash grok filter parsing is out of scope.

## Deployment instructions

```bash
git clone <...>
cd le-pip/ansible
```

- Add target hosts to *inventory.ini*. Keep ```[managers]``` number odd (2n+1)
- Define your ssh user as ```ansible_user=``` in ```inventory.ini```
- If you have ssh keys spread across target hosts, define ```ssh_key_name=``` in ```inventory.ini``` as well and remove ```--ask-pass --ask-become-pass``` from the command bellow

```bash
ansible-playbook -v --ask-pass --ask-become-pass -i inventory.ini le_pip_ex_playbook.yml
```

Kibana and Cerebro  will be available at the first ```[managers]``` host that is used as gateway.

NGINX default credentials are set to ```admin:admin```. To change, please add following flags to ```ansible-playbook``` command above: ```--extra-vars "NGINX_USER=... NGINX_PASSWORD=..."``` or use Ansible Vault