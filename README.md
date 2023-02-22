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
	- If set to TRUE, this role will install docker on management_server and tag & push the docker image to local-registry
	- If set to FALSE, the execution of all the tasks in this role will be skipped.
- Upgrade Container Runtime
	
- Upgrade Kubeadm Cluster
	- This role sets up High Availabilty cluster.
	- This role can also be used to set up single-master K8s-cluster.
- Upgrade Cluster Components
	- This role installs the optional cluster components such as metallb, nginx-ingress.
	- If set to FALSE in variable file, the exectuion of the task will be skipped

	

Inventory
----------------------------------------------------------------------------
The inventory will differ for each cluster.
The inventory file is located in the inventory/<cluster_name> directory with the name "hosts".
~~~
Example:
	For GDA-PTH01, the inventory file is located at inventory/GDAPH01/hosts

~~~
For GDA in PTH01 the inventory hosts file contains following entries for servers.

~~~
[kube_worker]
KW01
KW02
KW03
KW04

[kube_master_main]
KM01

[kube_master_backup]
KM02
KM03

[deploy_server]
MG01

[db_server]
DB01
DB02
DB03

** Please note that the nomenclature of the above should not be changed with actual hostname.
** The correct IP address should be present in /etc/hosts file for each server.
** For single master K8s cluster, the kube_master_backup group node should not have any entries.

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
files_dir: /ansible/fusion-caas-cluster-installation
# This is the name of parent directory where all ansible files/folders exists.

images_dir: "{{ files_dir }}/images"
# This directory under the files_dir contains all docker images to be uploaded on local docker registry

backup_dir: "{{ files_dir }}/backup"
# This directory under the files_dir contains backup of some existing configurations (if any) - specially IP related

cert_dir: "{{ files_dir }}/files"
# This directory under the files_dir contains certificates for HTTPS connection with docker registry and some other manifest/configuration files

logs_dir: "{{ files_dir }}/logs"
# This directory under the files_dir contains files with some error print during execution of playbooks.

source_crt_files_os: 
# This variable contains the name of the OS certificates files used in cert_dir

source_crt_files_docker: 
# This variable contains the name of CA certificate file used for docker registry HTTPS connection.

tag_and_push_images: no
# This instructs playbook to tag and push docker images to docker registry brfore K8s cluster setup {yes/no}

pre_install_reset_cluster: no
# This instructs playbook to reset the cluster if already present before fresh K8s setup {yes/no}

setup_docker_repo: yes
# This instructs playbook to create apt source list for docker/containerd installation in the directory /etc/apt/sources.list.d/

setup_kubernetes_repo: yes
# This instructs playbook to create apt source list in the directory /etc/apt/sources.list.d/ for kubeadm installation {yes/no}

setup_apt_local_repo: yes
# This instructs playbook to create apt source list in the directory /etc/apt/sources.list.d/ for ubuntu core packages installation {yes/no}

install_container_runtime: yes
# This instructs playbook to install container runtime (docker/containerd) before K8s cluster setup {yes/no}

setup_kubeadm_cluster: yes
# This instructs playbook to install kubeadm and setup new K8s cluster {yes/no}

install_metallb: no
# This instructs playbook to install metallb load_balancer after K8s cluster setup {yes/no}

install_calico: yes
# This instructs playbook to install calico cni after K8s cluster setup {yes/no}

install_ingress_nginx: no
# This instructs playbook to install ingress_nginx after K8s cluster setup {yes/no}

create_rbac_users: yes
# This instructs playbook to create addition RBAC user for K8s cluster defined in group_vars/all/users.yml {yes/no}

install_helm: yes
# This instructs playbook to install helm on management server {yes/no}

install_nfs_client: yes
# This instructs playbook to install nfs-client on master and worker nodes {yes/no}

install_kubectl_on_deploy_server: yes
# This instructs playbook to install kubectl on management server and copy kube_config file {yes/no}

container_runtime: options to install docker or containerd as container runtime {docker/containerd}

