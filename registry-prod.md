1. Install docker CS with instructions from :https://docs.docker.com/docker-trusted-registry/cs-engine/install/#install-on-ubuntu-14-04-lts
2. Setup up client verification to connect to it :
    1. Follow these steps to generate certs:
        https://github.nominum.com/nominum/kelpie/blob/master/doc/dockerhost.md
    2. open /etc/default/docker and replace DOCKER_OPTS:
       DOCKER_OPTS="--insecure-registry vish.ddns.nominum.com:445 -H unix:///var/run/docker.sock -H tcp://0.0.0.0:2377 --tlsverify  --tlscacert=/home/vish/ca-certs/ca-cert.pem --tlscert=/home/vish/ca-certs/server-cert.pem --tlskey=/home/vish/ca-certs/server-key.pem "
    3. Place the certificates in /root/ca-certs
3. Test if docker host and remote docker client can talk to each other
4. Install UCP:
    1. If you want to generate your own cert then ,create a volume to use own certificates:
        docker volume create ucp-controller-server-certs
    2. Place certificates under " /var/lib/docker/volumes/ucp-controller-server-certs/\_data/"
5. Start the UCP :
    docker run --rm -it --name ucp -v /var/run/docker.sock:/var/run/docker.sock docker/ucp:1.1.1 install -i --host-address vish.ddns.nominum.com --controller-port 444 --admin-password admin1  --fresh-install --debug
6. Check if UCP is up and running
7. Set up license in UCP and LDAP:
    ```
    LDAP SERVER URL: dc-04.win.nominum.com
    READER DN :CN=Docker,OU=Users-Sys,DC=WIN,DC=NOMINUM,DC=COM
    Reader Password : 3p{g47a{wrA4#k+Q
    DEFAULT PERMISSION FOR NEWLY DISCOVERED ACCOUNTS : No Access
    BASE DN :CN=Users,DC=WIN,DC=NOMINUM,DC=COM
    UserName Attribute: sAMAccountName
    FULL NAME ATTRIBUTE :cn
    Filter:(&(objectCategory=Person)(sAMAccountName=*))
    SEARCH SCOPE : One level
    Sync Configuration:1
    ```
8. Setup the load balancer:
    1. mkdir nginx
    2. cd nginx and create a conf file with the below contents:
    ```
    #user  nobody;
    #Defines which Linux system user will own and run the Nginx server

    worker_processes  15;
    #Referes to single threaded process. Generally set to be equal to the number of CPUs or cores.

    #error_log  logs/error.log; #error_log  logs/error.log  notice;
    #Specifies the file where server logs.

    #pid        logs/nginx.pid;
    #nginx will write its master process ID(PID).

    events {
        worker_connections  1024;
        # worker_processes and worker_connections allows you to calculate maxclients value:
        # max_clients = worker_processes * worker_connections
    }

    http {
        upstream myapp1 {
            least_conn;
            server vish.ddns.nominum.com:445;
            server vish.ddns.nominum.com:446;
            server vish.ddns.nominum.com:447;
        }
        server {
            listen 80;
            return 301 https://vish.ddns.nominum.com;
          }

        server {
            listen 443 ssl;
            server_name vish.ddns.nominum.com;
            ssl_certificate /etc/nginx/ssl/nginx.crt;
            ssl_certificate_key /etc/nginx/ssl/nginx.key;  
            location / {
                proxy_pass https://myapp1;
            }
        }
    }
    ```
    Note: Allow the load balancer to run

9. start the nginx-load balancer:
    docker run --name mynginx1 -v /home/vish/nginx/conf:/etc/nginx/nginx.conf:ro -p 443:443  -d nginx

10. Install DTR:
    1. Get the certificates used by UCP:
       curl -k https://$UCP_HOST/ca > ucp-ca.pem
    2. Launch up the docker registry:
    docker run -it --rm docker/dtr install --debug --ucp-url vish.ddns.nominum.com:444 --ucp-node vish.ddns.nominum.com --ucp-username admin --ucp-password admin1 --ucp-ca "$(cat ucp-ca.pem)" --dtr-external-url vish.ddns.nominum.com:443 --replica-http-port "8085" --replica-https-port "445"
    Note: --dtr-external-url should have the load balancer
11. Install DTR replica1:

    docker run -it --rm docker/dtr join --ucp-url vish.ddns.nominum.com  --ucp-node vish.ddns.nominum.com --existing-replica-id b0744db9792a --ucp-username admin --ucp-password admin1 --ucp-ca "$(cat ucp-ca.pem)" --replica-http-port "8086" --replica-https-port "446"

12. Install DTR replica2:

  docker run -it --rm docker/dtr join --ucp-url vish.ddns.nominum.com  --ucp-node vish.ddns.nominum.com --existing-replica-id b0744db9792a --ucp-username admin --ucp-password admin1 --ucp-ca "$(cat ucp-ca.pem)" --replica-http-port "8087" --replica-https-port "447"
