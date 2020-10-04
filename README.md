# ELK stack on kubernetes cluster
Elasticsearch, Logstash, Kibana and filebeat installation on kubernetes cluster using Jenkins CICD pipeline.

## Prerequisites
* kubernetes
* Ansible >= 2.9 version
* Docker

## Jenkins Installation
Jenkins installation is done via ansible-playbook with a custom docker image and making use of jenkins configuration as code plugin.
### Jenkins image
Dockerfile includes installation of
* Jenkins basic suggested plugins along with plugins like git, kubernetes, job-dsl etc
* Docker installation to enable docker commands inside Jenkins
* Ansible package
* kubectl package
### Jenkins slave image
A custom slave image is built from official jnlp image. This is important as we run all our ELK stack playbooks here. Similar to Jenkins image, Dockerfile consists of below 
* Docker
* Ansible
* kubectl
* kubeval
* polaris
### Jenkins configuration
Configuration is completely managed through Jenkins configuration as code plugin. It includes
* Setting up user access.
* Creates kubernetes cloud and adding pod template with our custom jenkins slave image in it
* Create our first job for ELK CICD pipeline with github hook enabled.
### Before running jenkins playbook
- Copy kubeconfig file to user home directory path("~/.kube/config") if not already present.
- Update below variables in jenkins inventory file
  * kubernetes api server address
  * **kubeconfig_file** - Convert file into base64 format before adding to inventory. 
   ```
   cat ~/.kube/config | base64 -w 0
   ```
  * **docker_login_username** - Docker hub login username
  * **docker_login_password**  - Docker hub login password
  * **github_repo_token** - Token to github repository
  * **adminuser_password** - Password for logging to Jenkins instance as "admin" user.

If needed other variable values can be changed based on requirement.

Note: For encrypting passwords and kubeconfig content before adding to inventory file, follow https://docs.ansible.com/ansible/latest/user_guide/vault.html#creating-encrypted-variables

### Running Jenkins playbook
Execute below ansible command and it should prompt for the vault password which is used previously to encrypt strings.
```
ansible-playbook jenkins.yml -i inventories/stage --ask-vault-pass
```
### Workflow of Jenkins playbook
* Builds core Jenkins image with custom plugins and packages. Pushes it to Dockerhub.
* Installs Jenkins on kubernetes cluster using ansible k8s module. i.e applying role-binding, configmap, service, deployment etc
* Builds Jenkins slave image and push it to Dockerhub

## ELK Stack
Elasticsearch, logstash, kibana and filebeats together are ELK stack. Installation includes three pod elasticsearch cluster, two logstash, one Kibana and one filebeat.
Below is the architecture of ELK setup.

