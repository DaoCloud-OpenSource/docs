## 目录
- [目录](#目录)
- [使用部署工具安装Kubernetes](#使用部署工具安装kubernetes)
  - [kubectl](#kubectl)
  - [Kind / kubeadm](#kind--kubeadm)
- [参考文档](#参考文档)




## 使用部署工具安装Kubernetes

### kubectl

- Linux：https://kubernetes.io/zh-cn/docs/tasks/tools/install-kubectl-linux/
- maxOS: https://kubernetes.io/zh-cn/docs/tasks/tools/install-kubectl-macos/
- Windows: https://kubernetes.io/zh-cn/docs/tasks/tools/install-kubectl-windows/

### Kind / kubeadm

1. **Kind**

   * 官方文档：https://kind.sigs.k8s.io/docs/user/quick-start/

   * **安装方法**
     1. 确保已安装 go 1.16+ 未安装请参见文档：https://go.dev/dl/
     2. 输入命令：``` go install sigs.k8s.io/kind@v0.20.0 ``` 

   * **使用方法**
     1. 创建集群：``` kind create cluster ```
     2. 与集群交互：使用kubectl（下载方式见上文）
     3. 删除集群：``` kind delete cluster ```
     4. 载入镜像：``` kind load docker-image my-custom-image-0 my-custom-image-1 ```
     5. 搭建镜像：``` kind build node-image /path/to/kubernetes/source ```

2. **kubeadm**

   * 官方安装文档：https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

   * 可行的使用方法帖子：https://developer.aliyun.com/article/927657

   * ***注意：最好保证 kubelet、kubeadm、kubectl 版本号一致***

   * **安装方法**
     1. 安装kubelet、kubeadm、kubectl：``` yum install kubelet-1.22.2 kubeadm-1.22.2 kubectl-1.22.2 -y ``` (建议使用更新的版本，1.22.2比较老了）
     2. 在Master上操作：
        ```
           #关闭防火墙
           systemctl stop firewalld
           systemctl disable firewalld

           #关闭selinux
           setenforce 0

           #关闭swap
           swapoff -a

           #将桥接的IPv4流量传递到iptables的链
           cat > /etc/sysctl.d/k8s.conf << EOF
           net.bridge.bridge-nf-call-ip6tables = 1
           net.bridge.bridge-nf-call-iptables = 1
           EOF

           sysctl --system

           #时间同步
           yum -y install ntpdate
           ntpdate ntp.aliyun.com
        
           # master 添加hosts
           cat >> /etc/hosts << EOF
           <ip addr> master
           <ip addr> node1
           <ip addr> node2
           EOF
        ```
     3. 所有nodes上操作
        ```
           #安装Docker
           wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo --no-check-certificate
           yum -y install docker-ce-18.06.1.ce-3.el7
           systemctl enable --now docker
           cat > /etc/docker/daemon.json << EOF
           {
              "registry-mirrors":["http://b9pmyelo.mirror.aliyunces.com"]
           }
           EOF
        
           # 重启docker 
           systemctl restart docker

           #添加阿里云yum软件源
           cat > /etc/yum.repos.d/kubernetes.repo << EOF
           [kubernetes]
           name=Kubernetes
           baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
           enabled=1
           gpgcheck=1
           repo_gpgcheck=1
           gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
           EOF
        ```
     4. 启动kubenetes
        ```
           #启动kubelet
           systemctl enable kubelet

           # 更改docker的驱动
           vim /etc/docker/daemon.json里加上这句话
           "exec-opts": ["native.cgroupdriver=systemd"]
           systemctl restart docker
           docker info | grep Cgroup
        ```

     5. Master节点上 Kubernentes init
        ```
           #代理，国内无法访问k8s.io
           kubeadm init \
           --apiserver-advertise-address=<Master's ip addr> \
           --image-repository registry.aliyuncs.com/google_containers \
           --kubernetes-version v1.22.2 \
           --service-cidr=10.96.0.0/12 \
           --pod-network-cidr=10.244.0.0/16

           #成功之后，添加权限
           mkdir -p $HOME/.kube
           sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
           sudo chown $(id -u):$(id -g) $HOME/.kube/config
           kubectl get nodes

     6. 子节点上执行
        ```
           #加入集群，指令在master节点init成功时自动打出来。
           #类似于 kubeadm join 192.168.31.46:6443 --token ufw0ia.wiaf4vru6t2v9982 --discovery-token-ca-cert-hash sha256:d21a2c79bf368e91927ef3ea68172d29097478918f655b1d0d4ddcc9a155037e

           #token有效期24小时，如果过期了就在master节点输入：
           kubeadm token create --print-join-command
        ```

     7. 部署CNI网络插件
        ```
           kubectl apply -f  https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
           kubectl get pods -n kube-system
        ```
        
   
## 参考文档
1. https://developer.aliyun.com/article/927657