ha_cluster: yes
# This flag instructs playbook to setup multi-master k8s cluster {yes/no}. 'no' value will setup single master k8s cluster

ext_load_balancer: no
# This flag instructs playbook to use external load balancer if ha_cluster is set to 'yes' {yes/no}

multi_ext_load_balancer: no
# This flag instructs playbook to use multiple external load balancer if ha_cluster is set to 'yes' and exl_load_balancer is set to 'yes' {yes/no}

LB_pod_deployment: no
#This flag instructs playbook to user ha-proxy & keepalived as pod if ha_cluster is set to 'yes' {yes/no}

registry_secret_name: docker-nexus-registry
# name of the kubernetes secret to access docker-registry. If doesn't exists already, playbook will create it.


apt_repo_protocol: {https/http}
k8s_repo_protocol: {https/http}
docker_repo_protocol: {https/http}
local_repo_protocol: {https/http}
image_registry_protocol: {https/http}

# The value {https/http} in the above variable is used to connect with repository/registry



dnslookup_apt_repo: no
dnslookup_k8s_repo: no
dnslookup_docker_repo: no
dnslookup_local_repo: no
dnslookup_image_registry: yes

# The flag {yes/no} in the above variable is used by ansible playbook for dns lookup for repository/registry hostname. 'no' value will not go for dns-lookup and will create a local entry in /etc/hosts.

~~~

variables@group_vars/all/users.yml
~~~
additional_users:
# list of users for K8s cluster along with role. If users are not present already, playbook will create it.
~~~

variables@group_vars/all/version.yml
~~~
ansible_python_interpreter: /usr/bin/python3
# instructs playbook to use python3 interpreter.

ubuntu_distribution: 'focal'
# this value is used to create apt source list in /etc/apt/sources.list.d/ {focal/bionic}

kubeadm_setup_version: '1.24.3'
# kubeadm/kubectl/kubelet version to be used. change the images name and version in the kubernetes_images variable accordingly.

cri_dockerd_ver: '0.2.5'
# version of cri_dockerd container runtime

docker_ver: '20.10.17'
# version of docker container runtime

crictl_version: "v1.24.1"
# version of crictl when containerd is used as container runtime.

helm_version: "v3.9.2"
#version of helm installer

kubernetes_images:
# kubernetes images name and version. match with kubeadm_setup_version variable. Mandatory when tag_and_push_images is set to 'yes'

coredns_extra_tag:
# coredns images name and version. match with kubeadm_setup_version variable. Mandatory when tag_and_push_images is set to 'yes'

pod_infra_container_image:
# pause image version to be used by containerd for k8s image pull and setup. match with kubeadm_setup_version variable


calico_cni_images:
# calico images name and version.

metallb_images:
# metallb images name and version.

ingress_nginx_images:
# nginx_ingress images name and version.

haproxy_images:
# haproxy images name and version.

keepalived_images:
# keepalived images name and version.

~~~

variables@inventory/<env_name>/group_vars/all/haproxy_lb.yml
~~~
This variable file is used for haproxy and keepalived configuration. Mandatory when ha_cluster is set to 'yes'

LOAD_BALANCER_IP: "10.61.20.210"
# this is the IP of virtual interface in the same subnet of OAM

LOAD_BALANCER_CIDR: 27
# cidr of OAM subnet

LOAD_BALANCER_DNS: "10.61.20.210"
# either FQDN or IP address of virtual interface. If FQDN is used, it must be resolved in /etc/hosts file or local dns.

LOAD_BALANCER_PORT: 8443
# port for apiserver. This should be different from 6443

LOAD_BALANCER_INTERFACE: "ens160"
# interface name on which virtual interface is created by keepalived. OAM interface

HOST1_ID: "GDAPH01-KM01"
# hostname of master-1

HOST2_ID: "GDAPH01-KM02"
# hostname of master-2

HOST3_ID: "GDAPH01-KM03"
# hostname of master-3

HOST1_ADDRESS: "10.61.20.196"
# OAM physical IP of master-1

