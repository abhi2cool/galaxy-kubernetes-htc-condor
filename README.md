# galaxy-kubernetes-htcondor-azure
# Overview
This repository reflects a cloud-based implementation designed to meet the following design criteria:
- Azure [container-based]() implementation of [Galaxy](https://usegalaxy.org/) biomedical research portal with [FTP upload directly from the portal](https://galaxyproject.org/ftp-upload/)
 - [Kubernetes](https://kubernetes.io/) container-based deployment
 - [HTCondor](https://research.cs.wisc.edu/htcondor/) for dynamic job and node scaling
 - a minimum of 60 TB of addressable (NFS) storage
 - [Helm](https://helm.sh/) chart for package management

The challenge was to enable a performant, cost-efficient configuration that could support the [Neurolincs](http://neurolincs.org/) ATAQSeq and RNASeq workflows on Azure in support of _its_ [AnswerALS Foundation research](http://research.answerals.org/) commitment _now_, but in a way that anticipates and is informed by a future [CloudMan release](url?) that will enable full cloud configurability and scaling directly from within Galaxy itself.

Until that time, however, a fully optimized cloud implementation with the current version of Galaxy, even container-based versions, still suffer from limitations imposed when supporting FTP upload and concomitant [POSIX-based filesystem](https://en.wikipedia.org/wiki/POSIX) and NFS constraints. 

As such, in lieu of full Helm chart external configurability,  a non-trivial amount of post-deployment steps and incantations obtain. (More fully decoupled and configurable implementations using persistent volume claims and whatnot were explored and achieved, but initial performance was _dismal_ and ultimately incompatible with NFS.)

This implementation, while less purdy, _performs_ and easily scales to multiple agents running multiple nodes or pods, with data uploaded through the ftp server participating directly.

# Implementation

While fundamentally a cloud-neutral implementation based on Kubernetes and HTCondor, this readme describes steps for Azure.

## Prerequisites
* An [Azure account and subscription](https://azure.microsoft.com) with [Azure Portal access](https://azure.microsoft.com/en-us/features/azure-portal/)
* [Azure CLI](https://github.com/Azure/azure-cli) installed on the computer you'll be working from

Azure CLI is available for multiple platforms.  If you don't see an option below that works for you, [check the docs](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest).
### Windows
[Download and run the MSI installer package](https://aka.ms/InstallAzureCliWindows).
### Mac
Use Homebrew.  From Terminal, run the following:
```
rc-cola$ brew update
brew install azure-cli
```
### Linux
[Read the docs](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)

### Verify install
Once you've successfully installed Azure CLI, open up your terminal and login to Azure with it
```
rc-cola$ az login
```
You should be prompted to log in from your system's default browser.
```
rc-cola$ az login
To sign in, use a web browser to open the page https://aka.ms/devicelogin and enter the code 'YOUR_CODE' to authenticate.
```
Congrats!  We're on our way.
## Create and connect to your cluster  

Once you've installed Azure CLI, open up a terminal window and create a resource group and the Azure region to create it in.
```
rc-cola$ az group create --name myResourceGroup --location westeurope
```
(If you don't know which region to use or its shortname, get a list of them with this command.)
```
rc-cola$ az account list-locations
```
After the resource group is created, create a Kubernetes cluster within it.
```
rc-cola$ az acs create --orchestrator-type kubernetes --resource-group myResourceGroup --name myK8sCluster --generate-ssh-keys
```

## Install kubectl

To manage our Kubernetes cluster, we need to [install kubectrl](https://kubernetes.io/docs/tasks/tools/install-kubectl/).
### Windows
On Windows you can use the Chocolatey package manager and install with:
```
$ choco install kubernetes-cli
```
### Mac
Use Homebrew.  From Terminal, run the following:
```
rc-cola$ brew install kubectl
```
### Linux
kubectl is available as a snap application.
If you are on Ubuntu or one of other Linux distributions that support snap package manager, you can install with:
```
rc-cola$ sudo snap install kubectl --classic
```
### Verify your install
```
rc-cola$ kubectl version
```
## Access your k8s cluster
Now that you've got kubectl installed, let's get some info about the cluster we just created

### Get node info from the kubectl CLI
```
rc-cola$ kubectl get nodes
NAME                    STATUS    ROLES     AGE       VERSION
k8s-agent-e0c9b167-0    Ready     agent     12d       v1.7.7
k8s-agent-e0c9b167-1    Ready     agent     12d       v1.7.7
k8s-agent-e0c9b167-2    Ready     agent     12d       v1.7.7
k8s-master-e0c9b167-0   Ready     master    12d       v1.7.7
```
Remember those names.  We're going to need to ssh into *all* of them (and into the agent nodes _through_ the master) to do stuff like mount NFS volumes and whatnot, so keep them handy :).

### Access the kubernetes web UI
The kubernetes web UI is an excellent tool to configure scaling options manually. To access it, run this command from your Terminal command prompt:

```
rc-cola$ az acs kubernetes browse -g rcAnswerAlsCluster -n rcAlsK8s 
Proxy running on 127.0.0.1:8001/ui
Press CTRL+C to close the tunnel...
Starting to serve on 127.0.0.1:8001
```  
You should see something like this.
![kubernetes web ui dashboard](https://raw.githubusercontent.com/rc-ms/galaxy-kubernetes-htcondor-azure/master/assets/kubernetes-web-ui.png)

## Set up an agent as Galaxy storage node and NFS server
Now that we've built and accessed our shiny new cluster, we can start getting it ready for our Galaxy installation.

### Set up the '-0' agent as a storage node
From your node listing above, select the '-0' _agent_ node as a storage node.  Run the following command in your Terminal window
 ```
 rc-cola$ kubectl label nodes k8s-agent-e0c9b167-0 type=store
 ```
 ### Create a user account on all the nodes to enable SSH access
 
 Before we can begin our node configuration dance, we have to create user accounts on all the nodes. For this we're going to use the Azure Portal.

 In your browser, head over to the [Azure Portal](https://portal.azure.com) and log in using the same credentials you used to create your subscription.

 When you get there, navigate to your Kubernetes cluster by browsing (select "**Resource Groups**" from left nav and then click into the resource group you created with the CLI ('rcAnswerAlsCluster' for this demo) or searching (start typing your cluster name in the search box and it should appear via autocomplete)
 ![finding your k8s cluster in the Azure Portal](https://github.com/rc-ms/galaxy-kubernetes-htcondor-azure/blob/master/assets/find-your-cluster.png?raw=true)

 

### Configure storage node as an NFS Server
 This is where it's easy to get lost. To access the agent nodes to configure them, you first have to ssh to the master node and then ssh from there to the agent nodes.

  - then ssh to any desired node with
  ```
  [Node master-0]:-# ssh username@Node agent-0
  ```
  - make directory export with
  ```
  [Node agent-0]: mkdir -p /export
  [Node agent-0]: chown nobody:nogroup /export
  ```
 -install NFS server 
 ```
 [Node agent-0]: sudo apt install nfs-kernel-server
 ```
    - add "/export" to list of directories eligible for nfs mount with both read and write privileges
    - You can configure the directories to be exported by adding them to the /etc/exports file. For example:
    ```
    /ubuntu  *(ro,sync,no_root_squash)
    /export    *(rw,sync,no_root_squash)
    ```
 - start service with
 ```
 [Node agent-0]: sudo systemctl start nfs-kernel-server.service
 ```
- ssh to all other nodes
```
[Node any]:-# ssh username@Node agent-(all except 0)
[Node agent-(all except 0)]: mkdir -p /export
[Node agent-(all except 0)]: sudo mount <Node agent-0>:/export /export
```
- Install helm chart with
```
[localhost]: helm install galaxy
```
- ssh to storage node using same steps above 
```
[Node agent-0]: cd /export
[Node agent-0]: chmod 777 -R * 
```
  - this step needs to be done in order to get rid of the permission denied error
  - check if galaxy is running with
  ```
  [localhost]: kubectl port-forward galaxy 8080:80
  ```
  - browse to localhost:8080

## Set up HTCondor

- get shell to galaxy-htcondor container
```
[localhost]:kubectl exec -it galaxy-htcondor -- /bin/bash
```
  - edit file /etc/hosts
  ```
  [root@galaxy-htcondor]: vi /etc/hosts
  ```
  - add 127.0.0.1   galaxy-htcondor
- get shell to galaxy-htcondor-executor container
```
[localhost]:kubectl exec -it galaxy-htcondor-executor -- /bin/bash
```
  - edit file /etc/hosts
  ```
  [root@galaxy-htcondor-executor]: vi /etc/hosts
  ```
  - add 127.0.0.1   galaxy-htcondor-executor 
- get shell to galaxy container
```
[localhost]:kubectl exec -it galaxy --container=galaxy  -- /bin/bash
```
  - edit file /etc/condor/condor_config.local
  ```
  [root@galaxy-htcondor]: vi /etc/condor/condor_config.local
  ```
  - add the following lines
  ```
  HOSTALLOW_READ = *
  HOSTALLOW_WRITE = *
  HOSTALLOW_NEGOTIATOR = *
  HOSTALLOW_ADMINISTRATOR = *
  ```
  - restart condor with 
  ```
    [root@galaxy]:condor_restart
  ```
