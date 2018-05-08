# Instructions to demo the Lab

The instructions defined hereafter are a summary about the official documentation that you can find from the `openshift-infra` project

- [Openshift Installation](https://github.com/snowdrop/openshift-infra/tree/3.9.0.SP2#openshift-deployment)
- [Post installation steps](https://github.com/snowdrop/openshift-infra/blob/3.9.0.SP2/ansible/README-post-installation.md)

and assume the following prerequisites 

**Prerequisites** :
- [Ansible - 2.4](http://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
- [Virtualbox](https://www.virtualbox.org/wiki/Downloads)
- [Centos VM](https://github.com/snowdrop/openshift-infra/tree/3.9.0.SP2#option-2--local---customized-linux-vm)

## Deploy OpenShift and needed features

- Git clone the `openshift-infra` project and checkout the latest release available for OpenShift (e.g : `3.9-SP2`)

  ```bash
  git clone -b 3.9.0.SP3 https://github.com/snowdrop/openshift-infra.git
  cd openshift-infra/ansible
  ```

- Git clone the `openshift-ansible` project (that we are using to provision OpenShift, Service Catalog) and checkout the release corresponding to the version used of OpenShift (E.g : `release-3.9`) 

  ```bash
  git clone -b release-3.9 https://github.com/openshift/openshift-ansible.git
  ```

- Generate the Ansible inventory to access your `local` Linux VM

  ```bash
  ansible-playbook playbook/generate_inventory.yml -e ip_address=192.168.99.50
  ```
  
  **Remark** : The scenario to install OpenShift on a vm managed by a cloud provider (Hetzner, Amazon, OpenStack) is the same. Please refer to the doc concerning what [could change](https://github.com/snowdrop/openshift-infra/blob/3.9.0.SP2/ansible/README-cloud.md) 

- Check if your VM matches the `prerequisites`

  ```bash
  ansible-playbook -i inventory/cloud_host openshift-ansible/playbooks/prerequisites.yml
  ```

- Install OpenShift

  ```bash
  ansible-playbook -i inventory/cloud_host openshift-ansible/playbooks/deploy_cluster.yml
  ```

- Post installation steps to : configure persistence, grant cluster-role to the admin user, install the Ansible Service Broker and Jenkins (S2I Build)

  ```bash
  ansible-playbook -i inventory/cloud_host playbook/post_installation.yml -e openshift_admin_pwd=admin --tags enable_cluster_admin
  ansible-playbook -i inventory/cloud_host playbook/post_installation.yml --tags persistence
  ansible-playbook -i inventory/cloud_host playbook/post_installation.yml --tags jenkins 
  ansible-playbook -i inventory/cloud_host playbook/post_installation.yml --tags nexus
  ansible-playbook -i inventory/cloud_host playbook/post_installation.yml -e infra_project=infra --tags jaeger
  ansible-playbook -i inventory/cloud_host openshift-ansible/playbooks/openshift-service-catalog/config.yml
  ```
  
- Install the Fabric8 Launcher using as Boosters's catalog the `cloud-native` catalog and your Git Hub account/token

```bash
ansible-playbook -i inventory/cloud_host playbook/post_installation.yml \
     --tags install-launcher \
     -e launcher_catalog_git_repo=https://github.com/snowdrop/cloud-native-catalog.git \
     -e launcher_catalog_git_branch=master \
     -e launcher_github_username=YOUR_GIT_TOKEN \
     -e launcher_github_token=YOUR_GIT_USER     
```  

## Lab's demo

See [Hands On Lab - objectives](HANDS_ON_LAB.md). Use master branch instead of the student's branch to use the solution !

## Temp commands - May 7 - 2018

```bash
# EXISTING PROJECT
echo "Existing project"
cd cloud-native-demo
rm *.zip
setopt rmstarsilent
rm -Rf booster-demo-backend-spring-boot/*
rm -Rf booster-demo-frontend-spring-boot/*

OR

# NEW PROJECT
echo "New project"
mkdir -p cloud-native-demo && cd cloud-native-demo

oc project default
oc delete project/demo 
oc scale --replicas=0 deployment jaeger -n infra
oc scale --replicas=1 deployment jaeger -n infra

oc new-project demo

# FRONTEND PART
echo "Use launcher to download Frontend"
cp ~/Downloads/booster-demo-frontend-spring-boot.zip .
unzip booster-demo-frontend-spring-boot.zip
cd booster-demo-frontend-spring-boot

!! Rename artifact to `cloud-native-frontend`

mvn clean compile
mvn clean spring-boot:run 
open http://localhost:8090

mvn package fabric8:deploy -Popenshift
...
export FRONTEND=$(oc get route/cloud-native-frontend --template '{{.spec.host}}') 
open http://$FRONTEND

# SERVICE CATALOG PART
See HOL instructions

# BACKEND part
echo "Use launcher to download Backend"
cp ~/Downloads/booster-demo-backend-spring-boot.zip .
unzip ~/Downloads/booster-demo-backend-spring-boot.zip
cd booster-demo-backend-spring-boot
mvn clean spring-boot:run -Ph2 -Drun.arguments="--spring.profiles.active=local,--jaeger.sender=http://jaeger-collector-infra.192.168.99.50.nip.io/api/traces,--jaeger.protocol=HTTP,--jaeger.port=0"
http http://localhost:8080/api/notes 
curl -k -H "Content-Type: application/json" -X POST -d '{"title":"My first note","content":"Spring Boot is awesome!"}' http://localhost:8080/api/notes 
http http://localhost:8080/api/notes/1
...

oc new-app -f openshift/cloud-native-demo_backend_template.yml
oc start-build cloud-native-backend-s2i --from-dir=. --follow
...

echo "Apply the secret to the backend app and wait ... till it will rebuild to play with the frontend"
export FRONTEND=$(oc get route/cloud-native-frontend --template '{{.spec.host}}') 
open http://$FRONTEND

echo "Play with Distributed tracing"
export JAEGER=$(oc get route/jaeger-query --template '{{.spec.host}}' -n infra)
open http://$JAEGER

echo "Debug app"
oc env dc/cloud-native-frontend JAVA_ENABLE_DEBUG=true
export POD_DEBUG=$(oc get pod -l app=cloud-native-frontend -o jsonpath='{.items[0].metadata.name}')
oc port-forward $POD_DEBUG 5005:5005

echo "Run integration test"
mvn clean verify -Popenshift-it

echo "Scale pods"
oc scale --replicas=2 dc cloud-native-frontend
oc get pods -l app=cloud-native-frontend
curl -v http://$FRONTEND | grep 'id="_http_booster"'

echo "Use jenkins pipeline"
oc delete bc/cloud-native-backend-s2i 
oc adm policy add-cluster-role-to-user edit system:serviceaccount:infra:jenkins -n $(oc project -q)
chmod +x create-pipeline.sh
chmod +x start-pipeline.sh 
./create-pipeline.sh
./start-pipeline.sh 
```


  