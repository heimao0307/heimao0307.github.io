# 在Kubernetes中使用NFS持久化存储

[TOC]

**文档格式说明:**
代码和指令:     `yum update -y`;
注释:             #notice that ...

## 一.搭建NFS服务器

**本章内容在NFS服务器执行**

### 1.挂载磁盘分区

参考文章[Linux系统挂载磁盘分区](http://10.22.30.200:3000/projects/gyyf-tdps-manage-tdjv/knowledgebase/articles/77) #将磁盘挂载到/nfs

### 2.安装

`yum install -y nfs-utils rpcbind`

### 3.配置

```bash
chown -R nfsnobody.nfsnobody /nfs

编辑/etc/exports,添加以下内容
/nfs 192.168.1.0/24(rw,async,no_root_squash)  #允许挂载的网段

# 开启相应防火墙端口
firewall-cmd --permanent --zone={trusted,public} --add-port={111,2049}/tcp
firewall-cmd --reload  #重新加载防火墙规则
firewall-cmd --zone={trusted,public} --list-ports  #查看是否开启相应端口
```
### 4.启动

```bash
systemctl enable nfs && systemctl start nfs
systemctl enable rpcbind && systemctl start rpcbind
```
### 5.验证

`showmount -e localhost`

## 二.集群使用NFS存储

### 1.准备(所有工作节点执行)

```bash
yum install -y nfs-utils
systemctl enable nfs && systemctl start nfs
```
### 2.创建PV(==Master==节点执行)

```yml
vi nfs-pv.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv  #可自定义
  labels:
    pv: nfs  #可自定义
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    path: /nfs  #与服务器一致
    server: <NFS server IP> #替换为实际NFS服务器IP

kubectl create -f nfs-pv.yaml -n <namespace> #创建在哪个命名空间
kubectl get pv -n <namespace>
```
### 3.创建PVC(Master节点执行)

```yml
vi nfs-pvc.yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc  #可自定义
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      pv: nfs  #与pv一致

kubectl create -f nfs-pvc.yaml -n <namespace> #创建在哪个命名空间
kubectl get pvc -n <namespace>
```
## 三.验证

### 1.创建验证文件(NFS服务器执行)

```bash
touch /nfs/test.html
echo "hello nfs" >> /nfs/test.html
```
### 2.创建测试POD(==Master==节点执行)

```yml
vi nfs-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: httpd-nfs-pod
spec:
  containers:
  - image: httpd
    name: httpd-withpvc-pod
    imagePullPolicy: Always
    volumeMounts:
    - mountPath: "/usr/local/apache2/htdocs/"
      name: httpd-volume
  volumes:
    - name: httpd-volume
      persistentVolumeClaim:
        claimName: gyyf-nfs-pvc  #与PVC名一致

kubectl create -f nfs-pod.yaml
```
### 3.验证(Master节点执行)

```bash
kubectl get pod -o wide  #获取POD IP

curl <POD IP>/test.html #替换为上句获取IP
```