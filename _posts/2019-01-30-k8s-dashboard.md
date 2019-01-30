---
layout: post
title: 'Kubernetes Dashboard的安装与坑'
#subtitle: 'linux'
date: 2019-01-30
categories: linux
tags: centos linux Kubernetes 
---



> Kubernetes Dashboard is a general purpose, web-based UI for Kubernetes clusters. It allows users to manage applications running in the cluster and troubleshoot them, as well as manage the cluster itself.  

# 1.前言

**一句话简单介绍下Kubernetes Dashboard**
Kubernetes Dashboard就是k8s集群的webui，集合了所有命令行可以操作的所有命令。界面如下所示：(ps：目前自动识别为中文版本)
![dashboard-ui.png](https://upload-images.jianshu.io/upload_images/15414238-d5f142cb8f97f170.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 2.安装

k8s的dashboard安装可以说是非常简单，参考github的指导既可。项目地址如下：

> https://github.com/kubernetes/dashboard
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml

```
但是这么安装存在几个问题：
1. 镜像国内无法直接访问，需要设置docker代理，才能下载镜像 

2. dashboard的默认webui证书是自动生成的，由于时间和名称存在问题，导致谷歌和ie浏览器无法打开登录界面，经过测试Firefox可以正常打开

## 2.1 设置docker代理

k8s dashboard 的 docker镜像是
`k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.0`
在执行 `kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml ` 前，首先设置docker代理

以下提供个脚本，可以方便切换docker代理
```bash
#/bin/bash

# you should set it to your proxy ip 
proxy_ip="http://192.168.246.1:1080"
# you need set it to the  host ip 
proxy_none_ip="192.168.0.0/16"   

proxy='Environment="HTTPS_PROXY='${proxy_ip}'"\
Environment="NO_PROXY=127.0.0.0/8"\
Environment="NO_PROXY='${proxy_none_ip}'"'
DOCKER_CONF="/usr/lib/systemd/system/docker.service"
#DOCKER_CONF="docker.service"
if [ ! -e $DOCKER_CONF ]; then 
	echo "INFO: docker not running "
	exit 2
fi
func_reload(){
	systemctl daemon-reload
	systemctl restart docker
	echo "INFO#: docker-reload finined!"
}
func_proxy_on(){
	if grep PROXY $DOCKER_CONF >> /dev/null ; then
		echo "INFO#: docker proxy may be on : "
		echo ""
		grep PROXY $DOCKER_CONF
		echo ""
	else
		echo "INFO: docker proxy on"
		sed -i "/ExecStart/i${proxy}" $DOCKER_CONF
		func_reload
	fi
}

func_proxy_off(){
	if grep PROXY $DOCKER_CONF >>/dev/null; then
        	echo "INFO: docker proxy off"
		sed -i "/PROXY/d" $DOCKER_CONF
		func_reload
	else
        	echo "INFO: docker proxy already off"
	fi
}

case $1 in
	on)
	  func_proxy_on
	  ;;
	off)
	  func_proxy_off
	  ;;
	*) 
	  echo "userage `basename $0` {on|off}"
	  exit 1
	  ;;
esac
```
请将 以上脚本中 `proxy_ip="http://192.168.246.1:1080" ` 替换为你自己的代理地址，保存为`dockersetproxy.sh` ，通过`chmod +x  dockersetproxy.sh`  增加执行权限 。
然后执行 `kubectl apply -f  https://......` 命令参考上面
如果能够正常下载，通过docker image ls查看，应该如下所示：

