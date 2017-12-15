
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


## Client --TLS--HAProxy--TCP--> EMQ

