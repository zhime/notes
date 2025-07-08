### **简介**

Harbor 是一个开源的云原生镜像仓库项目，由 VMware 发起并于 2018 年捐赠给云原生计算基金会（CNCF），现已成为 CNCF 的毕业项目。它旨在为企业提供安全、可扩展的容器镜像和云原生制品（如 Helm Chart）管理解决方案。Harbor 的核心功能包括存储、签名和扫描容器镜像，确保其安全性和合规性

### **安装**

#### **下载**

下载地址：https://github.com/goharbor/harbor/releases

#### **安装**

安装`Harbor`需提前安装：`Docker Engine`、`Docker Compose`

解压：`tar xf harbor-offline-installer-v2.12.2.tgz`

进入解压目录：`cp harbor.yml.tmpl harbor.yml`

修改`harbor.yml`配置文件

执行安装：`./install.sh`

#### **Nginx域名转发**

```
server {
	listen 80;
	server_name harbor.netopstec.com;

	return 301 https://$host$request_uri;
}

server {
        listen   443 ssl http2;
        server_name harbor.netopstec.com;

        ssl_certificate "/etc/nginx/certs/netopstec.com/netopstec.com.pem";
        ssl_certificate_key "/etc/nginx/certs/netopstec.com/netopstec.com.key";
        ssl_session_timeout 3h;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers off;
        client_max_body_size 20000M;

        location / {
                proxy_pass http://192.168.0.207:1080;
		proxy_http_version 1.1;
                #proxy_redirect off;
                #proxy_set_header Host $host:$server_port;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
        }
}
```

### **基本使用**

#### **Web UI**

登录地址：[https://harbor.netopstec.com](https://harbor.netopstec.com/)

用户名：`admin`

密码：`Harbor12345`

![image-10](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-10.png)

#### **Docker Cli**

登录：`docker login harbor.netopstec.com`，账号密码与`WEB UI`登录一致

拉取镜像：`docker pull harbor.netopstec.com/library/nginx:1.20.0`

上传镜像：`docker push harbor.netopstec.com/library/nginx:1.20.0`

退出登录：`docker logout harbor.netopstec.com`



<span style="color:red">注：`docker push`需要先登录`Harbor`，并将镜像`tag`与`Harbor`地址保持一致</span>

![image-11](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-11.png)