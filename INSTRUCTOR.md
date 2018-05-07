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

See [Hands On Lab - objectives](HANDS_ON_LAB.md)

  