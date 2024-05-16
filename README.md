# SECURING-PROMETHEUS-API-AND-UI-ENDPOINTS-USING-BASIC-AUTH

## Login to the target machine and follow the following steps:
- First we need to create certificate on the target to be used in authentication 
- Create working directory :
  
        mkdir /etc/node_exporter
        cd /etc/node_exporter

- Generate self-signed certificates using open SSL:

        sudo openssl req -x509 -newkey rsa:4096 -nodes -keyout node_exporter.key -out node_exporter.crt
  
The previous command creates node_exporter.crt and node_exporter.key files

- Create a config.yml file (could be named anything) in the same directory and add the following:

        tls_server_config:
          cert_file: node_exporter.crt
          key_file: node_exporter.key
  this file will be added later to node_exporter.service unit  file to obtain tls encryption
- Check user name in service file
  
          cat /etc/systemd/system/node_exporter.service
you should find  user in [service] section

                [Service]
                User=node_exporter
                Group=node_exporter
                Type=simple
for me I use node_exporter user
- Update the permission to the /etc/node_exporter for the user used in unit file:

        sudo chown -R node_exporter:node_exporter /etc/node_exporter

- Update the systemd service of node_exporter with TLS config as follows:

        [Unit]
        Description=Node Exporter
        Wants=network-online.target
        After=network-online.target
        [Service]
        User=node_exporter
        Group=node_exporter
        Type=simple
        ExecStart=/usr/local/bin/node_exporter --web.config.file="/etc/node_exporter/config.yml"
        [Install]
        WantedBy=multi-user.target

- Reload systemd manager and restart node_exporter service

        sudo systemctl daemon-reload
        sudo systemctl restart node_exporter

- Test the metrics on the local machine :
  
        curl https://localhost:<port>/metrics
the output will be :

        #curl: (60) SSL certificate problem: self-signed certificate
        More details here: https://curl.se/docs/sslcerts.html
        
        curl failed to verify the legitimacy of the server and therefore could not
        establish a secure connection to it. To learn more about this situation and
        how to fix it, please visit the web page mentioned above.
That is because we are using self signed certificate.
to solve this use -k option

        curl -k https://localhost:<port>/metrics

Currently we have enabled encryption on the node exporter.

The next step is to update promethues server configuration 
but before copy node_exporter.crt file from the target to the server:

         scp /etc/node_exporter/node_exporter.crt user@<prometheus-server-ip>:/etc/prometheus/
## On Prometheus server 

-  Update /etc/prometheus/prometheus at specific target as follows :

          - job_name: 'node'
            scheme: https
            tls_config:
              ca_file: /etc/prometheus/node_exporter.crt
              insecure_skip_verify: true
            static_configs:
              - targets: ["<target_IP_Address>:<Target_Port>"]

At this point the traffic between service and target is encrypted and metrics can be accessed using node_exporter self signed certificate:

## Authentication

To provide authentication between the  server and the target do the following:

## On target server:

- We have to create a hash of the password we will use to authenticate, you can use any hashing tool, anyway i used openssl to hash the password "pass":

        $ openssl passwd -6
        Password: <write password>
        Verifying - Password:  <write password>
- Update /etc/node_exporter/config.yml as follows:

        tls_server_config:
          cert_file: node_exporter.crt
          key_file: node_exporter.key
        
        basic_auth_users:
            <username>: <hashed_password>
- Restart node exporter service:

        systemctl restart node_exporter

## On Prometheus server:

- Update the configuration file with basic_auth section to use that credentials for authentication:

        sudo vi /etc/prometheus/prometheus.yml
add the following to a specific target 

        - job_name: 'node'
          scheme: https
          basic_auth:
            username: <username from config.yml file>
            password: <plain text passwoed from config.yml file>
          tls_config:
            ca_file: /etc/prometheus/node_exporter.crt
            insecure_skip_verify: true
          static_configs:
            - targets: ["<target_IP_Address>:<Target_Port>"]
- Restart prometheus

        sudo systemctl restart prometheus

- Then lets test authentication :

if we try :

        curl -k https://<target_IP_Address>:<Target_Port>/metrics

- The output will be :

        Unauthorized

to be authorized you need to use -u option:

        curl -u <username> -k https://<target_IP_Address>:<Target_Port>/metrics
then you need to enter the password 
        
                
                    
