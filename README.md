### 1. 操作脚本之前先要想清楚是部署集群模式还是单master模式

### 2. 检查变量文件，检查 IP 地址和端口好的配置，不要漏缺，检查网卡名称配置

### 3. 升级内核

```sh
# 提前下载到本地，避免网络问题漫长的等待
[root@k8s-master1 ~]# wget https://linux.cc.iitk.ac.in/mirror/centos/elrepo/kernel/el7/x86_64/RPMS/kernel-lt-5.4.160-1.el7.elrepo.x86_64.rpm
[root@k8s-master1 ~]# ansible -i hosts.ini k8s -m copy -a "src=./kernel-lt-5.4.160-1.el7.elrepo.x86_64.rpm dest=/tmp/kernel-lt-5.4.160-1.el7.elrepo.x86_64.rpm mode=0644" --become
[root@k8s-master1 ~]# ansible-playbook -i hosts.ini install_kernel.yml
[root@k8s-master1 ~]# wget https://github.com/etcd-io/etcd/releases/download/v3.5.1/etcd-v3.5.1-linux-amd64.tar.gz
[root@k8s-master1 ~]# ansible -i hosts.ini etcd -m copy -a "src=./etcd-v3.5.1-linux-amd64.tar.gz dest=/usr/local/src/etcd-v3.5.1-linux-amd64.tar.gz mode=0644" --become
```

### 4. 提前拷贝ETCD文件

```sh
# 由于网络问题可以提前搞一下，如果你的机器可以科学上网则无需
[root@k8s-master1 ~]# ansible -i hosts.ini etcd -m copy -a "src=./roles/etcd/files/cfssl_linux-amd64 dest=/usr/local/bin/cfssl mode=0755" --become
[root@k8s-master1 ~]# ansible -i hosts.ini etcd -m copy -a "src=./roles/etcd/files/cfssljson_linux-amd64 dest=/usr/local/bin/cfssl-json mode=0755" --become
[root@k8s-master1 ~]# ansible -i hosts.ini etcd -m copy -a "src=./roles/etcd/files/cfssl-certinfo_linux-amd64 dest=/usr/local/bin/cfssl-certinfo mode=0755" --become
[root@k8s-master1 ~]# ansible -i hosts.ini etcd -m copy -a "src=./roles/etcd/files/etcd-v3.5.1-linux-amd64.tar.gz dest=/usr/local/src/etcd-v3.5.1-linux-amd64.tar.gz mode=0644" --become
```

### 4. 运行脚本启动集群

```
[root@k8s-master1 ~]# yum -y install ansible sshpass

[root@k8s-master1 ~]# ssh-keygen -t rsa

[root@k8s-master1 ~]# vim hosts.txt
10.4.212.151
10.4.212.152
10.4.212.153
10.4.212.154
10.4.212.155
10.4.212.156
10.4.212.157
10.4.212.158
10.4.212.159
10.4.212.160

[root@k8s-master1 ~]# for host in $(cat iplist.txt); do sshpass -p 'your_password' ssh-copy-id -o StrictHostKeyChecking=no 'your_username'@$host; done

[root@k8s-master1 ~]# ansible -i hosts.ini all -m shell -a "whoami"

[root@k8s-master1 ~]# ansible-playbook -i hosts.ini multi-master-ha-deploy.yml   # 集群部署

[root@k8s-master1 ~]# ansible-playbook -i hosts.ini single-master-deploy.yml  # 单节点部署
```

### 5. 验证集群

