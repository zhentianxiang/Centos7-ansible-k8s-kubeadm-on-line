# Kubernetes kubeadm 高可用集群自动部署
## 安装条件
```
环境： Centos 7.x, ansible2
适用k8s版本: v1.16.x,v1.17.x,v1.18.x,v1.19,v1.20.x
```

### 1、修改Ansible文件

修改hosts文件，根据规划修改对应IP和名称
```
# vi hosts.ini
...
```

修改group_vars/all.yml文件, 修改k8s版本等信息
```
# vim group_vars/all.yml
...
```
### 3、一键部署

测试ansible是否能连接服务器, 显示root权限即可

```
ansible -i hosts.ini all -m shell -a "whoami"
```

单Master版：
```
ansible-playbook -i hosts.ini single-master-deploy.yml
```

Master-HA 版：
```
ansible-playbook -i hosts.ini multi-master-ha-deploy.yml
```

多Master-LVS 版：
```
ansible-playbook -i hosts.ini multi-master-lvs-deploy.yml
```

### 4、部署控制
如果安装某个阶段失败，可针对性测试.

例如：只运行部署插件
```
ansible-playbook -i hosts.ini single-master-deploy.yml -t master,node
```

### 5、节点扩容
修改hosts，添加新节点ip到[newnode]标签
```
# vim hosts.ini
[all]
node02 ansible_host=192.168.80.35 ip=192.168.80.35



[node]
node02

[newnode]
node02
...
```

执行部署
```
ansible-playbook -i hosts.ini add-node.yml
```

### 6、移除k8s集群
```
ansible-playbook -i hosts.ini remove-k8s.yml
```
