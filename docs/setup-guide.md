# Setup Guide

This guide contains the setup instructions for *gluster-kubernetes*.


## Infrastructure Requirements

* A running Kubernetes cluster with at least three Kubernetes worker nodes that each have an available raw block device attached (like an EBS Volume or a local disk).
* The three Kubernetes nodes intended to run the GlusterFS Pods must have the appropriate ports opened for GlusterFS communication. Run the following commands on each of the nodes.  
```
iptables -N HEKETI
iptables -A HEKETI -p tcp -m state --state NEW -m tcp --dport 24007 -j ACCEPT
iptables -A HEKETI -p tcp -m state --state NEW -m tcp --dport 24008 -j ACCEPT
iptables -A HEKETI -p tcp -m state --state NEW -m tcp --dport 2222 -j ACCEPT
iptables -A HEKETI -p tcp -m state --state NEW -m multiport --dports 49152:49251 -j ACCEPT
```
**** TODO: Needs to be updated to demonstrate how to persist your new iptables configuration so that it is still the same after a reboot of the host. Also needs an update to link in the new HEKETI iptables chain so that it is active. Right now the chain is not referenced so it is just set up but not being used. ****

## Client Setup

Heketi provides a CLI that provides users with a means to administer the deployment and configuration of GlusterFS in Kubernetes. [Download and install the heketi-cli](https://github.com/heketi/heketi/releases/tag/v3.0.0) on your client machine (usually your laptop). This cli will later connect to the heketi service through port-forwarding.


## Create a service account

We need to create a Service Account so that the GlusterFS containers can communicate securely with each other. We use the shipped [heketi-service-account.yaml](../config/heketi-service-account.yaml) file:
```
# kubectl create -f heketi-service-account.yaml
```

Verify the serviceaccount was created successfully
```
# kubectl get serviceaccount
NAME                       SECRETS   AGE
default                    1         27d
heketi-service-account     1         15s
```

Verify the appropriate secrets were created with the serviceaccount
```
# kubectl get secrets
NAME                                   TYPE                                  DATA      AGE
default-token-4wtpz                    kubernetes.io/service-account-token   3         27d
heketi-service-account-token-dz077     kubernetes.io/service-account-token   3         25s
```

## Install glusterfs storage nodes in kubernetes

Next we are going to deploy the Heketi Service Interface for GlusterFS as well as the GlusterFS ReplicaSets. To do this, we need to know the Kubernetes worker nodes that we are going to be deploying to. Let's start by obtaining a list of all our Kubernetes worker nodes: 

```
# kubectl get nodes
NAME                           STATUS    AGE
ip-172-20-0-217.ec2.internal   Ready     3h
ip-172-20-0-218.ec2.internal   Ready     3h
ip-172-20-0-219.ec2.internal   Ready     3h
ip-172-20-0-220.ec2.internal   Ready     3h
```

Within that node list you should see the servers you have identified to run GlusterFS (at least 3 of them). There is a  glusterfs-deployment.json file within this repo that you cloned. Run the following command for each of the 3 (or more) servers that you've set aside to run GlusterFS while making sure that the hostname or IP you provide matches the value for the node name returned from the previous command:

`# sed 's/<GLUSTERFS_NODE>/<hostname or IP>/g' glusterfs-deployment.json | kubectl create -f -`

Example:

`# sed 's/<GLUSTERFS_NODE>/ip-172-20-0-219\.ec2.internal/g' glusterfs-deployment.json | kubectl create -f -`


Verify that the deployments ran successfully by running the following command:
```
kubectl get deployments
NAME                                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
glusterfs-ip-172-20-0-217.ec2.internal   1         1         1            1           2m
glusterfs-ip-172-20-0-218.ec2.internal   1         1         1            1           2m
glusterfs-ip-172-20-0-219.ec2.internal   1         1         1            1           3m
```

## Create and configure the heketi service

Next we need to create the Pod and Service **heketi**, which will be our service interface to the GlusterFS cluster. This is done in multiple steps: We first install an intermediate instance used for bootstrapping, called *deploy-heketi*. This is mainly needed so that the final heketi instance can store its database on a shared gluster instead of on local, volatile container storage.

### Create a bootstrapping heketi

We are using the shipped [deploy-heketi-deployment.json](../config/deploy-heketi-deployment.json) file to create the deploy-heketi instance.

* Note the secret for the service account  
```
$ heketi_secret=$(kubectl get sa heketi-service-account -o="go-template" --template="{{(index .secrets 0).name}}")
```

* Create deploy-heketi.  Before deploying you will need to determine the Kubernetes API endpoint and namespace.  
In this example, we will use `https://1.1.1.1:443` as our Kubernetes API endpoint  
```
$ sed -e "s#<HEKETI_KUBE_SECRETNAME>#\"$heketi_secret\"#" \
      -e "s#<HEKETI_KUBE_APIHOST>#\"http://1.1.1.1:443\"#" deploy-heketi-deployment.json | kubectl create -f -
service "deploy-heketi" created
deployment "deploy-heketi" created
```

* verify everything is running properly as demonstrated below:  
```
# kubectl get pods
NAME                                                      READY     STATUS    RESTARTS   AGE
deploy-heketi-1211581626-2jotm                            1/1       Running   0          35m
glusterfs-ip-172-20-0-217.ec2.internal-1217067810-4gsvx   1/1       Running   0          1h
glusterfs-ip-172-20-0-218.ec2.internal-2001140516-i9dw9   1/1       Running   0          1h
glusterfs-ip-172-20-0-219.ec2.internal-2785213222-q3hba   1/1       Running   0          1h
```

### Configure port-forwarding for heketi

Now that the Bootstrap Heketi Service is running we are going to configure port-fowarding so that we can communicate with the service using the Heketi CLI. Using the name of the Heketi pod, run the command below:

`kubectl port-forward deploy-heketi-1211581626-2jotm :8080`

Now verify that the port forwarding is working, by running a sample query againt the Heketi Service. The command should return a local port that it will be forwarding from. Incorporate that into a localhost query to test the service, as demonstrated below:

```
curl http://localhost:57598/hello
Handling connection for 57598
Hello from Heketi
```
Lastly, set an environment variable for the Heketi CLI client so that it knows how to reach the Heketi Server.

`export HEKETI_CLI_SERVER=http://localhost:57598`


### Prepare topology and load it into heketi

Next we are going to provide Heketi with the information about the GlusterFS cluster it is to manage. We provide this information via [a topology file](https://github.com/heketi/heketi/wiki/Setting-up-the-topology). There is a sample topology file within the repo you cloned called topology-sample.json. Topologies primarily specify what Kubernetes Nodes the GlusterFS containers are to run on as well as the corresponding available raw block device for each of the nodes. Modify the topology file to reflect the choices you have made and then deploy it as demonstrated below:

MAKE SURE TO ONLY USE IP ADDRESSES IN THE HOSTNAMES SECTION

```
heketi-client/bin/heketi-cli topology load --json=topology-sample.json
Handling connection for 57598
	Found node ip-172-20-0-217.ec2.internal on cluster e6c063ba398f8e9c88a6ed720dc07dd2
		Adding device /dev/xvdg ... OK
	Found node ip-172-20-0-218.ec2.internal on cluster e6c063ba398f8e9c88a6ed720dc07dd2
		Adding device /dev/xvdg ... OK
	Found node ip-172-20-0-219.ec2.internal on cluster e6c063ba398f8e9c88a6ed720dc07dd2
		Adding device /dev/xvdg ... OK
```

### Create a volume for the heketi database

Next we are using the bootstrap Heketi instance to create a volume for the final heketi to storage its database:

```
# heketi-client/bin/heketi-cli setup-openshift-heketi-storage
# kubectl create -f heketi-storage.json
```

### Delete the bootstrap Heketi

Wait unti the volume creation is complete, and then run:

`# kubectl delete all,service,jobs,deployment,secret --selector="deploy-heketi" `


### Install the final heketi service

Before deploying you will need to determine the Kubernetes API endpoint and namespace.
In this example, we will use `https://1.1.1.1:443` as our Kubernetes API endpoint.
We are using the shipped [heketi-deployment.json](../config/heketi-deployment.json) file:

```
$ sed -e "s#<HEKETI_KUBE_SECRETNAME>#\"$heketi_secret\"#" \
      -e "s#<HEKETI_KUBE_APIHOST>#\"http://1.1.1.1:443\"#" heketi-deployment.json | kubectl create -f -
service "heketi" created
deployment "heketi" created
```


# Usage Example

Normally you would need to setup another project for an applications, and setup the appropriate [endpoints and services](https://github.com/kubernetes/kubernetes/tree/master/examples/glusterfs).  This creates 100GB Persistent volume which can be claimed from any application.

Create a 100G volume:

```
$ export HEKETI_CLI_SERVER=http://<address to heketi service>
$ heketi-cli volume create --size=100 \
  --persistent-volume \
  --persistent-volume-endpoint=heketi-storage-endpoints | kubectl create -f -
```

http://kubernetes.io/docs/user-guide/persistent-volumes/#claims-as-volumes
