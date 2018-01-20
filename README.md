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
### An [Azure account and subscription](https://azure.microsoft.com) with [Azure Portal access](https://azure.microsoft.com/en-us/features/azure-portal/)
### [Azure CLI](https://github.com/Azure/azure-cli) installed
#### Windows
[Download and run the MSI installer package](https://aka.ms/InstallAzureCliWindows).
#### Mac
Use Homebrew.  From Terminal, run the following:
```
brew update
brew install azure-cli
```
### Linux
[Read the docs](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)

Once you've successfully installed Azure CLI, open up your terminal and login to Azure with it
```
az login
```

## Create and connect to your cluster  

Once you've installed Azure CLI, open up a terminal window and create a resource group and the Azure region to create it in.
```
az group create --name myResourceGroup --location westeurope
```
(If you don't know which region to use or its shortname, get a list of them with this command.)
```
az account list-locations
```
After the resource group is created, create a Kubernetes cluster within it.
```
az acs create --orchestrator-type kubernetes --resource-group myResourceGroup --name myK8sCluster --generate-ssh-keys
```

## Configure your cluster

- Get the node info with 
```
[localhost]: kubectl get nodes
```
- Let the nodes be 
  - Node master-0 
  - Node agent-0 
  - Node agent-1 
  
### Steps
- Designate one node for storage through kubectl label command 
 ```
 [localhost]:kubectl label nodes <Node agent-0> type=store
 ```
- ssh to the storage node
  - To ssh to any node, first ssh to master node with the IP available on the web portal (create user and add password to all nodes through the portal, makes things easier :P)
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

#### Setup Htcondor

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
