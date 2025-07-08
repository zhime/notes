# **网营各平台SSL证书更换**

**说明：网营主域名(netopstec.com)SSL证书每年续费一次，续费时长一年。在SSL证书到期前一月，OA提交SSL证书续费申请。审核通过后，续费SSL证书，并更新各平台证书为新签证书。**

**网营各平台证书如下：**

## **1、阿里云**

### **1.1 阿里云ECS**

#### **1.1.1 WMS服务器(**47.110.254.254**)**

```
[root@midware netopstec.com]# pwd
/etc/nginx/certs/netopstec.com
[root@midware netopstec.com]# ls
backup-2024  netopstec.com.key  netopstec.com.pem
[root@midware netopstec.com]# tree -L 2 .
.
├── backup-2024
│   ├── netopstec.com.key
│   └── netopstec.com.pem
├── netopstec.com.key
└── netopstec.com.pem

1 directory, 4 files
[root@midware netopstec.com]# 
```

**说明：**

- 证书路径：`/etc/nginx/certs/netopstec.com`
- `backup-2024`：`2024`年证书备份文件
- `netopstec.com.key`：`nginx`配置`key`文件
- `netopstec.com.pem`：`nginx`配置`pem`文件

**证书替换**(以`2025`年新证书为例)：

```
[root@midware netopstec.com]# pwd
/etc/nginx/certs/netopstec.com
[root@midware netopstec.com]# ls
backup-2024  backup-2025  netopstec.com.key  netopstec.com.pem
[root@midware netopstec.com]# tree -L 2 .
.
├── backup-2024
│   ├── netopstec.com.key
│   └── netopstec.com.pem
├── backup-2025
│   ├── netopstec.com.key
│   └── netopstec.com.pem
├── netopstec.com.key
└── netopstec.com.pem

2 directories, 6 files
[root@midware netopstec.com]#
```

- 创建备份文件夹`backup-2025`，将证书文件上传至`backup-2025`
- 将证书文件命名`netopstec.com.key``netopstec.com.pem`
- 删除旧证书文件，将新证书拷贝至`/etc/nginx/certs/netopstec.com`
- `rm -f netopstec.* && mv backup-2025/* .`
- 测试`nginx`配置文件的语法：`nginx -t`
- 重新加载`nginx`配置文件：`nginx -s reload`

#### **1.1.2 E管家服务器(**118.31.169.153**)**

```
[root@iZbp18jskekhmitycmzq5aZ netopstec.com]# pwd
/usr/local/nginx/ssl/netopstec.com
[root@iZbp18jskekhmitycmzq5aZ netopstec.com]# ls
backup-2024  netopstec.com.key  netopstec.com.pem
[root@iZbp18jskekhmitycmzq5aZ netopstec.com]# tree -L 2 .
.
├── backup-2024
│   ├── netopstec.com.key
│   └── netopstec.com.pem
├── netopstec.com.key
└── netopstec.com.pem

1 directory, 4 files
```

**说明：**

- 证书路径：`/usr/local/nginx/ssl/netopstec.com`

**证书更换：**

- 证书更换与`WMS`服务器类似，`证书路径`不同
- 测试`nginx`配置文件的语法：`nginx -t`
- 重新加载`nginx`配置文件：`nginx -s reload`

#### **1.1.3 Godiva服务器(**8.129.173.187**)**

- `Godiva`服务器证书更换与`WMS`服务器类似
- 证书路径：`/etc/nginx/certs/netopstec.com`

## **2、聚石塔**

### **2.1 聚石塔ECS**

#### **2.1.1 OMS服务器(**39.100.74.107**)**

- `nginx`配置路径：`/etc/nginx`
- 未配置证书，无`DNS`解析

#### **2.1.2 CRM服务器(**39.100.75.80**)**

- 证书路径：`/home/netopstec/tengine/cert/netopstec.com`

#### **2.1.3 码商服务器(**8.142.27.17**)**

- 证书路径：`/usr/local/nginx/conf/certs/netopstec.com`

## **3、网营本地服务器**

**说明：网营本地服务器，公网入口映射为内网IP 192.168.1.236，nginx主从服务器绑定内网虚拟IP(192.168.1.236)，内网DNS记录值解析至192.168.1.236**

### **3.1 nginx服务器(主)(192.168.1.74)**

- 证书路径：`/etc/nginx/certs/netopstec.com`

### **3.2 nginx服务器(从)(192.168.1.75)**

- 证书路径：`/etc/nginx/certs/netopstec.com`

### **3.3 nginx服务器(旧)(192.168.1.73)**

- 证书路径：`/etc/nginx/certs/netopstec`

## **4、七牛云**

- 登录地址：`https://portal.qiniu.com`
- 上传证书：

![image-41](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-41.png)

- 修改每个`CDN`域名证书，`配置` -- `HTTPS配置` -- `修改配置` -- `选择证书` -- `确定`

![image-42](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-42.png)



![image-43](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-43.png)

- 修改配置有源站域名的对象存储，`friso-web`、`wms-prod`、`wms-dev`、`netops-bmp`

![image-44](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-44.png)



## **5、拼多多**

- 登录地址：`https://open.pinduoduo.com`
- 上传证书：

![image-45](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-45.png)

- 修改证书，`负载均衡` -- `实例管理` -- `管理` -- `公网域名管理` -- `编辑` -- `编辑证书` -- `保存`

![image-46](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-46.png)

## **6、其他**

### **6.1** `aiad.netopstec.com`**域名需将证书发给联世**

### **6.2** `bi-upload-prod.netopstec.com`**域名需将证书发给bi组(bi组七牛云源站域名)**