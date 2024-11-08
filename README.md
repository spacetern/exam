# exam
用来考试
【mission1】
1、
控制节点：
setenforce 0 
systemctl stop firewalld.service 
 hostnamectl set-hostname controller
 bash

计算节点：
setenforce 0 
systemctl stop firewalld.service 
 hostnamectl set-hostname compute
 bash

2、
控制节点
 mount -o loop private-cloud-antelope-v1.0.iso /mnt/
 cp -rvf /mnt/* /opt/
 umount /mnt/
 cd /etc/yum.repos.d/
[root@controller yum.repos.d]# cp openEuler.repo openEuler.repo.bak
[root@controller yum.repos.d]# vi openEuler.repo

#添加下面内容
[PRIVATE-CLOUD]
name=PRIVATE-CLOUD
baseurl=file:///opt/PRIVATE-CLOUD/antelope-repo
enabled=1
gpgcheck=0   

按esc
：wq退出

 yum install vsftpd -y
 vi /etc/vsftpd/vsftpd.conf
#在第一行添加
anon_root=/opt/
#修改
anonymous_enable=YES
按esc
：wq退出

 systemctl restart vsftpd


计算节点
 cd /etc/yum.repos.d/
[root@compute yum.repos.d]# cp openEuler.repo openEuler.repo.bak
[root@compute yum.repos.d]# vi openEuler.repo
添加下面内容
[PRIVATE-CLOUD]
name=PRIVATE-CLOUD
baseurl=ftp://192.168.1.140/PRIVATE-CLOUD/antelope-repo
# 192.168.1.140这个IP是控制节点的IP
enabled=1
gpgcheck=0
按esc
：wq退出
 dnf clean all 
dnf makecache

3、
 yum install private-cloud
  iaas-install-kolla-ansible-environment.sh

 vi /etc/ansible/openstack_cluster
[all:vars]
ansible_connection=ssh
ansible_port=22
ansible_user=root 
ansible_password=qazWSX@1234
ansible_shell_type=sh 
ansible_python_interpreter='/usr/bin/env python3'
[control]
192.168.1.140
[network]
192.168.1.140
[compute]
192.168.1.141
[monitoring]
192.168.1.140
[storage]
192.168.1.141
：wq退出
  ansible all -m ping 

4、
 kolla-genpwd 
 grep keystone_admin /etc/kolla/passwords.yml 
 vi /etc/kolla/passwords.yml
# 修改
keystone_admin_password: qazWSX@1234
：wq

5、
 vi /etc/kolla/globals.yml 
openstack_tag: "antelope"
docker_registry: "<控制节点 IP>:4000"
docker_registry_insecure: "yes"
docker_namespace: "private.cloud"
:wq 

6、
计算节点
[root@compute ~]# dnf install python3-libselinux -y
控制节点
 kolla-ansible bootstrap-servers 
 docker network create registry 
 gunzip -c /opt/PRIVATE-CLOUD/images/registry.tar.gz | docker load 
 cd /opt/PRIVATE-CLOUD/tools/registry 
 docker compose up -d


7、
控制节点
#进行部署前检查（此步骤非必需，可忽略执行）。 
 kolla-ansible prechecks 
#下载各个节点所需要的全部镜像。 
 kolla-ansible pull 
#部署OpenStack集群。 
 kolla-ansible deploy 
 kolla-ansible post-deploy 
ansible-playbook /etc/private-cloud/openstack_client.yaml 
cat /etc/kolla/admin-openrc.sh 



# 如果是自己搭的环境要改这些
[root@controller ~]# grep -Ev "^$|^#" /etc/kolla/globals.yml 
---
workaround_ansible_issue_8743: yes
kolla_base_distro: "debian"
openstack_release: "2023.1"
openstack_tag: "antelope"
kolla_internal_vip_address: "192.168.1.140"
docker_registry: "192.168.1.140:4000"
docker_registry_insecure: "yes"
docker_namespace: "private.cloud"
network_interface: "ens33"
neutron_external_interface: "ens34"
openstack_region_name: "RegionTyh"
enable_haproxy: "no"
enable_neutron_vpnaas: "yes"
enable_neutron_qos: "yes"
enable_neutron_bgp_dragent: "yes"
enable_neutron_provider_networks: "yes"
enable_skyline: "yes"
nova_compute_virt_type: "qemu"



【mission2】
Keystone认证服务运维管理

[root@controller kolla]#    source admin-openrc.sh 
[root@controller kolla]# openstack project list
1、
# 加载管理员凭证文件
source admin-openrc.sh
# 创建租户 A
openstack project create  --description "Tenant A" A
# 创建租户 B
openstack project create  --description "Tenant B" B
# 创建用户 userA
openstack user create --domain default --password 123456 userA
# 创建用户 userB
openstack user create --domain default --password 123456 userB
vopenstack user list
# 将 userA 分配到租户 A 并赋予 member 角色
openstack role add --project A --user userA member
# 将 userB 分配到租户 B 并赋予 member 角色
openstack role add --project B --user userB member

2、
Glance镜像服务运维管理
openstack image create \
  --file noble-server-cloudimg-amd64.img \
  --disk-format qcow2 \
  --container-format bare \
  --min-ram 1024 \
  --min-disk 10 \
  "cirros-0.6.2"

  #--id 20241109 \
3、
Neutron网络服务运维管理
openstack network create --provider-network-type vxlan private-net
openstack subnet create --network private-net --subnet-range 10.10.10.0/24 --gateway 10.10.10.1 --dns-nameserver 8.8.8.8 private-subnet

4、
Nova计算服务运维管理
步骤 1: 创建实例类型
首先，创建一个名称为 cirros-flavor 的实例类型，其中 VCPU 数量为 1、内存大小为 1GB、根磁盘大小为 10GB。
openstack flavor create --vcpus 1 --ram 1024 --disk 10 cirros-flavor

步骤 2: 创建云主机
接下来，使用刚刚创建的实例类型 cirros-flavor、镜像 cirros-0.6.2 和网络 private-net 创建一台名为 GX-cirros 的云主机。

假设已经创建了名为 cirros-0.6.2 的镜像和名为 private-net 的网络，您可以使用以下命令创建云主机：

openstack server create --flavor cirros-flavor --image cirros-0.6.2 --nic net-id=private-net GX-cirros

5、
Heat编排服务运维管理
vi Heat-FlavorUltra.yaml

heat_template_version: 2016-10-14

description: >
  Heat template to create a custom flavor s6.xlarge.2 with vcpu=2, ram=4096, ephemeral=5, disk=20.

parameters:
  flavor_name:
    type: string
    default: s6.xlarge.2
    description: The name of the flavor to create.
  vcpus:
    type: number
    default: 2
    description: The number of virtual CPUs for the flavor.
  ram:
    type: number
    default: 4096
    description: The amount of RAM in MB for the flavor.
  ephemeral:
    type: number
    default: 5
    description: The size of the ephemeral disk in GB for the flavor.
  disk:
    type: number
    default: 20
    description: The size of the root disk in GB for the flavor.

resources:
  custom_flavor:
    type: OS::Nova::Flavor
    properties:
      name: { get_param: flavor_name }
      vcpus: { get_param: vcpus }
      ram: { get_param: ram }
      ephemeral: { get_param: ephemeral }
      disk: { get_param: disk }

outputs:
  flavor_id:
    description: The ID of the created flavor.
    value: { get_resource: custom_flavor }

    ：wq


vi FlavorUltraEnv.yaml
parameters:
  flavor_name: s6.xlarge.2
  vcpus: 2
  ram: 4096
  ephemeral: 5
  disk: 20
：wq退出

openstack stack create --template /root/Heat-FlavorUltra.yaml --environment /root/FlavorUltraEnv.yaml FlavorUltraStack

openstack stack show FlavorUltraStack

6、
Ansible管理OpenStack云资源
vi /root/create_flavor.yml
---
- name: Create a compute flavor using OpenStack
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Create the 'tiny' flavor
      openstack.cloud.compute_flavor:
        cloud: kolla-admin
        state: present
        name: tiny
        vcpus: 1
        ram: 1024
        disk: 10
        ephemeral: 10
        swap: 0
        rxtx_factor: 1.0
        is_public: true
        description: ansible-flavor
：wq退出

cd /root
# 可能直接运行会出错，如果出错就先安装openstacksdk
pip install openstacksdk==0.62
ansible-playbook create_flavor.yml
openstack --os-cloud kolla-admin flavor list



7、
MinIO对象存储
openstack server create --image openEuler-24.03-LTS --flavor 2V-4G-40G --nic net-id=public-net --security-group default --key-name your-keypair-name GX-openEuler

ssh -i /path/to/your-private-key.pem root@<cloud-host-ip>

wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/

mkdir -p /data/minio

export MINIO_ROOT_USER=admin
export MINIO_ROOT_PASSWORD=haobo@2025
nohup minio server /data/minio --console-address ":9001" &

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

aws configure


编辑 ~/.aws/credentials 文件，确保内容如下：
[default]
aws_access_key_id=admin
aws_secret_access_key=haobo@2025

编辑 ~/.aws/config 文件，确保内容如下：
[default]
region=us-east-1
output=json


使用 AWS CLI 创建存储桶
aws s3api create-bucket --bucket gxzg --endpoint-url http://localhost:9000

验证存储桶创建
aws s3 ls s3://gxzg --endpoint-url http://localhost:9000

【mission3】
1、调用OpenStackAPI自动化管理资源
步骤 1: 创建项目目录结构
在 /root 目录下创建项目目录结构：
mkdir -p /root/openstack/utils
mkdir -p /root/openstack/glance
步骤 2: 编写 openstack_connection.py 文件
在 utils 目录下创建 openstack_connection.py 文件，并编写以下内容：
# /root/openstack/utils/openstack_connection.py

import os_client_config

def get_openstack_connection(cloud_name='kolla-admin-internal'):
    """
    获取 OpenStack SDK 连接对象
    :param cloud_name: 云环境名称，默认为 'kolla-admin-internal'
    :return: OpenStack SDK 连接对象
    """
    return os_client_config.make_sdk(cloud=cloud_name)

步骤 3: 编写 openstack_logger.py 文件
在 utils 目录下创建 openstack_logger.py 文件，并编写以下内容：
# /root/openstack/utils/openstack_logger.py
import logging

def setup_logger():
    """
    设置日志记录器
    """
    logger = logging.getLogger('openstack_logger')
    logger.setLevel(logging.DEBUG)

    # 创建一个文件处理器，将日志记录到文件
    file_handler = logging.FileHandler('/var/log/openstack.log')
    file_handler.setLevel(logging.DEBUG)

    # 创建一个控制台处理器，将日志输出到控制台
    console_handler = logging.StreamHandler()
    console_handler.setLevel(logging.INFO)

    # 创建一个日志格式器
    formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
    file_handler.setFormatter(formatter)
    console_handler.setFormatter(formatter)

    # 将处理器添加到日志记录器
    logger.addHandler(file_handler)
    logger.addHandler(console_handler)

    return logger
步骤 4: 编写 create_image.py 文件
在 glance 目录下创建 create_image.py 文件，并编写以下内容：
# /root/openstack/glance/create_image.py

import sys
from utils.openstack_connection import get_openstack_connection
from utils.openstack_logger import setup_logger

logger = setup_logger()

def create_image(name, filename):
    """
    创建镜像
    :param name: 镜像名称
    :param filename: 镜像文件路径
    """
    try:
        # 获取 OpenStack 连接
        conn = get_openstack_connection()

        # 创建镜像
        image = conn.image.create_image(
            name=name,
            filename=filename,
            disk_format='qcow2',
            container_format='bare',
            visibility='private'
        )

        logger.info(f"Image '{name}' created successfully with ID: {image.id}")
    except Exception as e:
        logger.error(f"Failed to create image '{name}': {e}")

if __name__ == "__main__":
    if len(sys.argv) != 3:
        print("Usage: python create_image.py <image_name> <image_file>")
        sys.exit(1)

    image_name = sys.argv[1]
    image_file = sys.argv[2]

    create_image(image_name, image_file)
步骤 5: 运行脚本
确保你已经加载了管理员凭证文件，并且镜像文件 cirros-0.6.2-x86_64-disk.img 存在于适当的位置。然后运行 create_image.py 脚本：

python /root/openstack/glance/create_image.py cirros-openstacksdk /path/to/cirros-0.6.2-x86_64-disk.img




【K8s】
1、
控制节点
 hostnamectl set-hostname master
 bash
 工作节点
 hostnamectl set-hostname worker
 bash

 unzip haobo-paas-openEuler24.03-v1.0.zip -d /opt/
rpm -ivh /opt/haobo-paas/kubernetes-repo/packages/haobo-paas-1.0-1.x86_64.rpm

2、
控制节点
set-deploy-environment
vi /etc/haobo/kubernetes-v1.31.0.yaml
  - {name: master, address: 192.168.2.145, internalAddress: 192.168.1.145, user: root, password: "qazWSX@1234"}
  - {name: worker, address: 192.168.2.146, internalAddress: 192.168.1.146, user: root, password: "qazWSX@1234"}
deploy-kubernetes-v1.30.1

3、
cd /opt/haobo-paas/images
gunzip -c master-images.tar.gz |ctr -n k8s.io  images import -
工作节点
 gunzip -c /root/worker-images.tar.gz |ctr -n k8s.io  images import -

控制节点
kubectl get pod -A
kubectl get nodes

echo 'source <(kubectl completion bash)>' >>~/.bashrc
unzip haobo-kubevirt-v1.3.1.zip -d /opt/
/opt/kubevirt-v1.3.1/bin/deploy-kubevirt-v1.3.1


1、
2、
3、
4、
