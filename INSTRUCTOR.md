# Instructions to demo the Lab

## Locally

The instructions defined hereafter are a summary about the official documentation that you can find from the `openshift-infra` project

- [Openshift Installation](https://github.com/snowdrop/openshift-infra/tree/3.9.0.SP2#openshift-deployment)
- [Post installation steps](https://github.com/snowdrop/openshift-infra/blob/3.9.0.SP2/ansible/README-post-installation.md)

and assumes the following prerequesites 

**Prerequisites** :
- [Ansible - 2.4](http://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
- [Virtualbox](https://www.virtualbox.org/wiki/Downloads)
- [Centos VM](https://github.com/snowdrop/openshift-infra/tree/3.9.0.SP2#option-2--local---customized-linux-vm)

**Instructions**

- Git clone the `openshift-infra` project and checkout the lastest release avaialble for OpenShift (e.g : `3.9-SP2`)

```bash
git clone -b 3.9.0.SP2 https://github.com/snowdrop/openshift-infra.git
cd openshift-infra/ansible
```

- Git clone the `openshift-ansible` project and checkout the release corresponding to the version used of OpenShift (E.g : `release-3.9`) 

```bash
git clone -b release-3.9 https://github.com/openshift/openshift-ansible.git
```

- Generate the Ansible inventory to access your `local` Linux VM

```bash
ansible-playbook playbook/generate_inventory.yml -e ip_address=192.168.99.50
```

- Check if your VM matches the `prerequesites`

```bash
ansible-playbook -i inventory/cloud_host openshift-ansible/playbooks/prerequisites.yml
```

- Install OpenShift

```bash
ansible-playbook -i inventory/cloud_host openshift-ansible/playbooks/deploy_cluster.yml
```

**Post installation**

In order to play with the Lab, we will setup the persistence, grant cluster-role to the admin user and install the Ansible Service Broker

```bash
ansible-playbook -i inventory/cloud_host playbook/post_installation.yml -e openshift_admin_pwd=admin --tags "enable_cluster_admin"
ansible-playbook -i inventory/cloud_host playbook/post_installation.yml --tags persistence 
ansible-playbook -i inventory/cloud_host openshift-ansible/playbooks/openshift-service-catalog/config.yml
```