# Traefik 入手及简单配置

## 日志

``` toml
[accessLog]

# Sets the file path for the access log. If not specified, stdout will be used.
# Intermediate directories are created if necessary.
#
# Optional
# Default: os.Stdout
#
filePath = "./traefik-access.json"

# Format is either "json" or "common".
#
# Optional
# Default: "common"
#
format = "json"
```

日志文件配置为 `json` 格式，方便调试。同时，强烈推荐 [jq](https://github.com/stedolan/jq)，一款 linux 下解析 json 的工具。

以下是两个常用的命令，统计某个站点的请求以及响应时间。不过最好建议有专门的日志系统去处理，可以获取更完善的，更定制化的信息。另外，traefik 无法查看请求的 body。

``` shell
# 筛选特定站点的请求
cat traefik-access.json | jq 'select(.["RequestHost"] == "shici.xiange.tech") | {RequestPath, RequestHost, DownstreamStatus, "request_User-Agent", OriginDuration}'


# 筛选大于 300ms 的接口
cat traefik-access.json | jq 'select(.["RequestHost"] == "shici.xiange.tech" and .OriginDuration > 300000000) | {RequestPath, RequestHost, DownstreamStatus,
"request_User-Agent", OriginDuration, DownstreamContentSize}'
```

## http

http 配置在 `entryPoints` 中，暴露出80端口。开启 `gzip` 压缩，使用 `compress = true` 来配置。

``` toml
[entryPoints]
    [entryPoints.http]
    address = ":80"
    compress = true

    # 如果配置了此项，会使用 307 跳转到 https 
    [entryPoints.http.redirect]
    entryPoint = "https"
```

考虑到隐私以及安全，这些地址可以配置 `Basic Auth`，`Digest Auth` 或者 `WhiteList`，或者直接搭建 VPN，在内网内进行访问。

更多文档查看 [Traefik entrypoints](https://docs.traefik.io/configuration/entrypoints/)。

## https

使用 `Let's Encrypt` 安装证书后，在 `entryPoints.https.tls.certificats` 中指定证书位置。

``` toml
[entryPoints]
    [entryPoints.https]
    address = ":443"
    compress = true

        [[entryPoints.https.tls.certificates]]
            certFile = "/etc/letsencrypt/live/xiange.tech/fullchain.pem"
            keyFile = "/etc/letsencrypt/live/xiange.tech/privkey.pem"
```

另外，traefik 默认开启 http2。

## Docker

traefik 会监听 `docker.sock`，根据容器的 label 进行配置。容器的端口号需要暴露出来，但是不需要映射到 Host。因为 traefik 可以通过 `docker.sock` 找到 container 的 IP 地址以及端口号，无需使用 `docker-proxy` 转发到 Host。

``` yaml
version: '3'
services:
  whoami:
    image: your-frontend-server-image
    labels:
      - "traefik.frontend.rule=Host:demo.xiange.tech"

  api:
    image: your-api-server-image
    expose: 80
    labels:
      # 同域配置， /api 走server
      - "traefik.frontend.rule=Host:demo.xiange.tech;PathPrefix:/api"
```

## 负载均衡

如果使用docker，对一个容器进行扩展后，traefik 会自动做负载均衡，而 nginx 需要手动干预。

``` yaml
version: '3'
services:
  whoami:
    image: emilevauge/whoami
    labels:
      - "traefik.prod.frontend.rule=Host:whoami.xiange.tech"
      - "traefik.dev.frontend.rule=Host:whoami.xiange.me"
```

手动扩展为3个实例，可以自动实现负载均衡。实现效果可以直接访问 [whoami.xiange.tech](https://whoami.xiange.tech)，每次通过 WRR 策略分配到不同的容器处理，可以通过 Hostname 和 IP 字段看出。

```
docker-compose up whoami=3
```

## 手动配置

当然，以上都是基于 docker，那如何像 nginx 一样配置服务呢。如把 8500 的端口配置到 `consul.xiange.me`。

``` toml
[file]
    [backends]
        [backends.consul]
            [backends.consul.servers]
                [backends.consul.servers.website]
                url = "http://0.0.0.0:8500"
                weight = 1

   [frontends]
        [frontends.consul]
        entryPoints = ["http"]
        backend = "consul"
            [frontends.consul.routes]
                [frontends.consul.routes.website]
                rule = "Host:consul.xiange.me"

                # 可以配置多个域名
                [frontends.consul.routes.website2]
                rule = "Host:config.xiange.me"
```