HOST2_ADDRESS: "10.61.20.197"
# OAM physical IP of master-2

HOST3_ADDRESS: "10.61.20.198"
# OAM physical IP of master-3

~~~

variables@inventory/<env_name>/group_vars/all/repository.yml
~~~
This variable file is used for local repository connectivity

apt_repo_ipv4: 10.61.20.207
# IP address of local repository

apt_repo_hostname: local-repository
# hostname/fqdn of local-repository

apt_repo_url: local-repository:6001
# url of local-repository excluding http/https

Similarly, there are variable for docker repo, kubernetes repo and all other repos


~~~

variables@inventory/<env_name>/group_vars/host_vars/{KM01.yml|KM02.yml|KM03.yml}
~~~
apiserver:
# IP address of OAM interface
~~~



Vault
----------------------------------------------------------------------------
The vault contains passwords to be used during task execution by ansible.
~~~
To edit the file:
	ansible-vault edit vault/secrets.yml
	Password: <provide vault password here to edit the file>
~~~

Tasks Sequence
----------------------------------------------------------------------------
~~~
The following tasks are executed by playbook.

Role: update-source-repository
	1. Create source repo list on all nodes for docker/containerd and kubernetes installation.
	2. Check for any missing package for docker/containerd, kubernetes and all other dependencies installation.
	3. If there are any missing packages, the playbook interrupts and generates a file containing lists of all missing packages.
Role: setup-container-runtime
	4. Install docker on all master & worker nodes (only if container_runtime=cri-dockerd)
	5. Install cri-dockerd on all master & worker nodes (only if container_runtime=cri-dockerd)
	6. Install containerd and configure all parameters required to run kubernetes cluster (only if container_runtime=containerd)
Role: update-image-registry (optional)
	7. Installs docker on management server
	8. Loads the docker images from the fusion-caas-cluster-installation/files/<sub-directories> directories
	9. Tag & Push images to the local image registry
Role: setup-kubeadm-cluster
	10. Installs HA-proxy & Keepalived on all the master nodes
	11. Installs kubeadm, kubectl, kubelet on all master & worker node
	12. Installs kubeclt on management server
	13. Kubeadm reset if already K8s cluster is present with running/failed state.
	14. Initialize Kubeadm cluster on master-1
	15. Installs Calico CNI in the cluster
	16. Create docker secrets in kube-system namespace and patch into the service-account for coredns, kube-proxy, and calico CNI
	17. Join master-2 & master-3 node in the cluster
	18. Join all the worker nodes in the cluster
	19. Waits for all nodes to be in Ready state & Core Pods to be in Running state
Role: setup-cluster-components (optional)
	20. Create namespace and service account for metallb
	21. Create docker secrets in metallb-system namespace and patch into the service-account.
	22. Install metallb through manifest from local image-registry
	23. Create namespace and service account for nginx-ingress
	24. Create docker secrets in ingress-nginx namespace and patch into the service-account.
	25. Install nginx-ingress through manifest from local image-registry

Role: implement-user-rbac
	26. Create OS user with basic OS privileges (if not present)
	27. Create Certificates and kube_config file for each user and copy accross all master node & management_server
	28. Create context for each user
	29. Create Role & Role-binding for each user
Role: client-and-package
	30. Install helm installer on Management server
	31. Install nfs-client (optional)
	
~~~


Tasks Descriptions
------------------------------------------------------------------------------------------
Present in each Role README