![ELK Setup](https://github.com/vivekreddy94/elk-kubernetes/blob/master/images/elk_architecture.png)

### Overview of setup
**Elasticsearch**: Three pod elasticsearch cluster is installed where they all talk to eachother to maintain replicas of incoming data and indices. As it is stateful application with requirement of persistent data, 'statefulsets' object is used which creates unique persisten identifier and recreates the pod with same identifier on failures. For data storage, persistent volumes are created and claimed through statefulset. Configuration is loaded to elasticsearch through configmaps.

**Logstash**: Two pod logstash in installed to share incoming data load. Logstash is deployed through 'deployment' object and configuration is loaded through configmap. Logstash is configured to accept data on 5044 beats port from filebeats, then applies filters and other operations on data to finally send it to elasticsearch cluster.

**Kibana**: One pod kibana is installed to visualize and quer data from elasticsearch. Similar to logstash, kibana is deployed via 'deployment' set and configuration is loaded through configmap.

**filebeat**: One pod filebeat is installed to gather logs from docker containers and send it to logstash. For deploying filebeat 'daemonset' object is used as we want to pod to run all kubernetes cluster nodes. In this case only one pod is deployed, but on addition of new nodes to kuberenetes cluster, daemonset takes care of initiating pod on new nodes.

## ELK deployment strategy
![ELK stack deployment](https://github.com/vivekreddy94/elk-kubernetes/blob/master/images/elk_jenkins_deploy.png)

Workflow of ELK stack deployment in relation to above image
1. Github notifies Jenkins elk CICD pipeline job on a new commit.
2. Jenkins master using kubernetes plugin requests a slave pod.
3. Jenkins slave pod is created dynamically by pulling image from docker registry
4. CICD pipeline using kubeconfig file requests kubernetes api to deploy ELK stack
5. ELK stack is deployed on kubernetes cluster

### CICD Pipeline
Below explains the steps performed in each stage of pipeline.

![Pipeline stages](https://github.com/vivekreddy94/elk-kubernetes/blob/master/images/cicd_arch.png)

#### Checkout
Checks out elk stack github repo.

#### Test slave setup
Checks the slave if required packages ansible, kubectl, docker, kubeval and polaris are installed as these are necessary for coming pipeline stages.

#### Setup requirements
* Copies the kubeconfig file from jenkins credentails to home directory in jenkins slave.
* Executes kubernetes-lingting.yml playbook to copy ELK stack kubernetes files to slave node and make it ready for [Validate kubernetes code](https://github.com/vivekreddy94/elk-kubernetes#validate-kubernetes-code) stage

#### Perform ansible linting
* Builds a custom ansible-lint docker image by coping ansible files into the image
* Runs docker image to perform linting on elk_stack.yml playbook
* Finally cleans up ansible-linting docker image

#### Validate kubernetes code
* Kubernetes files are validated by [kubeval](https://kubeval.instrumenta.dev/) 
* Next the files are validated by [polaris](https://github.com/FairwindsOps/polaris) for doing checks on images, tags, security priviliges, probes etc.

#### Deploy and test elasticsearch
* Deploys elasticsearch using ansible playbook
```
ansible-playbook elasticsearch.yml -i inventories/stage
```
* Checks if all pods in statefulset are ready
* Loads sample data to elasticsearch cluster
* Compares the loaded data with expected test output data.

#### Deploy and test logstash
* Deploys logstash using ansible playbook. For testing logstash, deployment includes 'http' plugin in logstash configuration to allow input data on 8080 port and creates output log file at /tmp/output.log
```
ansible-playbook logstash.yml -i inventories/stage
```
* Check if logstash pod is ready.
* Loads sample data into logstash at 8080.
* Tests if the output log file is created.

#### Deploy and test filebeat
* Deploys filebeat using ansible playbook.
```
ansible-playbook filebeat.yml -i inventories/stage
```
* Check if filebeat pod is ready.
* Test filebeat configuration if its able to communicate with logstash

#### Deploy and test kibana
* Deploys kibana using ansible playbook.
```
ansible-playbook kibana.yml -i inventories/stage
```
* Check if kibana pod is ready
* Check if kibana port is listening.

#### Load data, test and cleanup
* Executes data_loading_pods.yml playbook to deploy custom pods for generating 100000 random logs.
* All the components are tested again after loading data.
* Elasticsearch is queried to check if 100000 logs are loaded.
* Finally, removes all components.
```
ansible-playbook elk_stack.yml -i inventories/stage --extra-vars "install_action=absent"
```

#### Deploy on production
* Deploys [reloader](https://github.com/stakater/Reloader) for performing rolling upgrade on deployment, daemonset, and statefulsets.
* Deploys elk_stack.yml playbook which includes elasticsearch, logstash, filebeat and kibana playbooks
```
ansible-playbook elk_stack.yml -i inventories/production
```
Note: Kubeconfig file of production cluster should be loaded here, but as it is a single node cluster kubeconfig file loaded in [Setup requirements](https://github.com/vivekreddy94/elk-kubernetes#setup-requirements) stage will be reused.