```
[root@master ~]# docker image ls
REPOSITORY                              TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy                   v1.12.3             ab97fa69b926        2 weeks ago         96.5 MB
k8s.gcr.io/kube-apiserver               v1.12.3             6b54f7bebd72        2 weeks ago         194 MB
k8s.gcr.io/kube-controller-manager      v1.12.3             c79022eb8bc9        2 weeks ago         164 MB
k8s.gcr.io/kube-scheduler               v1.12.3             5e75513787b1        2 weeks ago         58.3 MB
k8s.gcr.io/etcd                         3.2.24              3cab8e1b9802        2 months ago        220 MB
k8s.gcr.io/coredns                      1.2.2               367cdc8433a4        3 months ago        39.2 MB
k8s.gcr.io/kubernetes-dashboard-amd64   v1.10.0             0dab2435c100        3 months ago        122 MB
quay.io/coreos/flannel                  v0.10.0-amd64       f0fad859c909        10 months ago       44.6 MB
k8s.gcr.io/pause                        3.1                 da86e6ba6ca1        11 months ago       742 kB
```
`k8s.gcr.io/kubernetes-dashboard-amd64` 即为下载的docker image 镜像文件
下载完成后，k8s dashboard 应该正常运行起来了，但是这时候我们还无法访问到。
##2.2 修改service通过NodePort方式访问k8s dashboard
> 小技巧，由于后面的操作都是在 kube-system 名称空间中进行，可以设置个别名  ksys=kubectl -n kube-system 这样就可以使用ksys操作该名称空间了
> 命令参考：`alias ksys='kubectl -n kube-system'`
```
[root@master ~]# alias ksys='kubectl -n kube-system'
[root@master ~]# ksys get svc
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
kube-dns               ClusterIP   10.96.0.10     <none>        53/UDP,53/TCP   15d
kubernetes-dashboard   ClusterIP   10.106.68.90   <none>        443/TCP         15s
[root@master ~]# 

```
可以看到 kubernetes-dashboard  service 在集群内部，无法再外部访问，为了方便访问，我们暴露kubernetes-dashboard 443端口给NodePort
`ksys edit svc kubernetes-dashboard` 通过ksys edit svc 直接编辑service
```
[root@master ~]# ksys edit svc kubernetes-dashboard
```
找到type字段，将ClusterIP，修改为NodePort
```
spec:
  clusterIP: 10.106.68.90
  ports:
  - port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: ClusterIP ## <------修改为NodePort
status:
  loadBalancer: {}

```
wq 保存退出，然后重新查看 service
```
[root@master ~]# ksys get svc
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
kube-dns               ClusterIP   10.96.0.10     <none>        53/UDP,53/TCP   15d
kubernetes-dashboard   NodePort    10.106.68.90   <none>        443:32248/TCP   4m41s
[root@master ~]# 
```
可以看到当前NodePort 端口是随机的32248，通过ifconfig 查看节点ip地址，该节点ip为：192.168.246.200
```
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 02:42:3a:a2:76:1f  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.246.200  netmask 255.255.255.0  broadcast 192.168.246.255
        inet6 fe80::1d7c:9fdf:c738:7459  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:21:65:3b  txqueuelen 1000  (Ethernet)
        RX packets 10074  bytes 1051745 (1.0 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 10716  bytes 7583211 (7.2 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
通过谷歌浏览器访问，发现居然无法继续，如下图所示：

![image.png](https://upload-images.jianshu.io/upload_images/15414238-19c712711db26f57.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过360浏览器访问，发现居然直接无法访问

![image.png](https://upload-images.jianshu.io/upload_images/15414238-f3eb6c5185f045a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在测试IE、QQ等浏览器，均无法访问，
在测试windows机器上通过curl命令测试，可以确认网络和端口是通的。

![image.png](https://upload-images.jianshu.io/upload_images/15414238-851ad0aa37d26c5c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

难道就无解了么？
再拿出firefox测试，发现证书是0001年1月签发的

![image.png](https://upload-images.jianshu.io/upload_images/15414238-757b2a5163a2ca58.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

添加例外后，居然能正常打开了。

![image.png](https://upload-images.jianshu.io/upload_images/15414238-3ba722a194e715f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

难道这就完事了么？ 通过Firefox查看证书，怀疑其他浏览器打不开和证书过期有关系。

![image.png](https://upload-images.jianshu.io/upload_images/15414238-cc1cfbae2df9788c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2.2 解决证书过期问题
### 2.2.1 首先需要生成证书

生成证书通过openssl生成自签名证书即可，不再赘述，参考如下所示：

```
[root@master keys]# pwd
/root/keys
[root@master keys]# ls
[root@master keys]# openssl genrsa -out dashboard.key 2048
Generating RSA private key, 2048 bit long modulus
.+++
.................................................+++
e is 65537 (0x10001)
[root@master keys]# openssl req -new -out dashboard.csr -key dashboard.key -subj '/CN=192.168.246.200'
[root@master keys]# ls
dashboard.csr  dashboard.key
[root@master keys]# 
[root@master keys]# openssl x509 -req -in dashboard.csr -signkey dashboard.key -out dashboard.crt 
Signature ok
subject=/CN=192.168.246.200
Getting Private key
[root@master keys]# 
[root@master keys]# ls
dashboard.crt  dashboard.csr  dashboard.key
[root@master keys]# 
[root@master keys]# openssl x509 -in dashboard.crt -text -noout
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number:
            f0:8a:26:aa:9f:24:bf:92
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=192.168.246.200
        Validity
            Not Before: Dec 13 08:10:36 2018 GMT
            Not After : Jan 12 08:10:36 2019 GMT
        Subject: CN=192.168.246.200
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:f6:7a:b4:4a:ad:bd:b3:00:9c:d1:fe:06:2d:09:
                    cf:eb:28:54:0f:3f:6e:dc:29:6b:67:e1:9b:58:e4:
                    82:00:15:ee:35:25:00:4c:c1:e0:1b:29:8b:b2:6b:
                    8d:e8:09:77:66:4d:f3:9e:9d:85:36:94:80:da:1b:
                    35:c8:a1:b3:0b:b2:7f:6f:1e:e9:fe:fc:15:1b:7b:
                    ba:85:1f:2b:70:16:d5:c3:7f:36:18:f1:8e:44:1e:
                    8a:13:a2:9c:b8:bf:b8:08:3f:a0:5c:ef:19:f5:ce:
                    73:0c:3e:0a:b5:87:7a:de:25:74:36:0e:26:52:ff:
                    4b:d0:24:40:c9:03:9a:44:f6:17:a7:d7:fa:7e:e0:
                    fb:6a:76:5b:dc:0f:43:c2:63:f4:22:20:4c:4e:5d:
                    b7:a0:83:54:58:1c:10:0f:57:ef:ad:1f:36:0b:8f:
                    8d:f4:a2:52:ab:e7:39:57:ea:30:c3:1d:30:93:ee:
                    44:7f:73:ef:41:94:e8:34:8c:c4:bb:02:d9:17:da:
                    55:07:ff:43:6c:f3:8e:91:5f:81:03:e9:94:2e:f1:
                    25:e7:41:86:e2:25:c4:b9:07:b4:9c:d9:04:36:31:
                    82:43:1b:26:10:17:8c:98:4a:f3:23:69:15:1b:76:
                    75:ae:4e:27:6f:70:4c:c6:f7:cc:75:e4:ed:48:b7:
                    51:c5
                Exponent: 65537 (0x10001)
    Signature Algorithm: sha256WithRSAEncryption
         28:55:3c:0a:66:77:2a:fd:8a:b6:81:54:59:13:d7:03:17:7f:
         d4:fa:e4:94:2b:bc:f4:11:ea:0c:18:e9:c0:2c:02:86:eb:39:
         12:38:19:71:6c:b8:7a:4d:03:57:59:4f:c0:50:c4:19:92:c1:
         9f:2f:0d:18:92:9e:2b:2e:a2:44:52:9a:32:2b:75:35:fb:43:
         66:fb:fa:32:77:ce:b8:4e:80:cb:38:52:c4:2c:17:11:1a:38:
         c3:a9:62:43:5e:60:ae:47:d4:f7:46:12:29:f5:e4:75:35:e5:
         90:5d:2e:4f:2f:c5:65:9a:e5:6a:4d:8a:cd:69:ba:e0:4f:43:
         d1:ab:9a:62:74:fc:d5:88:9c:3a:ba:22:2d:38:96:fc:35:b0:
         3c:23:f7:8c:23:07:4e:05:8e:ae:53:82:9c:fd:54:24:86:75:
         12:a6:e9:77:62:bd:f6:bb:f9:4d:5b:64:1e:d0:48:68:31:86:
         f5:36:b5:6b:fc:b6:36:f0:01:3c:0a:9f:2b:27:56:28:1d:1f:
         c4:e9:f7:c6:5d:16:5e:88:c5:e0:43:00:bf:79:d7:04:2f:45:
         57:df:e6:17:dd:5a:f8:53:e9:ca:f6:33:ed:19:f0:d9:0a:ae:
         f0:ba:c6:5b:7e:70:af:c3:f3:a5:b0:95:b0:ee:cd:35:29:5c:
         34:4a:ce:49

