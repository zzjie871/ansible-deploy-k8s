一、环境准备

操作系统：CentOS 7.7，最小化安装

ansible host 配置文件
# cat /etc/ansible/hosts
[master]
# 如果部署单Master，只保留一个 Master 节点
# 默认Naster节点也部署Node组件
172.16.106.187    node_name=master01
172.16.106.188    node_name=master02
172.16.106.189    node_name=master03

[node]
172.16.106.195    node_name=node01
172.16.106.196    node_name=node02
172.16.106.197    node_name=node03

[etcd]
172.16.106.187    etcd_name=etcd01
172.16.106.188    etcd_name=etcd02
172.16.106.189    etcd_name=etcd03

[lb]
# 如果部署单Master，该项忽略
172.16.106.187 lb_name=lb-master LB_ROLE=master EX_APISERVER_VIP=172.16.106.198 EX_APISERVER_PORT=8443
172.16.106.188 lb_name=lb-backup LB_ROLE=backup EX_APISERVER_VIP=172.16.106.198 EX_APISERVER_PORT=8443

[k8s:children]
master
node

环境变量配置文件，放在与 roles 同级目录的 group_vars 下
# cat group_vars/all.yml 
# 安装目录
ansible_dir: '/app/ansible/deploy_k8s'        # ansible roles 文件目录
software_dir: '/root/software/binary_pkg'     # 提前准备二进制包
base_dir: '/opt/work'                         # ansible   
k8s_work_dir: '/opt/kubernetes'               # kubernetes 组件工作目录，包括 bin、cfg、ssl
etcd_work_dir: '/opt/etcd'                    # etcd 组件工作目录
tmp_dir: '/tmp/k8s'                           # 文件分发临时目录

# 证书目录
etcd_ca_dir: '/etc/etcd/ssl'

# 集群网络
service_cidr: '10.254.0.0/16'
cluster_dns: '10.254.0.2'                     # 与 coredns 中IP一致，并且是service_cidr中的IP
pod_cidr: '10.10.0.0/16'                      # 与 calico 中网段一致
service_nodeport_range: '30000-32767'
cluster_domain: 'cluster.local'

# 高可用，如果部署单Master，该项忽略
vip: '172.16.106.198'                         # 作为 虚拟地址，apiserver 的代理地址
nic: 'eth0' 

# 自签证书可信任IP列表，为方便扩展，可添加多个预留IP，添加域名等
cert_hosts:
  master:
    - 172.16.106.187
    - 172.16.106.188
    - 172.16.106.189

  node:
    - 172.16.106.195
    - 172.16.106.196
    - 172.16.106.197

  # 包含所有etcd节点IP
  etcd:
    - 172.16.106.187
    - 172.16.106.188
    - 172.16.106.189

二、部分配置说明
1、证书部分：
大部分证书都在 tls roles 中生成。如果需要添加其他组件证书，可以在这里提前添加，或者是在新的 roles 中独立签发。

2、关于 calico 部分：
$ diff calico.yaml.orig calico.yaml
630c630,632
<               value: "192.168.0.0/16"         # 新版本的这项处于注释状态，取消注释即可
---
>               value: "10.10.0.0/16"           # 修改
>             - name: IP_AUTODETECTION_METHOD   # 添加
>               value: "interface=eth.*"        # 添加
# 将 Pod 网段地址修改为 10.10.0.0/16;
# calico 自动探查互联网卡，如果有多快网卡，则可以配置用于互联的网络接口命名正则表达式，如上面的 'eth.*'(根据自己服务器的网络接口名修改)；

3、容器部分
注意：使用 containerd 需要添加以下2行；使用 docker 时，不需要添加
--container-runtime=remote \
--container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \


4、关于 kubeconfig 说明
这里的所有 kubeconfig 的 server 项都是使用 nginx 代理地址。
通过 nginx 透传代理 apiserver。
这里有个问题，必须使用透传代理，不然无法正常使用，普通的 upstream 需要带上证书，会有一些列的麻烦。也是没有使用 haproxy 的原因。

kubelet-bootstrap.kubeconfig 配置文件需要绑定 node name，在部署 node 时独立配置。
这时需要使用到一些证书去签发，一般 ca 的证书只在签发节点上。有些证书不需要分发到 node 节点上。
但是 签发 kubelet 证书和配置文件时需要使用，先分发到 node 节点上，部署完成后，可以删除部分证书。



三、使用ansible 部署 kubernetes 集群
# pwd
/app/ansible/deploy_k8s
# tree -L 1
├── 00-setup.yml
├── 01-common.yml
├── 02-tls.yml
├── 03-prepare.yml
├── 04-etcd.yml
├── 05-ha.tml
├── 06-master.yml
├── 07-node.yml
├── 08-docker.yml
├── 09-network.yml
├── clean-etcd.yml
├── clean-master.yml
├── group_vars
├── restart-all-service.yml
└── roles

# tree roles -L 1
roles
├── calico
├── common
├── docker
├── etcd
├── lb-haproxy
├── lb-nginx
├── master
├── node
├── prepare
└── tls

执行部署
# ansible-playbook 00-setup.yml

