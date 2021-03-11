---
author : "jihongwei"
tags : ["k8s"]
date : 2021-03-11T11:24:00Z
title : "k8s本地调试环境搭建"
---


将代码下载到gopath内
https://github.com/kubernetes/kubernetes

```
mkdir -p $GOPATH/src/k8s.io
cd $GOPATH/src/k8s.io
git clone https://github.com/kubernetes/kubernetes
cd kubernetes

k8s版本和go的版本要相互对应
git checkout  tags/v1.16.7
go version go1.14.6 linux/amd64

按照对应平台编译
KUBE_BUILD_PLATFORMS=linux/amd64 make all  GOFLAGS=-v GOGCFLAGS="-N -l"


本地编译
./hack/local-up-cluster.sh

假设修改了api-server的代码 后编译重启时的命令
make GOGCFLAGS="-N -l" WHAT="cmd/kube-apiserver" # 假设只编译kube-apiserver这一个模块
./hack/local-up-cluster.sh -O
```



1.在启动的过程中可以看到要从网络上下载cfssl cfssljson cfssl-certinfo 
尝试预先下载命令，但是没有成功
```
curl -s -L -o /bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
curl -s -L -o /bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
curl -s -L -o /bin/cfssl-certinfo https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x /bin/cfssl*
```

后从go get下载 可以使用，避免了每次重启都要花时间下载
```go get -u github.com/cloudflare/cfssl/cmd/...```


2.启动时所需要的的容器
启动后查看k8s 状态 发现缺少很多容器，使用标准地址会由于网络问题超时
所需容器image可以提前下载，网络不通可以通过阿里云容器镜像服务-镜像中心-镜像搜索服务
或者使用dockerhub克隆镜像和阿里云镜像加速服务(https://blog.csdn.net/championzgj/article/details/93299777)
```
k8s.gcr.io/k8s-dns-sidecar                        
k8s.gcr.io/k8s-dns-kube-dns                       
k8s.gcr.io/k8s-dns-dnsmasq-nanny                  
k8s.gcr.io/kube-proxy-amd64                       
k8s.gcr.io/kube-apiserver-amd64                   
k8s.gcr.io/kube-scheduler-amd64                   
k8s.gcr.io/kube-controller-manager-amd64          
k8s.gcr.io/pause-amd64                            
k8s.gcr.io/coredns                                
k8s.gcr.io/etcd-amd64                             
k8s.gcr.io/pause
```

