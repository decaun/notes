## Cloud shell provides the following:

```
Temporary Compute Engine VM
Command-line access to the instance via a browser
5 GB of persistent disk storage ($HOME dir)
Pre-installed Cloud SDK and other tools
gcloud: for working with Google Compute Engine and many GCP services
gsutil: for working with Cloud Storage
kubectl: for working with Google Container Engine and Kubernetes
bq: for working with BigQuery
Language support for Java, Go, Python, Node.js, PHP, and Ruby
Web preview functionality
Built-in authorization for access to resources and instances
```

# --Storage commands --

```
gsutil mb gs://<BUCKET_NAME> > creates new bucket
gsutil cp [MY_FILE] gs://[BUCKET_NAME] > copy files from shell to bucket
gcloud compute regions list

source infraclass/config > set env variables inside file (...INFRACLASS_REGION=us-central1)

edit .profile (not .bash_profile) to set scripts at shell startup
```

# --Network commands --

```
gcloud compute networks create $NETWORK_NAME --subnet-mode=custom
gcloud compute networks subnets create $SUBNET_NAME --network=privatenet --region=us-central1 --range=172.16.0.0/24

gcloud compute networks list
gcloud compute networks subnets list --sort-by=NETWORK

With CIDR make sure to include the /0 in the Source IP ranges to specify all networks.

gcloud compute firewall-rules create $RULE_NAME --direction=INGRESS --priority=1000 --network=$NETWORK_NAME --action=ALLOW --rules=icmp,tcp:22,tcp:3389 --source-ranges=0.0.0.0/0

gcloud compute firewall-rules list --sort-by=NETWORK
```

# --Compute Engine commands --

```
gcloud compute instances create $INSTANCE_NAME --zone=us-central1-c --machine-type=f1-micro --subnet=$NETWORK_NAME

gcloud compute instances create $MY_VMNAME \
--machine-type "n1-standard-1" \
--image-project "debian-cloud" \
--image-family "debian-9" \
--subnet "default"

gcloud compute instances list --sort-by=ZONE


free > shows memory
sudo dmidecode -t 17  > shows memory details
nproc > number of cpus
lscpu > shows cpu details

yum -y install screen > extra console at background
```
# --Cloud Storage commands --
```
gsutil acl get gs://$BUCKET_NAME_1/setup.html  > acl.txt
cat acl.txt
```
- make priv.
```
gsutil acl set private gs://$BUCKET_NAME_1/setup.html
```
- make public read
```
gsutil acl ch -u AllUsers:R gs://$BUCKET_NAME_1/setup.html
```
The encryption controls are contained in a gsutil configuration file named .boto

- generate .boto 
```
gsutil config -n

gsutil cp setup2.html gs://$BUCKET_NAME_1/
gsutil rewrite -k gs://$BUCKET_NAME_1/setup2.html

gsutil lifecycle get gs://$BUCKET_NAME_1
gsutil lifecycle set life.json gs://$BUCKET_NAME_1

gsutil versioning get gs://$BUCKET_NAME_1
gsutil versioning set on gs://$BUCKET_NAME_1
```
## Copy with versioning
```
gsutil cp -v setup.html gs://$BUCKET_NAME_1

gsutil rsync -r ./firstlevel gs://$BUCKET_NAME_1/firstlevel

gcloud auth activate-service-account --key-file credentials.json
```
## SQL
 
- Proxy script
```
wget https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64 -
O cloud_sql_proxy && chmod +x cloud_sql_proxy
```
- Enable proxy on VM
```
./cloud_sql_proxy -instances=$INSANCE_NAME=tcp:3306 &
```
# --Deployment commands --
```
gcloud app deploy app.yaml
gcloud app browse
```
- redeploy
```
gcloud app deploy app.yaml --quiet
```
# Stress test
```
ab -n 500000 -c 1000 http://$LB_IP/
```
# --Deployment Manager Commands -- 
```
gcloud deployment-manager types list | grep network
gcloud deployment-manager types list | grep firewall
gcloud deployment-manager types list | grep instance
```
## example config.YAML

```
imports:
- path: instance-template.jinja
resources:
# Create the auto-mode network
- name: mynetwork
  type: compute.v1.network
  properties:
    autoCreateSubnetworks: true

# Create the firewall rule
- name: mynetwork-allow-http-ssh-rdp-icmp
  type: compute.v1.firewall
  properties:
    network: $(ref.mynetwork.selfLink)
    sourceRanges: ["0.0.0.0/0"]
    allowed:
    - IPProtocol: TCP
      ports: [22, 80, 3389]
    - IPProtocol: ICMP

# Create the mynet-us-vm instance
- name: mynet-us-vm
  type: instance-template.jinja
  properties:
    zone: us-central1-a
    machineType: n1-standard-1
    network: $(ref.mynetwork.selfLink)
    subnetwork: regions/us-central1/subnetworks/mynetwork

# Create the mynet-eu-vm instance
- name: mynet-eu-vm
  type: instance-template.jinja
  properties:
    zone: europe-west1-d
    machineType: n1-standard-1
    network: $(ref.mynetwork.selfLink)  
    subnetwork: regions/europe-west1/subnetworks/mynetwork
```