分步部署
# ansible-playbook 01-common.yml

需要重新执行某个任务，可以单独添加 tags
# ansible-playbook --tags=restart_node_service 07-node.yml



四、修改部分配置
初始化部署完成后，有些需要调整的内容
# 手动 approve server cert csr
kubectl get csr | grep Pending | awk '{print $1}' | xargs kubectl certificate approve

修改 kubelet 配置
vim ./roles/node/templates/kubelet.service.j2
添加下面此项
--network-plugin=cni \

重新分发 kubelet.service 文件，重启 kubelet 服务
# ansible-playbook --tags=restart_node_service 07-node.yml

删除之前部署的 calico 
kubectl delete -f /opt/kube/kube-system/calico.yaml

如下图：
# kubectl get pod -n kube-system -o wide
NAME                                       READY   STATUS              RESTARTS   AGE     IP               NODE       NOMINATED NODE   READINESS GATES
calico-kube-controllers-5554fcdcf9-dkwzd   1/1     Running             0          2m57s   172.17.0.2       master01   <none>           <none>
calico-node-484zj                          1/1     Running             0          103s    172.16.106.195   node01     <none>           <none>
calico-node-674zg                          0/1     Init:ErrImagePull   0          102s    172.16.106.196   node02     <none>           <none>
calico-node-dr49f                          0/1     PodInitializing     0          108s    172.16.106.187   master01   <none>           <none>
calico-node-fhlst                          0/1     PodInitializing     0          103s    172.16.106.188   master02   <none>           <none>
calico-node-v9h4j                          0/1     Init:0/3            0          102s    172.16.106.197   node03     <none>           <none>
calico-node-wpdcx                          1/1     Running             0          103s    172.16.106.189   master03   <none>           <none>

之前部署的 pod calico-kube-controllers 是 docker 自带的网网段，这也说明之前的 calico 没有生效。

重新部署 calico 
kubectl apply -f /opt/kube/kube-system/calico.yaml
# kubectl get pod -n kube-system -o wide
NAME                                       READY   STATUS    RESTARTS   AGE     IP               NODE       NOMINATED NODE   READINESS GATES
calico-kube-controllers-5554fcdcf9-g8bsk   1/1     Running   0          4h37m   10.10.59.193     master02   <none>           <none>
calico-node-454g6                          1/1     Running   0          4h37m   172.16.106.187   master01   <none>           <none>
calico-node-4pdk6                          1/1     Running   0          4h37m   172.16.106.195   node01     <none>           <none>
calico-node-6k82q                          1/1     Running   0          4h37m   172.16.106.188   master02   <none>           <none>
calico-node-hqkcd                          1/1     Running   0          4h37m   172.16.106.197   node03     <none>           <none>
calico-node-tn6fq                          1/1     Running   0          4h37m   172.16.106.189   master03   <none>           <none>
calico-node-vg77f                          1/1     Running   0          4h37m   172.16.106.196   node02     <none>           <none>

再次手动 approve server cert csr
kubectl get csr | grep Pending | awk '{print $1}' | xargs kubectl certificate approve



五、问题记录：
1、部署 kubelet 和 calico 时需要注意:
在部署 kubelet 时添加了 --network-plugin=cni 项。
会一直报错：KubeletNotReady runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized

这个问题，导致 kubernetes work node 一直处于 NotReady 状态。就无法部署 calico。
calico需要以 DaemonSet 的方式托管在 kubernetes 集群上面，而 node 无法就绪，就无法部署 calico。

临时解决办法。
部署 kubelet 时去掉 --network-plugin=cni 项，重启 kubelet。

# 手动 approve
# kube-controller-manager 为各 node 生成了 kubeconfig 文件和公私钥：
kubectl get csr | grep Pending | awk '{print $1}' | xargs kubectl certificate approve

通过上面的步骤，使用 node 处于就绪状态。

但是 去掉 --network-plugin=cni 项，又无法使用 calico 网络。默认使用的是 docker 网络。
在 node 处于就绪状态后，部署 calico 网络组件，再次添加此项，重启 kubelet。
再次，删除 calico，重新部署 calico。

之前使用 flannel 网络组件，并且 flannel 部署的步骤先于 master 和 node 组件，没有这个问题。本次学习中遇到了这个问题。

2、其他问题
如果 kube-apiserver、kube-controller-manager 等报错证书问题。
1) 检查证书配置文件；
2) 在使用 kube-apiserver.service 一些不了解配置项，也可能导致这个问题，删除一些配置项，重新部署。

3) 如果 master 组件有报错，先解决问题。别往下走。



六、参考文档说明
1) 和我一步步部署 kubernetes 集群
https://github.com/opsnull/follow-me-install-kubernetes-cluster

2) kubernetes-handbook
https://github.com/feiskyer/kubernetes-handbook/blob/master/SUMMARY.md

3) kubeasz 项目
https://github.com/easzlab/kubeasz/

4) kube-apiserver 配置参数，kube-apiserver.service 服务文件
https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-apiserver/



七、其他
以上内容是个人学习记录，并参考网络上很多资料。
如果侵权，请联系删除。邮箱地址：y5530961@163.com
有问题也可相互交流学习。