```
# 也可使用 https://etc1:2379 域名
[root@k8s-master1 ~]# etcdctl --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/server.pem --key=/etc/etcd/ssl/server-key.pem --endpoints="https://10.4.212.151:2379,https://10.4.212.152:2379,https://10.4.212.153:2379" member list -w table
+------------------+---------+-------+---------------------------+---------------------------+------------+
|        ID        | STATUS  | NAME  |        PEER ADDRS         |       CLIENT ADDRS        | IS LEARNER |
+------------------+---------+-------+---------------------------+---------------------------+------------+
| ac050d32d6a110ec | started | etcd3 | https://10.4.212.153:2380 | https://10.4.212.153:2379 |      false |
| ea122616f14b31db | started | etcd2 | https://10.4.212.152:2380 | https://10.4.212.152:2379 |      false |
| f17683dfe42353c0 | started | etcd1 | https://10.4.212.151:2380 | https://10.4.212.151:2379 |      false |
+------------------+---------+-------+---------------------------+---------------------------+------------+
[root@k8s-master1 ~]# etcdctl --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/server.pem --key=/etc/etcd/ssl/server-key.pem --endpoints="https://10.4.212.151:2379,https://10.4.212.152:2379,https://10.4.212.153:2379" endpoint status -w table
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://10.4.212.151:2379 | f17683dfe42353c0 |   3.5.1 |  6.8 MB |     false |      false |         7 |      23532 |              23532 |        |
| https://10.4.212.152:2379 | ea122616f14b31db |   3.5.1 |  6.8 MB |     false |      false |         7 |      23532 |              23532 |        |
| https://10.4.212.153:2379 | ac050d32d6a110ec |   3.5.1 |  6.8 MB |      true |      false |         7 |      23532 |              23532 |        |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
[root@k8s-master1 ~]# etcdctl --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/server.pem --key=/etc/etcd/ssl/server-key.pem --endpoints="https://10.4.212.151:2379,https://10.4.212.152:2379,https://10.4.212.153:2379" endpoint health --write-out=table
+---------------------------+--------+-------------+-------+
|         ENDPOINT          | HEALTH |    TOOK     | ERROR |
+---------------------------+--------+-------------+-------+
| https://10.4.212.152:2379 |   true | 21.748801ms |       |
| https://10.4.212.151:2379 |   true | 21.666785ms |       |
| https://10.4.212.153:2379 |   true | 24.024215ms |       |
+---------------------------+--------+-------------+-------+

[root@k8s-master1 ~]# kubectl get cs

[root@k8s-master1 ~]# kubectl get node

[root@k8s-master1 ~]# kubectl get pods -A

[root@k8s-master1 ~]# kubectl create deployment nginx --image=harbor.meta42.indc.vnet.com/library/nginx:latest --replicas=4

[root@k8s-master1 ~]# kubectl expose deployment nginx --port=80 --target-port=80 --type=NodePort
```

### 7. 调整 kube 启动参数

```sh
# 请手动执行以下命令来修改 kube 自定义配置：
[root@k8s-master1 ~]# sed -i "/image:/i\    - --feature-gates=RemoveSelfLink=false" /etc/kubernetes/manifests/kube-apiserver.yaml # 每台 master 都执行
[root@k8s-master1 ~]# sed -i "s/bind-address=127.0.0.1/bind-address=0.0.0.0/g" /etc/kubernetes/manifests/kube-controller-manager.yaml # 每台 master 都执行
[root@k8s-master1 ~]# kubectl get cm -n kube-system kube-proxy -o yaml | sed "s/metricsBindAddress: \"\"/metricsBindAddress: \"0.0.0.0\"/g" | kubectl replace -f -
[root@k8s-master1 ~]# kubectl rollout restart daemonset -n kube-system kube-proxy
```

### 8. 解决 node 节点报错

```
Sep 14 00:59:22 k8s-node1 kubelet[1611]: E0914 00:59:22.040084    1611 file_linux.go:61] "Unable to read config path" err="path does not exist, ignoring" path="/etc/kubernetes/manifests"
[root@k8s-master1 ~]# ansible -i hosts.ini node -m shell -a "mkdir -pv /etc/kubernetes/manifests"
[root@k8s-master1 ~]# ansible -i hosts.ini node -m shell -a "systemctl restart kubelet"
```
