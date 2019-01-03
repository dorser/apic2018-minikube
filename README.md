# API Connect 2018 on Minikube
This document will take in hand in hand to deploy API Connect v2018.4.1.1 locally on your laptop using Minikube.
This is still work in progress and I haven't written the prereqs part yet (As most of my colleagues are familiar with it).
## Installing Prerequisites //TODO:
* Install docker
* Install kubectl
* Install helm
* Install minikube
* Download IBM resources
	* Download images
	* Download toolkit
	* Download apicup

### Create local docker registry //TODO:
https://hub.docker.com/_/registry/
Added the registry to my /etc/hosts file as "registry.apic.lab" and it points to the virtualbox interface address.

### Upload APIC images to the registry //TODO:
Download the .tgz files and push the images to the docker registry using apicup

## Create a Minikube machine
This configuration allows the deployment of Management, Portal and Gateway. Analytics requires more memory (6GB more)

    minikube start --cpus 8 --memory 24576 --insecure-registry registry.apic.lab:5000 --disk-size=100GB

Once Minikube is ready, let's get its IP:

    minikube ip

We need to create "DNS records" but instead of using a DNS server we can use the host's /etc/hosts file instead.

    sudo vi /etc/hosts

And add the following records to your hosts file (Replace '192.168.99.100' with your Minikube's IP):

    192.168.99.100 platform-api.apic.lab
    192.168.99.100 api-manager-ui.apic.lab
    192.168.99.100 cloud-admin-ui.apic.lab
    192.168.99.100 consumer-api.apic.lab
    192.168.99.100 api-gateway.apic.lab
    192.168.99.100 portal-admin.apic.lab
    192.168.99.100 portal-www.apic.lab
    192.168.99.100 analytics-ingestion.apic.lab
    192.168.99.100 analytics-client.apic.lab

Now we need to do some basic configuration on our Minikube, let's start by creating a namespace:

    kubectl create namespace apiconnect

Now we need to create a secret for our Docker Registry which we created before:

    kubectl create secret docker-registry registry-secret --docker-server=registry.apic.lab:5000 --docker-username=admin --docker-password=password --docker-email=registry@apic.lab -n apiconnect

Create cluster role binding (Authorization record which allows us to deploy stuff to our namespace):
`kubectl create clusterrolebinding add-on-cluster-admin --clusterrole=cluster-admin --serviceaccount=apiconnect:default`

apicup uses helm charts under the hood. For Helm to deploy we need to initialize it (Creating a tiller pod on our namespace).

    export TILLER_NAMESPACE=apiconnect
    helm init --upgrade

Wait for tiller to come online. You can check its status using:
    watch kubectl get pod -n apiconnect

The output will be similar to this:

    NAME                             READY     STATUS    RESTARTS   AGE
    tiller-deploy-6bfd959c48-hgr7f   1/1       Running   0          52s

Now we will deploy nginx which will serve us as our cluster's load balancer:

    helm install --name ingress -f nginx-values.yml  stable/nginx-ingress --namespace apiconnect

Wait for nginx to come online

    watch kubectl get pod -n apiconnect
The output will be similar to this:

    NAME                                                     READY     STATUS    RESTARTS   AGE
    ingress-nginx-ingress-controller-d9xvz                   1/1       Running   0          25s
    ingress-nginx-ingress-default-backend-845f7f5785-mbx8p   1/1       Running   0          25s

Now we are ready to install API Connect 2018 on our Minikube!

## API Connect 2018 Installation

We start by creating a new directory and initializing our apic project there:

    mkdir apic-lab
    cd apic-lab
    apicup init

### Creating the management subsystem
The following are suggested defaults. If you were following my naming convention so far then just copy and paste it as is:

    apicup subsys create manager management --k8s
    apicup subsys set manager platform-api platform-api.apic.lab
    apicup subsys set manager api-manager-ui api-manager-ui.apic.lab
    apicup subsys set manager cloud-admin-ui cloud-admin-ui.apic.lab
    apicup subsys set manager consumer-api consumer-api.apic.lab
    apicup subsys set manager registry registry.apic.lab:5000
    apicup subsys set manager registry-secret registry-secret
    apicup subsys set manager mode dev
    apicup subsys set manager namespace apiconnect
    apicup subsys set manager storage-class standard

Deploy the subsystem using:

    apicup subsys install manager

### Creating the gateway subsystem
The following are suggested defaults. If you were following my naming convention so far then just copy and paste it as is (Deploying **non-compatible** v5 gateway):

    apicup subsys create gwy gateway --k8s
    apicup subsys set gwy api-gateway api-gateway.apic.lab
    apicup subsys set gwy apic-gw-service apic-gw-service.apic.lab
    apicup subsys set gwy namespace apiconnect
    apicup subsys set gwy image-repository ibmcom/datapower
    apicup subsys set gwy image-tag 2018.4.1.1.305192
    apicup subsys set gwy image-pull-policy IfNotPresent
    apicup subsys set gwy replica-count 1
    apicup subsys set gwy storage-class standard
    apicup subsys set gwy v5-compatibility-mode false
    apicup subsys set gwy enable-tms true
    apicup subsys set gwy mode dev
    apicup subsys set gwy max-cpu 4

Deploy the subsystem using:

    apicup subsys install gwy

Once you are done, find the ClusterIP of the gateway service and add it to your /etc/hosts file:

    kubectl get svc -n apiconnect | grep dynamic-gateway-service-ingress
You'll see output similar to this:

    r554d996560-dynamic-gateway-service-ingress   ClusterIP      10.100.150.111   <none>        3000/TCP,9443/TCP                              1h
Copy the ClusterIP and create the following record in your /etc/hosts:

    10.100.150.111 apic-gw-service.apic.lab

### Creating the portal subsystem
The following are suggested defaults. If you were following my naming convention so far then just copy and paste it as is:

    apicup subsys create ptl portal --k8s
    apicup subsys set ptl portal-admin portal-admin.apic.lab
    apicup subsys set ptl portal-www portal-www.apic.lab
    apicup subsys set ptl namespace apiconnect
    apicup subsys set ptl storage-class standard
    apicup subsys set ptl registry registry.apic.lab:5000
    apicup subsys set ptl registry-secret registry-secret
    apicup subsys set ptl mode dev
Deploy the subsystem using:

    apicup subsys install ptl

### Creating the analytics subsystem
**Only if you created a Minikube machine with 30GB Memory. Otherwise, the pods won't start.**
The following are suggested defaults. If you were following my naming convention so far then just copy and paste it as is:

    apicup subsys create analyt analytics --k8s
    apicup subsys set analyt analytics-ingestion analytics-ingestion.apic.lab
    apicup subsys set analyt analytics-client analytics-client.apic.lab
    apicup subsys set analyt namespace apiconnect
    apicup subsys set analyt registry registry.apic.lab:5000
    apicup subsys set analyt registry-secret registry-secret
    apicup subsys set analyt storage-class standard
    apicup subsys set analyt mode dev
    apicup subsys set analyt coordinating-max-memory-gb 6
    apicup subsys set analyt data-max-memory-gb 6
     apicup subsys set analyt master-max-memory-gb 6
Deploy the subsystem using:

    apicup subsys install analyt


**Enjoy!**