```
这样就有了证书文件dashboard.crt 和 私钥 dashboad.key

### 2.2.2 下载yaml，并修改
`wget  https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml`
将该配置文件下载下来

```yaml
# ------------------- Dashboard Secret ------------------- #

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kube-system
type: Opaque
---
....................省略一堆信息
 spec:
      containers:
      - name: kubernetes-dashboard
        image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.0
        ports:
        - containerPort: 8443
          protocol: TCP
        args:
          - --auto-generate-certificates
          # Uncomment the following line to manually specify Kubernetes API server Host
          # If not specified, Dashboard will attempt to auto discover the API server and connect
          # to it. Uncomment only if the default does not work.
          # - --apiserver-host=http://my-address:port
        volumeMounts:
        - name: kubernetes-dashboard-certs   <-----------这里可以看到secret挂载到了certs目录
          mountPath: /certs
          # Create on-disk volume to store exec logs
        - mountPath: /tmp
          name: tmp-volume
。。。。。。。。。。省略无用信息
volumes:
      - name: kubernetes-dashboard-certs
        secret:
          secretName: kubernetes-dashboard-certs <---secret 可以看到secret创建为了volume
      - name: tmp-volume
        emptyDir: {}
      serviceAccountName: kubernetes-dashboard

```
所以，我们需要重新生成secret，并且将该配置文件中创建secret的配置文件信息去掉，将以下内容 从配置文件中去掉：
```yaml
# ------------------- Dashboard Secret ------------------- #

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kube-system
type: Opaque

