# Nginx EMQ Dashboard

[TOC]

## 使用 nginx 部署 EMQ Dashboard

### 1.概述

#### 1.1 功能结构 

EMQ Dashboard 前端页面使用 SPA (single-page application) 设计 , 后端使用 MochiWeb 服务器提供 [RESTful API](http://emqtt.io/docs/v2/rest.html)，同时后端服务器也代理了所有的前端路由请求并转发给前端。 这个架构在部署上可以实现良好的前后端分离，且依赖少，较大简化了 Dashboard 的启动流程。

#### 1.2 EMQ API 说明

> EMQ 提供了两类 API 接口：8080端口的 API 是提供给外部应用使用的，18083端口的 API 是提供给 EMQ Dashboard 自身使用的。　

- 提供给外部应用访问的 API

  ```bash
  http(s)://host:8080/api/v2/
  ```

- Dahboard 自身使用的 API

  ```bash
  http(s)://host:18083/api/v2/
  ```

####1.3  Dashboard 静态资源文件目录结构

> 注意: Dashboard 主页总是以绝对路径加载 API 以及其他资源。

```bash
www
    ├── favicon.ico
    ├── index.html
    └── static
        ├── css
        │   ├── *
        ├── fonts
        │   ├── *
        └── js
            ├── *
```



### 2. 部署说明

#### 2.1 nginx 服务器配置说明
> nginx 配置文件一般为 `/etc/nginx/nginx.conf` 或者 `install_path/conf/nginx.conf`, 这取决于你的安装方式。修改配置文件之后需要重启服务器才能应用更改。

- 基本的 nginx 虚拟主机配置

```bash
http {
        server {
                listen  80;
                # 多个域名使用空格分开
                server_name  example.com dashboard.example.com;

                # 开启 gzip 可以大幅提高 Dashboard 加载速度
                gzip  on;
                gzip_types  text/plain application/x-javascript text/css application/javascript text/javascript;

                # 直接支持静态文件
                location ~* ^.+.(jpg|jpeg|gif|css|png|js|ico|html)$ {
                    access_log  off;
                    expires  30d;
                }
        }
}
```

#### 2.2 简单的反向代理配置

> 该配置仅仅只是将浏览器发起的请求无差别反向代理到 Dashboard 后端服务器，Dashboard 后端服务器仍然处理了繁重的静态资源分发任务。

```bash
server {
        listen  80;
        server_name  example.com;
        
        location / {
            proxy_pass  http://127.0.0.1:18083;
        }
}
```

通过如上配置后，访问 `example.com` 就可以使用 Dashboard。

#### 2.3 使用 nginx 处理静态资源，并代理转发 API 的数据

- nginx 处理静态资源文件

  > 前端静态资源文件在 emq-dashboard 的 priv/www 目录下, 参见[https://github.com/emqtt/emq-dashboard/tree/master/priv/www](https://github.com/emqtt/emq-dashboard/tree/master/priv/www)  

- 将 `/api/v2` 的 的请求转发至 http://127.0.0.1:18083/api/v2

- 将 `/external_api` 的请求转发至 http://127.0.0.1:8080/api/v2

```bash
server {
        listen  80;
        server_name  example.com;
        
        # 静态资源
        location / {
            # 复制一份静态资源到此目录下
            root  /www;
            # 或者使用已安装在本机的 EMQ 里面的静态资源
            # /emqttd/lib/emq_dashboard-2.3.0/priv/www/
            index  index.html;
        }
        
        # 供 Dashboard 使用的 API，location 必须设置为 /api/v2
        location /api/v2/ {
            proxy_pass   http://127.0.0.1:18083/api/v2;
        }

        # 供外部应用调用的 API，/external_api 可自定为其他路径
        location /external_api {
            proxy_pass   http://127.0.0.1:8080/api/v2;
        }
}
```

#### 2.4 将 Dashboard 绑定到某个路径上
- 如需使用类似 `http://example.com/dashboard` 的 URL 访问 Dashboard ，参考配置如下

```bash
server {
        listen  80;
        server_name  example.com;

        location /dashboard {
            proxy_pass  http://127.0.0.1:18083/;
        }
        
        # 确保静态资源依赖与 Dashboard API 能通过绝对路径上访问到
        location /static {
            proxy_pass  http://127.0.0.1:18083/static/;
        }
        
        location /api/v2 {
            proxy_pass  http://127.0.0.1:18083/api/v2/;
        }
}
```


#### 2.5 其他配置
- 在 `proxy_pass` 配置时设置代理请求头，以携带客户端 IP 地址等信息
```bash
server {
        listen  80;
        server_name  example.com;

        location /dashboard {
            proxy_pass  http://127.0.0.1:18083/;
            # 设置代理请求头
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Real-PORT $remote_port;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
}
```

### 3. 参考资料

- [nginx 中文文档](http://www.nginx.cn/doc/)


- [nginx 英文文档](http://www.nginx.cn/doc/)

