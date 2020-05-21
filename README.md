
### kubernetes binary 部署方式(使用ansible playbook 部署)
> 该项目由原来的k8s部署playbook(https://github.com/lmdkfs/paas-playbooks/tree/master/k8s) 迁移过来
### 环境说明

- Centos 7
- Ansible(2.8及以上)
- Repository 仓库(用来提供给ansible 下载二进制文件使用, Repository 仓库参考[repo 部署脚本](https://github.com/lmdkfs/paas-playbooks/tree/master/repository))

> 由于flannel不支持etcd 3.4 所以部署etcd的时候需要使用3.4以下版本

### 仓库目录结构

```python
# 将相关的安装二进制文件 拷贝到目录
repo/
├── etcd
│   ├── v3.3.21
│   │   ├── etcd
│   │   └── etcdctl
│   └── v3.4.7
│       ├── etcd
│       └── etcdctl
├── flannel
│   └── v0.12.0
│       ├── flanneld
│       └── mk-docker-opts.sh
├── index.html
└── kubernetes
    └── v1.18.2
        └── server
            └── bin
                ├── kube-apiserver
                ├── kube-controller-manager
                ├── kube-proxy
                ├── kube-scheduler
                ├── kubectl
                └── kubelet

9 directories, 13 files
```

> 二进制文件请自行去官网下载
===================================

### 项目结构说明

项目目录
```python
├── files
│   ├── config
│   └── ssl
├── inventories
│   └── production
├── roles
│   ├── docker
│   ├── etcd
│   ├── flannel
│   ├── gen-ssl
│   ├── init-server
│   ├── k8s-master
│   ├── kubelet-kubeproxy
│   └── test
├── README.md
├── etcd.yml
├── gen-ssl.yml
├── init-server.yml
├── k8s-master.yml
└── k8s-node.yml

```

- files: files/ssl 用来存放证书文件, files/config用来存放配置文件 
- inventories: inventories/production/k8s.yaml 是主机信息,inventories/production/group_vars/all.yml 环境变量(*需要修改all.yaml*进行配置)

- etcd.yml, gen-ssl.yml, init-server.yml, k8s-master.yml, k8s-node.yml: 里面的hosts 变量匹配 `inventories/production/group_vars/all.yml`的hosts组来进行 部署

### 环境变量说明

inventories/production/group_vars/all.yml

```python
repo_address: http://172.16.26.1:88
KUBE_APISERVER: 172.16.26.156

etcd_hosts:
  - name: etcd01
    ip: 172.16.26.156
  - name: etcd02
    ip: 172.16.26.157
  - name: etcd03
    ip: 172.16.26.158

etcd_cluster_data_dir: /data/etcd-data
etcd_version: v3.3.21
network: 172.68.0.0/16

kube_api_cluster_service_ip: 10.254.0.1
BOOTSTRAP_TOKEN: 0f7b180f3712121979fb8a5981cde7a2
gen_ssl_path: /data/ssl

project_config_path: /etc/kubernetes
project_bin_path: /usr/local/bin

NODE_ADDRESS: "{{ ansible_default_ipv4.address }}"


DNS_SERVER_IP: 10.254.0.2
kube_api_cluster_service_ip: 10.254.0.1
service_cluster_ip_range: 10.254.0.0/16
k8s_version: v1.18.2

# flannel
flannel_version: v0.12.0

infra_container_image: registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.2
```

### 部署步骤

#### 第一步 初始化系统

`ansible-playbook -i inventories/production init-server.yml -u root -k`

#### 第二步 生成ssl 证书

`ansible-playbook -i inventories/production gen-ssl.yml -u root -k`

#### 第三步 安装etcd

`ansible-playbook -i inventories/production etcd.yml -u root -k`

#### 第四步 安装 k8s-master

`ansible-playbook -i inventories/production k8s-master.yml -u root -k`

#### 第五步 安装kube-node

`ansible-playbook -i inventories/production k8s-node.yml -u root -k`

#### 检查

```python
[root@k8s-master ~]# kubectl  get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
etcd-1               Healthy   {"health":"true"}
etcd-2               Healthy   {"health":"true"}
etcd-0               Healthy   {"health":"true"}
controller-manager   Healthy   ok

[root@k8s-master ~]# kubectl  get nodes
NAME            STATUS   ROLES    AGE   VERSION
172.16.26.157   Ready    <none>   96m   v1.18.2
172.16.26.158   Ready    <none>   96m   v1.18.2
```

 

