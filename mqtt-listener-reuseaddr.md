
## MQTT listener reuseaddr

Multiple MQTT listeners can listen on the same port with `reuseaddr` option.

## Configuration

```
listener.tcp.external= 0.0.0.0:1883
listener.tcp.external.reuseaddr = true

listener.tcp.localhost = 127.0.0.1:1883
listener.tcp.localhost.reuseaddr = true
```