---
```
可以在配置文件中，修改service 为nodeport类型，固定访问端口
修改前：
```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:

  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
```
修改后：
```yaml
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
      nodePort:30001
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
```
### 2.2.3 生成secret

 创建同名称的secret：
名称为： kubernetes-dashboard-certs

```bash
[root@master keys]# ls
dashboard.crt  dashboard.csr  dashboard.key  kubernetes-dashboard.yaml
[root@master keys]# ksys create secret generic kubernetes-dashboard-certs --from-file=dashboard.key --from-file=dashboard.crt 
secret/kubernetes-dashboard-certs created
[root@master keys]# 
[root@master keys]# ksys get secret | grep dashboard
kubernetes-dashboard-certs                        Opaque                                2      25s
kubernetes-dashboard-key-holder                  Opaque                                2      25h
[root@master keys]# 
[root@master keys]# ksys describe secret kubernetes-dashboard-certs  
Name:         kubernetes-dashboard-certs
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
dashboard.crt:  993 bytes
dashboard.key:  1675 bytes
[root@master keys]# 
```
可以看到，已经成功创建了 secret文件

### 2.2.4 重新apply yaml文件

应用下载到本地并且修改过的yaml文件，如下所示：

```bash
[root@master keys]# ls
dashboard.crt  dashboard.csr  dashboard.key  kubernetes-dashboard.yaml
[root@master keys]# 
[root@master keys]# 
[root@master keys]# kubectl apply -f kubernetes-dashboard.yaml 
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created
[root@master keys]# 
```

查看服务状态：

```
[root@master keys]# ksys get svc
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
kube-dns               ClusterIP   10.96.0.10     <none>        53/UDP,53/TCP   15d
kubernetes-dashboard   NodePort    10.111.32.20   <none>        443:30001/TCP   2m14s
[root@master keys]# 

```

通过浏览器访问：

![image.png](https://upload-images.jianshu.io/upload_images/15414238-0ab378fd5cb1e4dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/15414238-03f483be1dba7e8e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

查看证书信息如下所示：

![image.png](https://upload-images.jianshu.io/upload_images/15414238-4a02895611f60f1b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

firefox 上查看证书信息：

![image.png](https://upload-images.jianshu.io/upload_images/15414238-43bf2062b30580e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


至此，k8s dashboard 部署完成。