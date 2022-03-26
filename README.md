## Postal Mailserver for OpenShift

I was working on a simple mailserver to support various apps internally to my lab for notifcations - without setting up a full featured server / client. After looking at a few, I chose Postal and figured I'd share my configuration with everyone else trying to get it to work on Kubernetes/ OpenShift (OKD). I only enabled in-bound smtp/25 and the web interface, but everything is here if you'd like to expand that. 

#### The Install
1. Create your namespace/project \
`oc new-project postal`

2. MariaDB 
    ##### Note: I'm using 'thin' vmware storage type for storage persistence
    - First, create a secret to store your username and password \
    `oc create secret generic mariadb-secret -n postal --from-literal=username=${USERNAME} --from-literal=password=${PASSWORD}` \
    - Deploy the application and service \
    `oc create -f mariadb/deployment.yaml -f mariadb/service.yaml`

3. RabbitMWQ
    - First, create a secret to store your username and password \
    `oc create secret generic rabbitmq-secret -n postal --from-literal=username=${USERNAME} --from-literal=password=${PASSWORD}`
    - Deploy the application and service \
    `oc create -f rabbitmq/deployment.yaml -f rabbitmq/service.yaml` 

4. While all of that is loading, now's a good time to configure and stage your postal.yml.
   There are a couple of ways that I've done this, as a configmap and as a file on a nfs share. During my troubleshooting, I stuck with the nfs share so I could keep editing my postal.yml file easier.  However, it'd be better stored as a configmap, so I do encourage you doing that once your system is up and running the way you like.

     - A copy of the generic file can be found at postal/postal.yml
     - You'll need to edit this with your settings, for me, I just had to focus on the connections to MariaDB/RabbitMQ and ensure my MX record was set. When postal ran, the DNS Record test failed, but routing mail was successful. I was able to verify everyting using 'dig'.
     - For DNS I have 1 MX record for domain redcloud.land pointing to mail.redcloud.land (internal / private network)
     - In order to access SMTP, I added a record in my haproxy to forward port 25 to nodePort 32025 (you'll see this in the postal/deployment.yaml)
        ```
        listen postal-smtp-25            
        bind *:25              
        mode tcp                   
        balance source            
        server worker0 worker0.redcloud.land:32025 check inter 1s
        server worker1 worker1.redcloud.land:32025 check inter 1s
        server worker2 worker2.redcloud.land:32025 check inter 1s
        ```
5. Postal
   - Create the PV/PVC \
   `oc create -f postal/nfs-volume.yaml`
   - Deploy the initilize job to configure the database and build your signing.key for RabbitMQ \
   `oc create -f postal/initilize/deploy-job.yaml`
    - Deploy everything else \
    `oc create -f postal/`

6. Troubleshoot... likely

7. When all the pods are up, you'll need to exec into the web pod and create your user so you can log into the portal.
    - First, create a route to access the portal - I've chosen to use the default *.apps for access \
    `oc create route edge webui --service=webui --port=5000`
    - Find the pod and exec into it... \
    `oc get pod | grep web`
    ```
    shaker@local: postal-mailserver\>  oc get pod | grep web
    web-596599887d-2p624        1/1     Running   0             14h
    ```
    - Create you user to log into the web portal.. follow the prompts. \
    `oc exec -it ${POD} -- bash` \
    `postal make-user`
    ```
    shaker@local: postal-mailserver\>  oc exec -it web-596599887d-2p624 -- bash
    1000750000@web-596599887d-2p624:/opt/postal/app$ postal make-user
    ```
Now you should be able to access and log into the web portal. For the rest of the setup, please use the official docs (link below).
Again, I'd like to point out that the DNS tests did not work, but I could verify the records were accurate using 'dig mx redcloud.land'. When sending test messages, they were successful. Once Postal was up and running, I ended up not liking it much and may switch to a simple postfix relay to my gmail.com account later on. Stay tuned if you're interested in that.

>>> Docs: https://docs.postalserver.io/



