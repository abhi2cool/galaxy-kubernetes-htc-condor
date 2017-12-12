# galaxy-kubernetes-htc-condor
## This is an attempt to replicate Galaxy docker swarm/compose implementation on kubernetes Deployed over ACS
### Pre-requisites
ACS kubernetes clusrter with one master and preferably two or more agent nodes 
- Let the nodes be 
  - Node master-0 
  - Node agent-0 
  - Node agent-1 
### Steps
- Designate one node for storage through kubectl label command 
 ```
 kubectl label pods <Node agent-0> type=store
 ```
- ssh to the storage node
  - make directory export with
  ```
  mkdir -p /export
  ```
- add "/export" to list of directories eligible for nfs mount with both read and write privileges
- ssh to all other nodes and mount "/export" directory with command
```
mkdir -p /export
sudo mount <Node agent-0>:/export /export
```
- expose required pods(services) with kubectl expose
```
expose pod galaxy-postgres
```
- create galxy webserver with command
```
kubectl create -f galaxy-web.yaml
```
- ssh to storage node 
```
cd /export
chmod 777 -R * 
```
#### Setup Htcondor
- create galaxy-htcondor with
```
kubectl create -f galaxy-htcondor.yaml
```
- expose service using
```
kubectl expose pod galaxy-htcondor
```
- create htcondor-executor and htcondor-executor-big with
```
kubectl create -f galaxy-htcondor-executor.yaml -f galaxy-htcondor-executor-big
```
- get shell to galaxy-htcondor pod 
  - edit etc/hosts
  - add 127.0.0.1   galaxy-htcondor
  - restart condor with condor_restart