Cluster Setup Procedure
------------------------------------------------------------------------------------------
1. On the management_server create the following directory
	~~~
	/ansible/fusion-caas-cluster-installation
	/ansible/fusion-caas-cluster-installation/files/
	/ansible/fusion-caas-cluster-installation/logs/
	/ansible/fusion-caas-cluster-installation/images/
	/ansible/fusion-caas-cluster-installation/images/calico
	/ansible/fusion-caas-cluster-installation/images/haproxy
	/ansible/fusion-caas-cluster-installation/images/keepalived
	/ansible/fusion-caas-cluster-installation/images/ingress-nginx
	/ansible/fusion-caas-cluster-installation/images/kubernetes/
	/ansible/fusion-caas-cluster-installation/images/metallb
	~~~
		
	~~~
	sudo mkdir -p /ansible/fusion-caas-cluster-installation/files/
	sudo mkdir -p /ansible/fusion-caas-cluster-installation/logs
	sudo mkdir -p /ansible/fusion-caas-cluster-installation/images/calico
	sudo mkdir -p /ansible/fusion-caas-cluster-installation/images/haproxy
	sudo mkdir -p /ansible/fusion-caas-cluster-installation/images/keepalived
	sudo mkdir -p /ansible/fusion-caas-cluster-installation/images/ingress-nginx
	sudo mkdir -p /ansible/fusion-caas-cluster-installation/images/kubernetes/
	sudo mkdir -p /ansible/fusion-caas-cluster-installation/images/metallb
	
	sudo chown -R gda-admin:gda-admin /ansible/fusion-caas-cluster-installation
	~~~
	
2. Copy all the files and directories from this repository to /ansible/fusion-caas-cluster-installation directory

3. Create the OS certificates - VodafoneInternalRootCA.crt, VodafoneInternalCA.crt, ca.crt 
   and place them in /ansible/fusion-caas-cluster-installation/files directory.
   Refer: https://confluence.sp.vodafone.com/x/PGaZE
   
4. Put the docker images.tar file in the kubernetes, calico, metallb, haproxy, keepalived, ingress-nginx directory respectively (optional)
   This is not needed if images are already present in the image registry.


5. Update the hosts inventory at inventory/<cluster_name>/hosts

6. Change the environment specific parameters in variable files inventory/<cluster_name>/group_vars/all/haproxy_lb.yml
	~~~

	LOAD_BALANCER_IP:
	LOAD_BALANCER_CIDR:
	LOAD_BALANCER_DNS:
	LOAD_BALANCER_PORT:
	LOAD_BALANCER_INTERFACE:


	HOST1_ID:
	HOST2_ID:
	HOST3_ID:
	HOST1_ADDRESS:
	HOST2_ADDRESS:
	HOST3_ADDRESS:

        ~~~

7. Change the environment specific parameters in variable files inventory/<cluster_name>/group_vars/all/repository.yml
        ~~~

	repo_ipv4: 10.61.20.207              # local apt repository IP
	repo_hostname: local-repository      # local apt repository hostname
	repo_url: local-repository:6001      # local apt repository url

        *** change it for apt repository, k8s repository, docker repository and local repository

	

	image_registry_ipv4:
	image_registry_hostname:
	image_registry_url:

	~~~
	
7. Change the host specific parameters in variable files inventory/<cluster_name>/host_vars/[KM01.yml,KM02.yml,KM03.yml]
	~~~
	apiserver: 10.61.20.196       # OAM IP of the own node
	~~~

8. You may change the install options (if needed), or leave it with default values in group_vars/all/main.yml
	~~~
	tag_and_push_images: no
	pre_install_reset_cluster: no
	setup_docker_repo: yes
	setup_kubernetes_repo: yes
	setup_apt_local_repo: yes
	install_container_runtime: yes
	setup_kubeadm_cluster: yes
	install_metallb: no
	install_calico: yes
	install_ingress_nginx: no
	create_rbac_users: yes
	install_helm: yes
	install_nfs_client: yes
	~~~
	
9. Verify the protocols for local-repository and image-registry in the variable file group_vars/all/main.yml
	~~~
	repo_protocol: http
	image_registry_protocol: https
	~~~
	
10. Execute the playbook
	~~~
	ansible-playbook k8s-setup.yml -i inventory/<cluster_name> -e "@vault/secrets.yml" --ask-vault-pass
	password: <provide the vault password here>

	Example:
	ansible-playbook k8s-setup.yml -i inventory/gdaph01 -e "@vault/secrets.yml" --ask-vault-pass
	~~~


Netplan Implementation [Ansible]
============================================================================
Description is present in the role's README file.
Role: update-netplan
