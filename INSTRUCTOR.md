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
  git clone -b 3.9.0.SP2 https://github.com/snowdrop/openshift-infra.git
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
  ansible-playbook -i inventory/cloud_host playbook/post_installation.yml -e openshift_admin_pwd=admin --tags "enable_cluster_admin"
  ansible-playbook -i inventory/cloud_host playbook/post_installation.yml -e openshift_admin_pwd=admin --tags "identity_provider" 
  ansible-playbook -i inventory/cloud_host playbook/post_installation.yml --tags persistence 
  ansible-playbook -i inventory/cloud_host playbook/post_installation.yml --tags nexus
  ansible-playbook -i inventory/cloud_host playbook/post_installation.yml --tags jenkins
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
# FRONTEND PART
mkdir -p cloud-native-demo && cd cloud-native-demo
echo "Use launcher to download Frontend"
cp ~/Downloads/booster-demo-frontend-spring-boot.zip .
unzip booster-demo-frontend-spring-boot.zip
idea .
cd booster-demo-frontend-spring-boot
mvn clean compile
mvn clean spring-boot:run 
open http://localhost:8090

oc new-project cloud-demo
mvn package fabric8:deploy -Popenshift
...
export FRONTEND=$(oc get route/cloud-native-frontend --template '{{.spec.host}}') 
open http://$FRONTEND

# SERVICE CATALOG PART
See HOL instructions

# BACKEND part
cp ~/Downloads/booster-demo-backend-spring-boot.zip .
unzip ~/Downloads/booster-demo-backend-spring-boot.zip
cd booster-demo-backend-spring-boot
mvn clean spring-boot:run -Ph2 -Drun.arguments="--spring.profiles.active=local,--jaeger.sender=http://jaeger-collector-tracing.192.168.64.85.nip.io/api/traces,--jaeger.protocol=HTTP,--jaeger.port=0"
curl -k http://localhost:8080/api/notes 
curl -k -H "Content-Type: application/json" -X POST -d '{"title":"My first note","content":"Spring Boot is awesome!"}' http://localhost:8080/api/notes 
curl -k http://localhost:8080/api/notes/1
...

oc new-app -f openshift/cloud-native-demo_backend_template.yml
oc start-build cloud-native-backend-s2i --from-dir=. --follow
...

apply the secret to the backend app and wait

Next, play with the app using the Frontend
```

## Update Catalog

Procedure used to recreate the catalog using latest version supported by the `fabric8-backend`

- Update mission catalog

```bash
cd catalog
git rm -r * 
git commit -m 'Delete all the stuff'
git push   
wget https://github.com/fabric8-launcher/launcher-booster-catalog/archive/v36.tar.gz
tar xvfz *.tar.gz -C ./
mv launcher-booster-catalog-36/metadata.yaml .
mv launcher-booster-catalog-36//{fuse,nodejs,spring-boot,vert.x,wildfly-swarm} .
rm -rf launcher-booster-catalog-*
rm *.tar.gz
```

- Amend `metadata.yaml` file to include our boosters

```yaml
missions:
- id: demo-frontend
  name: Cloud Native Demo - Frontend
  description: Cloud Native Frontend Spring Boot Application
  metadata:
    level: advanced
- id: demo-backend
  name: Cloud Native Demo - Backend
  description: Cloud Native Backend Spring Boot Application
  metadata:
    level: advanced
```

- Add project's folders to the catalog

```bash
cp -r demo-backend ../catalog/spring-boot/current-community
cp -r demo-frontend ../catalog/spring-boot/current-community
```

- Commit modifications

```bash
git add .
git commit -m "Update Catalog" -a
git push
```



  