## example instance-template.jinja

```
resources:
- name: {{ env["name"] }}
  type: compute.v1.instance  
  properties:
     machineType: zones/{{ properties["zone"] }}/machineTypes/{{ properties["machineType"] }}
     zone: {{ properties["zone"] }}
     networkInterfaces:
      - network: {{ properties["network"] }}
        subnetwork: {{ properties["subnetwork"] }}
        accessConfigs:
        - name: External NAT
          type: ONE_TO_ONE_NAT
     disks:
      - deviceName: {{ env["name"] }}
        type: PERSISTENT
        boot: true
        autoDelete: true
        initializeParams:
          sourceImage: https://www.googleapis.com/compute/v1/projects/debian-cloud/global/images/family/debian-9
```



- Test depl. man.
```
gcloud deployment-manager deployments create $DEPLOYMENT_NAME --config=config.yaml --preview
```
- Execute depl.
```
gcloud deployment-manager deployments update $DEPLOYMENT_NAME
```
- Delete deployment
```
gcloud deployment-manager deployments delete $DEPLOYMENT_NAME
```

> Alternative for deployment-manager could be Terraform

https://cloud.google.com/deployment-manager/docs/configuration/supported-resource-types

https://cloud.google.com/compute/docs/reference/latest/instances

- convert proto to json
```
echo $1
sed -i 's/string/"string"/g' $1
sed -i 's/boolean/"boolean"/g' $1
sed -i 's/integer/"integer"/g' $1
sed -i 's/unsigned long/"unsigned long"/g' $1
sed -i 's/bytes/"bytes"/g' $1
sed -i 's/(key)/"(key)"/g' $1
sed -i 's/float/"float"/g' $1
sed -i 's/etag/"etag"/g' $1
sed -i 's/long,/"long",/g' $1
```
https://www.json2yaml.com/

- URI for compute
```
gcloud compute zones list
```
- compute unit availability
```
gcloud compute machine-types list | grep [YOUR_ZONE]
```
- URI for network
```
gcloud compute networks list
gcloud compute networks describe default
```
- URI for image
```
gcloud compute images list --uri | grep debian
```
## example
```
resources:
- name: appserver
  type: compute.v1.instance
  properties:
    zone: us-west1-b
    machineType: https://www.googleapis.com/compute/v1/projects/architectingdp/zones/us-west1-b/machineTypes/f1-micro
    networkInterfaces:
    - network: https://www.googleapis.com/compute/v1/projects/architectingdp/global/networks/default
      accessConfigs:
      - name: External_NAT
        type: ONE_TO_ONE_NAT
    disks:
    - type: PERSISTENT
      deviceName: boot
      boot: true
      autoDelete: true
      initializeParams:
        sourceImage: https://www.googleapis.com/compute/v1/projects/debian-cloud/global/images/debian-9-stretch-v20170816
```

- Deploy
```
gcloud deployment-manager deployments create appserver --config appserver.yaml

gcloud deployment-manager deployments list
```
- Useful py lib
```
from setuptools import setup
python3 setup.py sdist
```
> Set public access to bucket (with cache headers not for CDN to cache it as best practice)
```
gsutil -h 'Content-Type: application/gzip' -h 'Cache-Control:private' cp -a public-read echo-0.0.1.tar.gz gs://$MY_BUCKET
```
- CAP theorem

- ACID & BASE

# --Load Balancing --
```
iperf -P
```
# --IAM --
```
gcloud iam service-accounts create test-service-account2 --display-name "test-service-account2"

gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT --member serviceAccount:test-service-account2@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com --role roles/viewer
```
- Proj. ID as default env variable
> $GOOGLE_CLOUD_PROJECT

- list current env
```
gcloud config list
```
- Switch to service account to shell
```
gcloud auth activate-service-account --key-file credentials.json
```
- See authenticated users
```
gcloud auth list
```
- Switch back to user account
```
gcloud config set account [USERNAME]
```
- Set to public
```
gsutil iam ch allUsers:objectViewer gs://$MY_BUCKET_NAME_1
```


# --Tooling --
```
# Disk measurement tools;
Bonnie++
Copy
Fio
Systhetic storage

# Network measurement tools;
Iperf
Mesh
Network
Netperf
Ping

# Workload Measurement;
Perfkit
Aerospike
Cassandra
Hadoop
HPCC
MongoDB
Oldisim
Redis
```
# --Kubernetes --

