
# Authentication Via X509 Certificate

## Client --TLS--> EMQ 

etc/emq.conf 配置 EMQ TLS 监听器，开启 TLS 双向认证并配置 `peer_cert_as_username`：

```
## SSL Listener: 8883, 127.0.0.1:8883, ::1:8883
listener.ssl.external = 8883

listener.ssl.external.keyfile = etc/certs/key.pem

listener.ssl.external.certfile = etc/certs/cert.pem

listener.ssl.external.cacertfile = etc/certs/cacert.pem

listener.ssl.external.verify = verify_peer

listener.ssl.external.fail_if_no_peer_cert = true

## Use the CN or DN value from the client certificate as a username.
## Notice: 'verify' should be configured as 'verify_peer'
## 使用证书 CN 作为 MQTT 客户端用户名
listener.ssl.external.peer_cert_as_username = cn
```

`mosquitto_sub` 测试 TLS 连接:

```
mosquitto_sub -t topic -p 8883 --cafile etc/certs/cacert.pem --cert etc/certs/client-cert.pem --key etc/certs/client-key.pem  --insecure
```

EMQ 控制台查看连接`用户名`是否替换为证书`CN`。


## Client --TLS--> HAProxy --TCP--> EMQ

HAProxy 通过Proxy Protocol V2 传递客户端证书 CN 到 EMQ。

HAProxy 监听18883端口，并开启 SSL 设置:

```
listen mqtt-ssl
    bind *:18883 ssl ca-file /opt/emqttd/etc/certs/emq-ca.pem verify required crt /usr/local/etc/openssl/certs/emq.pem no-sslv3 no-tlsv10 no-tlsv11 no-tls-tickets
    mode tcp
    maxconn 50000
    timeout client 600s
    default_backend emq_nodes

backend emq_nodes
    mode tcp
    balance source
    timeout server 50s
    timeout check 5000
    server emq1 127.0.0.1:1883 check-send-proxy send-proxy-v2-ssl-cn
```

EMQ 开启 TCP 1883 端口 `proxy_protocol_timeout`, `peer_cert_as_username` 配置:

```
## Proxy Protocol V1/2
listener.tcp.external.proxy_protocol = on

listener.tcp.external.proxy_protocol_timeout = 3s

### Use the PP2_SUBTYPE_SSL_CN from Proxy Protocol V2 as a username.
listener.tcp.external.peer_cert_as_username = cn
```

`mosquitto_sub` TLS 连接 HAProxy 18883 端口:

```
mosquitto_sub -t topic -p 18883 --cafile etc/certs/cacert.pem --cert etc/certs/client-cert.pem --key etc/certs/client-key.pem  --insecure
```

EMQ 控制台查看连接`用户名`是否替换为证书`CN`。

