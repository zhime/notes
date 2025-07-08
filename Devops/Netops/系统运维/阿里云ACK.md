## 简介

地址：[https://csnew.console.aliyun.com](https://csnew.console.aliyun.com/)

阿里云容器服务 Kubernetes 版（Alibaba Cloud Container Service for Kubernetes，简称 ACK）是阿里云提供的一种托管的 Kubernetes 服务。ACK 帮助用户在云端轻松部署、管理和扩展容器化应用，支持原生 Kubernetes API，并与阿里云的其他服务无缝集成。

ACK 提供了全托管的 Kubernetes 集群管理能力，帮助开发者专注于应用开发和业务创新，而无需关心底层基础设施的运维工作。无论是初创公司还是大型企业，都可以通过 ACK 实现高效的应用交付和弹性伸缩。

------

## 核心功能

### **集群信息**

![image-30](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-30.png)

### **集群节点池**

集群节点扩缩容在节点池操作

![image-31](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-31.png)

![image-32](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-32.png)

### **命名空间**



### **集群资源**

- 集群资源使用与自建一致
    - 支持 Kubernetes 标准的资源对象，例如 Pod、Node、Namespace、PersistentVolume 等
    - 资源调度、自动扩缩容（HPA）、网络策略等功能
    - 支持标准的 Kubernetes 网络插件（如 Calico、Flannel），同时也提供了阿里云优化的网络插件 Terway
    - 支持标准的 PersistentVolume 和 StorageClass，并且集成了阿里云的云盘、NAS、OSS 等存储服务

### **权限管理**

```
安全管理`-- `授权` -- `RAM用户`--`管理权限
添加权限`-- `选择空间`-- `选择权限`--`提交授权
```

### **集群监控**