> Example docker file
```
FROM alpine
COPY quickstart.sh /
CMD ["/quickstart.sh"]
```
- Build Docker container in Cloud Build
```
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/quickstart-image .

gcloud builds submit --config cloudbuild.yaml .
```
**Services provide load-balanced access to specified Pods. There are three primary types of Services:**

>ClusterIP: Exposes the service on an IP address that is only accessible from within this cluster. This is the default type.

>NodePort: Exposes the service on the IP address of each node in the cluster, at a specific port number.

>LoadBalancer: Exposes the service externally, using a load balancing service provided by a cloud provider.
In Google Kubernetes Engine, LoadBalancers give you access to a regional Network Load Balancing configuration by default. To get access to a global HTTP(S) Load Balancing configuration, you can use an Ingress object.

**This reading explains the relationship among several Kubernetes controller objects:**
```
ReplicaSets
Deployments
Replication Controllers
StatefulSets
DaemonSets
Jobs
```
>A ReplicaSet controller ensures that a population of Pods, all identical to one another, are running at the same time. Deployments let you do declarative updates to ReplicaSets and Pods. In fact, Deployments manage their own ReplicaSets to achieve the declarative goals you prescribe, so you will most commonly work with Deployment objects.

>Deployments let you create, update, roll back, and scale Pods, using ReplicaSets as needed to do so. For example, when you perform a rolling upgrade of a Deployment, the Deployment object creates a second ReplicaSet, and then increases the number of Pods in the new ReplicaSet as it decreases the number of Pods in its original ReplicaSet.

>Replication Controllers perform a similar role to the combination of ReplicaSets and Deployments, but their use is no longer recommended. Because Deployments provide a helpful "front end" to ReplicaSets, this training course chiefly focuses on Deployments.

If you need to deploy applications that maintain local state, **StatefulSet** is a better option. A StatefulSet is similar to a Deployment in that the Pods use the same container spec. The Pods created through Deployment are not given persistent identities, however; by contrast, Pods created using StatefulSet have unique persistent identities with stable network identity and persistent disk storage.

If you need to run certain Pods on all the nodes within the cluster or on a selection of nodes, use DaemonSet. **DaemonSet** ensures that a specific Pod is always running on all or some subset of the nodes. If new nodes are added, DaemonSet will automatically set up Pods in those nodes with the required specification. The word "daemon" is a computer science term meaning a non-interactive process that provides useful services to other processes. A Kubernetes cluster might use a DaemonSet to ensure that a logging agent like fluentd is running on all nodes in the cluster.

The Job controller creates one or more Pods required to run a task. When the task is completed, Job will then terminate all those Pods. A related controller is **CronJob**, which runs Pods on a time-based schedule.


# --kubectl --
```
kubectl sends https req. to kubernetes api server in master
kubectl [command] [type] [name] [flags]
kubectl get pods -o=wide
kubectl get pod my-app -o=yaml
```
- Config file
```
$HOME/.kube/config

kubectl get pods
kubectl config view
```
- Updates config for new cluster
```
gcloud container clusters get-credentials [cluster_name] --zone [zone_name]
```
- Create cluster
```
gcloud container clusters create $my_cluster --num-nodes 3 --zone $my_zone --enable-ip-alias
```
- Resize cluster
```
gcloud container clusters resize $my_cluster --zone $my_zone --size=4
```
- To modify single cluster
```
gcloud container cluster
```
- Create config for cluster at $HOME
```
gcloud container clusters get-credentials $my_cluster --zone $my_zone
```
- After the kubeconfig file is populated and the active context is set to a particular cluster, you can use the kubectl
```
kubectl config view
```
- See current info
```
kubectl cluster-info
```
- See active
```
kubectl config current-context
```
- Change context
```
kubectl config use-context gke_${GOOGLE_CLOUD_PROJECT}_us-central1-a_standard-cluster-1
```
- Add bash autocompletion
```
source <(kubectl completion bash)
```
- Top for nodes
```
kubectl top nodes
```
- Top for pods
```
kubectl top pods
```
- Run image on a new pod
```
kubectl run nginx-1 --image nginx:latest

kubectl get pods
```
- Get pod info
```
kubectl describe pod $my_nginx_pod
```
- Copy file in Pod
```
kubectl cp ~/test.html $my_nginx_pod:/usr/share/nginx/html/test.html
```
- Expose POD port
```
kubectl expose pod $my_nginx_pod --port 80 --type LoadBalancer
```
- Get service details
```
kubectl get services
```
- Interactive shell
```
kubectl exec -it new-nginx /bin/bash
```
- Port forward
```
kubectl port-forward new-nginx 10081:80
```
- Get which Nodes running which pods
```
kubectl get pods -o=wide
```