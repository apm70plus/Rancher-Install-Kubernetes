# Rancher-Install-Kubernetes
基于Rancher的REK（Rancher Kubernetes Engine），在Ubuntu16.04上实现简化安装kubernetes  
由于谷歌被墙的缘故，Kubernetes官网推荐的安装过程对于城墙内的人来说，就是一场噩梦。经过多方尝试，采用Rancher的REK方式安装K8S，简单高效，怎一个爽字了得。


## 1、环境准备

### 1.1 服务器或虚拟机（使用Ubuntu 16.04 Server版）  
| 角色        | 系统   |  主机名  |
| --------   | -----  | :----:  |
| rke (安装k8s) | Ubuntu 16.04 |  rke_server |
| master node  | Ubuntu 16.04 |  master1 |
| worker node  | Ubuntu 16.04 |  worker1 |
| worker node  | Ubuntu 16.04 |  worker2 |  

*注意：各主机的hostname主机名必须不同！可通过 /etc/hostname 和 /etc/hosts 文件修改；另外，/etc/hosts 配置所有节点的本地域名解析，例如：*

```
10.0.100.3  master1
10.0.100.4  worker1
10.0.100.5  worker2
```

### 1.2 安装Docker  
Rancher官方推荐版本 1.12.6，1.13.1，17.03.2，这三个版本都是官方测试通过的，我采用的17.03.2-ce版  

| Docker Version| Install Script  |
| --------   | -----  |
| 17.03.2  | curl https://releases.rancher.com/install-docker/17.03.sh \| sh |
| 1.13.1  | curl https://releases.rancher.com/install-docker/1.13.sh \| sh |
| 1.12.6  | curl https://releases.rancher.com/install-docker/1.12.sh \| sh |

### 1.3 设置防火墙策略  
开放集群主机节点之间6443、2379、2380端口，如果是刚开始试用，可以先关闭防火墙  
Ubuntu默认未启用UFW防火墙。若已经启用也可手工关闭：sudo ufw disable

### 1.4 禁用Swap  
一定要禁用swap，否则kubelet组件无法运行。  
1）永久禁用swap  
```
# 编辑fstab文件，注释掉swap项
vi /etc/fstab
```
2）临时禁用swap  
```
swapoff -a
```

### 1.5 启用cgroup
修改配置文件/etc/default/grub，启用cgroup内存限额功能,配置两个参数
```
GRUB_CMDLINE_LINUX_DEFAULT="cgroup_enable=memory swapaccount=1"
GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1"
```
执行sudo update-grub 更新grub

### 1.6 设置SSH免密登录  
RKE通过SSH tunnel进行安装部署，需要事先建立RKE到各节点的SSH免密登录。在RKE机器上执行秘钥生成命令 **ssh-keygen**（一路回车就行），并将生成的公钥通过命令 **ssh-copy-id {user}@{ip}** 分发到所以服务器节点。

1）在各个节点上创建ssh用户，并将其添加至docker组中 
```
adduser docker_user
usermod -aG docker docker_user
```

2）在rke所在主机上~/.ssh目录下创建ssh密钥  
```
ssh-keygen -t rsa
```

3）将生成密钥的公钥分发到各个节点
```
ssh-copy-id docker_user@{ip}
```

### 1.7 所有节点重启，使配置生效

## 2、安装Kubernetes
RKE主机采用ssh原理远程安装kubernetes集群，所以kubernetes节点不需要安装RKE。本章节所有操作只在RKE机器上执行

