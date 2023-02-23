K8s Cluster Upgrade [Ansible]
============================================================================

Description
----------------------------------------------------------------------------
The objective of this ansible-playbook is to upgrade the kubernetes cluster from 1.18 to 1.23.
The playbook has 5 roles with some pre-tasks and post-tasks.


Roles:
- Update Source Repository
	- This role creates local source repo for docker/containerd, kubernetes and its dependencies installation.
	- Then it checks for any missing packages in the local-repository.
	- If any of the package is missing in the local apt repository, the playbook is interrupted generating a file containing missing package name.
- Update image registry
	- This role is optional.
	- If set to TRUE, this role will install docker on management_server(db_server in case of OCD/LRF-RDF and tag & push the docker image to local-registry
	- If set to FALSE, the execution of all the tasks in this role will be skipped.
- Upgrade Container Runtime
	- This role upgrades the existing container runtime, either docker or containerd (depends on selecyion in variable file)
	- This role can also migrate container runtime from docker to containerd if migrate_to_containerd is set to true.	
- Upgrade Kubeadm Cluster
	- This role upgrades the kubeadm cluster from current version 1.18 to 1.23
	- This role also sense the current version of the cluster and applies the upgrade from current to 1.23
- Upgrade Cluster Components
	- This role upgardes the cluster component metallb if upgrade_metallb is set to true
	- This role upgardes the cluster component calico if upgrade_calico is set to true

	

Inventory
----------------------------------------------------------------------------
The inventory will differ for each cluster.
The inventory file is located in the inventory/<cluster_name> directory with the name "hosts".
~~~
Example:
	For OCD-PTH01, the inventory file is located at inventory/ocdph01/hosts

~~~
For OCD in PTH01 the inventory hosts file contains following entries for servers.

~~~
[kubernetes:children]
kube_worker
kube_master

[kube_worker]
OCDPH01-W01
OCDPH01-W02

[kube_master]
OCDPH01-M01

[db_server]
OCDPH01-DB01

[local_repository]
OCDPH01-DB01

[local_image_registry]
OCDPH01-DB01

** Please note that the nomenclature of the above should not be changed with actual hostname.
** The correct IP address should be present in /etc/hosts file for each server.

~~~

Variables
----------------------------------------------------------------------------
There are 3 main location of variable files
~~~
1. group_vars/all/ ----> mainly, for those parameters whose values are likely to be same accross all environment

2. inventory/<cluster>/group_vars/all/ ---> mainly, for those parameters whose values are likely to be different for each environment.
   inventory/<cluster>/host_vars/<inventory_hostname> ---> host specific variables and likely to be different for each environment.
   
3. roles/vars/main.yaml ---> This variable files contains parameters for developers.
	
~~~

variables@group_vars/all/main.yml
~~~

logs_dir: /tmp
# The directory where ansible playbook error logs are stored.

images_dir: /tmp/kubernetes-upgrade
# This directory under the /tmp contains all docker images to be uploaded on local docker registry

tag_and_push_images: yes
# All container images are pulled from the local registry during upgrades. Push images to local registry if not already done. Select [yes/no]

migrate_to_containerd: no
# Docker is not supported as container-runtime in K8s version 1.24 and later. Migrate from Docker to containerd container-runtime. Select [yes/no]

container_runtime: docker
# Select the container runtime to upgrade

upgrade_container_runtime: yes
# Existing container runtime is upgraded is the value is set to YES


upgrade_calico: yes
# Existing cni calico is upgraded is the value is set to YES

upgrade_metallb: yes
# Existing load balancer metallb is upgraded is the value is set to YES




~~~

variables@group_vars/all/version.yml
~~~

ansible_python_interpreter: /usr/bin/python3
# instructs playbook to use python3 interpreter.

kubeadm_upgrade_sequence:
# Kubeadm cluster can be upgraded to the next minor version only. Please enter the sequence of upgrade.

upgrade_calico_version:
# Enter the calico version to upgrade

upgrade_metallb_version: 0.12.1
# Enter the metallb version to upgrade

kubernetes_images:
# Enter the list of all docker images for K8s upgrade to be pushed to local registry. Use the correct format only. Refer to the variable file.

calico_cni_images:
# Enter the list of all docker images for calico upgrade to be pushed to local registry. Use the correct format only. Refer to the variable file.

metallb_images:
# Enter the list of all docker images for metallb upgrade to be pushed to local registry. Use the correct format only. Refer to the variable file.

~~~

Vault
----------------------------------------------------------------------------
The vault contains passwords to be used during task execution by ansible.
~~~
To edit the file:
	ansible-vault edit vault/secrets.yml
	Password: <provide vault password here to edit the file>
~~~


Execute the playbook
	~~~
	ansible-playbook upgrade-k8s-cluster.yml -i inventory/<cluster_name> -e "@vault/secrets.yml" --ask-vault-pass
	password: <provide the vault password here>

	Example:
	ansible-playbook upgrade-k8s-cluster.yml -i inventory/gdaph01 -e "@vault/secrets.yml" --ask-vault-pass
	~~~