### 2.1 安装RKE
从Rancher的官方[GitHub](https://github.com/rancher/rke/releases/latest)仓库下载RKE。RKE可以在Linux和MacOS机器上运行。

```
# 下载后安装rke
mv rke_linux-amd64 rke
chmod +x rke
sudo mv rke /usr/local/bin/
```

### 2.2 集群配置文件  
默认情况下，RKE将查找名为cluster.yml的文件，该文件中包含有关将在服务器上运行的远程服务器和服务的信息。  可以参考Rancher官网的[配置模版](https://rancher.com/docs/rke/v0.1.x/en/example-yamls/)。  
集群配置文件包含一个节点列表。每个节点至少应包含以下值：

- 地址 – 服务器的SSH IP
- 用户 – 连接到服务器的SSH用户
- 角色 – 主机角色列表：worker，controlplane或etcd

以下是1个master,2个worker的最简模版配置:
```
nodes:
    - address: master1
      user: docker_user
      role:
        - controlplane
        - etcd
    - address: worker1
      user: docker_user
      role:
        - worker
    - address: worker2
      user: docker_user
      role:
        - worker
```
### 2.3 安装Kubernetes
cluster.yml文件所在目录下，运行REK命令，安装K8S。
*注意：当前版本的REK默认安装v1.11.3版本的K8S，如果需要指定版本，请参考Rancher官网[Kubernetes Version](https://rancher.com/docs/rke/v0.1.x/en/config-options/)一节来指定版本*
```
rek up
```
若想指向某个具体配置文件，运行如下命令：
```
rek up -f /etc/k8s/cluster.yml
```
### 2.4 安装网络插件
RKE是一个幂等工具，可以运行多次，且每次均产生相同的输出。如下的网络插件它均可以支持部署：

- Calico
- Flannel (default)
- Canal

要使用不同的网络插件，您可以在配置文件中指定（参考Rancher官网[Full cluster.yml example](https://rancher.com/docs/rke/v0.1.x/en/example-yamls/)）

```
network: 
    plugin: calico
```
### 2.5 安装Helm（k8s应用的包管理工具）
1）安装helm客户端  
```
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```
2）创建tiller的serviceaccount和clusterrolebinding  
```
kubectl -n kube-system create serviceaccount tiller
kubectl create clusterrolebinding tiller \
  --clusterrole cluster-admin \
  --serviceaccount=kube-system:tiller
```
3) 安装helm服务端tiller    
```
# 指定aliyun镜像（不用gcr.io/kubernetes-helm/tiller:v2.11.0的原因你懂的...）
helm init --service-account -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.11.0 tiller
```
### 2.6 高可用
RKE工具是满足高可用的。您可以在集群配置文件中指定多个控制面板主机，RKE将在其上部署主控组件。

默认情况下，kubelets被配置为连接到nginx-proxy服务的地址——127.0.0.1:6443，该代理会向所有主节点发送请求。

要启动HA集群，只需使用controlplane角色指定多个主机，然后正常启动集群即可。

## 3、测试验证
### 3.1 安装kubectl
1）采用的是二进制包方式安装，其它安装方式参考[官网](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl)

```
# 1. 下载指定版本（v1.11.3）的二进制文件
wget https://storage.googleapis.com/kubernetes-release/release/v1.11.3/bin/linux/amd64/kubectl
# 2. 修改为可执行文件
chmod +x ./kubectl
# 3. 移动到系统路径$PATH
sudo mv ./kubectl /usr/local/bin/kubectl
```

2）配置kubeconfig文件

RKE会在配置文件所在的目录下部署一个本地文件，该文件中包含kube配置信息以连接到新生成的群集。

默认情况下，kube配置文件被称为.kube_config_cluster.yml。将这个文件复制到你的本地~/.kube/config，就可以在本地使用kubectl了。

需要注意的是，部署的本地kube配置名称是和集群配置文件相关的。例如，如果您使用名为mycluster.yml的配置文件，则本地kube配置将被命名为.kube_config_mycluster.yml。

### 3.2 验证
执行kubectl命令，获取nodes的信息

```
kubectl get nodes
kubectl get pods --all-namespaces
kubectl get services --all-namespaces
```
## 4、发布服务
### 4.1 安装Web UI
1）下载kubernetes-dashboard.yaml文件
```
curl -LO https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

2）编辑kubernetes-dashboard.yaml文件，支持外部访问
在此文件中的Service部分下添加type: NodePort和nodePort: 30001
```
# ------------------- Dashboard Service ------------------- #

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001
  selector:
    k8s-app: kubernetes-dashboard
```

3）编辑kubernetes-dashboard.yaml文件。  
修改image为 rancher/kubernetes-dashboard-amd64:v1.8.3  

4）通过执行如下的命令部署Web UI
```
kubectl apply -f kubernetes-dashboard.yaml
```

### 4.2 访问Web UI
1）在浏览器中输入:https://{Master IP}:30001，提示录入登录凭证  
2）获取管理员用户的Token

通过执行如下命令获取系统Token信息：
```
kubectl describe  secret tiller --namespace=kube-system
```

3）导入Token

在界面中导入token,完成UI登录

## 参考资料
[Kubernetes-基于RKE进行Kubernetes的安装部署](https://www.kubernetes.org.cn/4004.html)  
[Rancher官网](https://rancher.com/docs/rancher/v2.x/en/installation/)
[Kubernets handbook](https://jimmysong.io/kubernetes-handbook/practice/helm.html)
