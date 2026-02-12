---
title: jfrog分发到harbor-500错误
date: 2025-8-15 22:43:05
tags: [jfrog, s3, harbor]
---

### 现象及背景

> 上海的机房(jfrog)需要分发制品到北京(harbor)
> 分发出现报错

![](./replica-harbor.png)


### 分析


#### 服务日志
> 很明显的服务报错

```
time="2025-08-14T14:30:15.303948291Z" level=error msg="response completed with error" auth.user.name=devops err.code=unknown err.detail="s3aws: SerializationError: failed to decode S3 XML error response
        status code: 404, request id: , host id:
caused by: expected element type <Error> but have <html>" err.message="unknown error" go.version=go1.23.4 http.request.host="harbor-core:80" http.request.id=b91d15b4-9743-4799-851a-1d573b5afa18 http.request.method=HEAD http.request.remoteaddr=100.116.135.20 http.request.uri="/v2/stc-docker-repo_127/stc-docker-repo/stc-docker/02-iflydata-dev/iddp/v3.8.2/1002/dp-hirule/blobs/sha256:5ad559c5ae16b8980924ceae7f7662d07740debd4467db19e69339926ec8f255" http.request.useragent=harbor-registry-client http.response.contenttype="application/json; charset=utf-8" http.response.duration=90.393743ms http.response.status=500 http.response.written=123 vars.digest="sha256:5ad559c5ae16b8980924ceae7f7662d07740debd4467db19e69339926ec8f255" vars.name="stc-docker-repo_127/stc-docker-repo/stc-docker/02-iflydata-dev/iddp/v3.8.2/1002/dp-hirule
```


#### 稳定重现
docker push  harbor-bj.xxxx.xxxx.com/stc-docker-repo_127/busybox:1.32

- 抓包分析: 错误包

```
GET /harbor/docker/registry/v2/repositories/stc-docker-repo_127/busybox/_layers/sha256/f531cdc67389c92deac44e019e7a1b6fba90d1aaa58ae3e8192f0e0eed747152/link HTTP/1.1
Host: 172.xx.xxx.206:9020
User-Agent: aws-sdk-go/1.15.11 (go1.23.4; linux; amd64)
Authorization: AWS4-HMAC-SHA256 Credential=BGZNN74F1POJ5NMB7PKM/20250815/beiji-senghua-1/s3/aws4_request, SignedHeaders=host;range;x-amz-content-sha256;x-amz-date, Signature=sfdfsdfsfd
Range: bytes=0-
X-Amz-Content-Sha256: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
X-Amz-Date: 20250815T062519Z

HTTP/1.1 404 Not Found
Server: openresty/1.15.8.2
Content-Type: text/html
Content-Length: 159
Connection: keep-alive
Date: Fri, 15 Aug 2025 06:25:19 GMT

<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>openresty/1.15.8.2</center>
</body>
</html>
```


#### 结合错误包 + 报错信息  
> 再看下抓包内容
> s3返回的格式不对导致, sdk期望解析xml但是返回了 html格式 


#### 直接请求s3服务

```
GET /harbor/docker/registry/v2/repositories/stc-docker-repo_127/nginx/_layers/sha256/0368fd46e3c6d237d81390ff086f93aee216df5cfa814041a491453fb0932a12/link HTTP/1.1
Host: 10.xxx.xxx.200
User-Agent: aws-sdk-go/1.15.11 (go1.23.4; linux; amd64)
Authorization: AWS4-HMAC-SHA256 Credential=BGZNN74F1POJ5NMB7PKM/20250821/beiji-senghua-1/s3/aws4_request, SignedHeaders=host;range;x-amz-content-sha256;x-amz-date, Signature=b59415228f19c8eeb18cd9815243e407c1e7b5fb87f2841941d684db223ff4cd
Range: bytes=0-
X-Amz-Content-Sha256: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
X-Amz-Date: 20250821T043224Z

HTTP/1.1 404 Not Found
content-length: 219
x-amz-request-id: tx000000000000001bad898-0068a6a158-4cbc9e5-default
accept-ranges: bytes
content-type: application/xml
date: Thu, 21 Aug 2025 04:32:24 GMT

<?xml version="1.0" encoding="UTF-8"?><Error><Code>NoSuchKey</Code><BucketName>harbor</BucketName><RequestId>tx000000000000001bad898-0068a6a158-4cbc9e5-default</RequestId><HostId>4cbc9e5-default-default</HostId></Error>
```

#### 网关地址改为s3服务地址

> 会有新的报错 。。  

![](./docker-push.png)

#### 403报错分析

>  为啥会有403报错
1. 确认  harbor-nginx 无403
2. 确认  harbor-registry 无 403
3. 确认  s3无403  通过在 registry 添加一个tcpdump 的sidecar 转包确认的 
一顿排查 无果
搜索403报错
直接找到解决方案   https://github.com/goharbor/harbor/issues/5656#issuecomment-1875146709

#### 深入理解为啥会有403？ 以及为什么之前的抓包抓不到呢
> 403报错
> 通过harbor代码分析 使用的是  harbor v2.12.2

```yaml
// /etc/registry/config.yml

storage:
  s3:
    xxx
  redirect:
    disable: {{ $storage.disableredirect }}  # 默认是false 也就是允许redirect
```

```golang
// src/registryctl/config/connfig.go
import (
    "fmt"
    "os"

    "github.com/docker/distribution/configuration"
    storagedriver "github.com/docker/distribution/registry/storage/driver"
    "github.com/docker/distribution/registry/storage/driver/factory"
    yaml "gopkg.in/yaml.v2"

    "github.com/goharbor/harbor/src/lib/log"
)

// setStorageDriver set the storage driver according the registry's configuration.
func (c *Configuration) setStorageDriver() error {
    # 这里c.RegistryConfig 就是/etc/registry/config.yml
    fp, err := os.Open(c.RegistryConfig)
    if err != nil {
        return err
    }
    defer fp.Close()
    rConf, err := configuration.Parse(fp)
    if err != nil {
        return fmt.Errorf("error parsing registry configuration %s: %v", c.RegistryConfig, err)
    }
    storageDriver, err := factory.Create(rConf.Storage.Type(), rConf.Storage.Parameters())
    if err != nil {
        return err
    }
    c.StorageDriver = storageDriver
    return nil
}
```

```golang
    // /root/go/pkg/mod/github.com/distribution/distribution@v2.8.2+incompatible/registry/handlers/app.go
    
    var redirectDisabled bool
    if redirectConfig, ok := config.Storage["redirect"]; ok {
        v := redirectConfig["disable"]
        switch v := v.(type) {
        case bool:
            redirectDisabled = v
        default:
            panic(fmt.Sprintf("invalid type for redirect config: %#v", redirectConfig))
        }
    }
    if redirectDisabled {
        # 如果禁用  这里其实没有设置  
        dcontext.GetLogger(app).Infof("backend redirection disabled")
    } else {
        # 默认 disableRedirect -> false , 也就是允许redirect 
        options = append(options, storage.EnableRedirect)
    }

    if !config.Validation.Enabled {
        config.Validation.Enabled = !config.Validation.Disabled
    }
```

```golang
///root/go/pkg/mod/github.com/distribution/distribution@v2.8.2+incompatible/registry/storage/blobserver.go 

type blobServer struct {
    driver   driver.StorageDriver
    statter  distribution.BlobStatter
    pathFn   func(dgst digest.Digest) (string, error)
    redirect bool // allows disabling URLFor redirects    # 如果不设置默认是 false
}

...
    # 在允许redirect的时候， 不会把内容直接返回给docker client 
    if bs.redirect {
        redirectURL, err := bs.driver.URLFor(ctx, path, map[string]interface{}{"method": r.Method})
        switch err.(type) {
        case nil:
            // Redirect to storage URL.
            # 其实就是把s3 url发送给docker客户端， 这就是为啥 我们在registry服务抓包也看不到403请求的原因
            http.Redirect(w, r, redirectURL, http.StatusTemporaryRedirect)
            return err

        case driver.ErrUnsupportedMethod:
            // Fallback to serving the content directly.
        default:
            // Some unexpected error.
            return err
        }
    }
```

- 结合本地抓包 
s3给的 redirectUrl 其实在本地客户端直接请求是403的 
tcpdump host 10.xxx.xxx.200

```
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
20:36:05.546933 IP maint-linux.57208 > 10.xxx.xxx.200.http: Flags [S], seq 4118211652, win 29200, options [mss 1460,sackOK,TS val 2873180208 ecr 0,nop,wscale 7], length 0
20:36:05.568342 IP 10.xxx.xxx.200.http > maint-linux.57208: Flags [S.], seq 3017494843, ack 4118211653, win 28960, options [mss 1460,sackOK,TS val 3721072482 ecr 2873180208,nop,wscale 7], length 0
20:36:05.568382 IP maint-linux.57208 > 10.xxx.xxx.200.http: Flags [.], ack 1, win 229, options [nop,nop,TS val 2873180230 ecr 3721072482], length 0
20:36:05.568555 IP maint-linux.57208 > 10.xxx.xxx.200.http: Flags [P.], seq 1:614, ack 1, win 229, options [nop,nop,TS val 2873180230 ecr 3721072482], length 613: HTTP: HEAD /harbor/docker/registry/v2/blobs/sha256/f5/f531cdc67389c92deac44e019e7a1b6fba90d1aaa58ae3e8192f0e0eed747152/data?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=BGZNN74F1POJ5NMB7PKM%2F20250825%2Fbeiji-senghua-1%2Fs3%2Faws4_request&X-Amz-Date=20250825T123605Z&X-Amz-Expires=1200&X-Amz-SignedHeaders=host&X-Amz-Signature=8ea6cfe746941a23ddfbcf51ca5e49054e342ccc28aad20e9772f34154f87808 HTTP/1.1
20:36:05.590200 IP 10.xxx.xxx.200.http > maint-linux.57208: Flags [P.], seq 1:227, ack 614, win 236, options [nop,nop,TS val 3721072504 ecr 2873180230], length 226: HTTP: HTTP/1.1 403 Forbidden
20:36:05.590242 IP maint-linux.57208 > 10.xxx.xxx.200.http: Flags [.], ack 227, win 237, options [nop,nop,TS val 2873180252 ecr 3721072504], length 0
20:36:05.590250 IP 10.xxx.xxx.200.http > maint-linux.57208: Flags [F.], seq 227, ack 614, win 236, options [nop,nop,TS val 3721072504 ecr 2873180230], length 0
20:36:05.590379 IP maint-linux.57208 > 10.xxx.xxx.200.http: Flags [F.], seq 614, ack 228, win 237, options [nop,nop,TS val 2873180252 ecr 3721072504], length 0
20:36:05.612111 IP 10.xxx.xxx.200.http > maint-linux.57208: Flags [.], ack 615, win 236, options [nop,nop,TS val 3721072525 ecr 2873180252], length 0
20:36:06.504403 IP maint-linux.57240 > 10.xxx.xxx.200.http: Flags [S], seq 3693280526, win 29200, options [mss 1460,sackOK,TS val 2873181166 ecr 0,nop,wscale 7], length 0
20:36:06.526380 IP 10.xxx.xxx.200.http > maint-linux.57240: Flags [S.], seq 3157260753, ack 3693280527, win 28960, options [mss 1460,sackOK,TS val 3721073440 ecr 2873181166,nop,wscale 7], length 0
20:36:06.526425 IP maint-linux.57240 > 10.xxx.xxx.200.http: Flags [.], ack 1, win 229, options [nop,nop,TS val 2873181188 ecr 3721073440], length 0
20:36:06.526596 IP maint-linux.57240 > 10.xxx.xxx.200.http: Flags [P.], seq 1:614, ack 1, win 229, options [nop,nop,TS val 2873181188 ecr 3721073440], length 613: HTTP: HEAD /harbor/docker/registry/v2/blobs/sha256/f5/f531cdc67389c92deac44e019e7a1b6fba90d1aaa58ae3e8192f0e0eed747152/data?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=BGZNN74F1POJ5NMB7PKM%2F20250825%2Fbeiji-senghua-1%2Fs3%2Faws4_request&X-Amz-Date=20250825T123606Z&X-Amz-Expires=1200&X-Amz-SignedHeaders=host&X-Amz-Signature=fa02d7f955bc82c7a357c641ecf98bda157b281be74b8f6c8acaa99d7eb45548 HTTP/1.1
20:36:06.548162 IP 10.xxx.xxx.200.http > maint-linux.57240: Flags [P.], seq 1:227, ack 614, win 236, options [nop,nop,TS val 3721073462 ecr 2873181188], length 226: *HTTP: HTTP/1.1 403 Forbidden*
20:36:06.548194 IP maint-linux.57240 > 10.xxx.xxx.200.http: Flags [.], ack 227, win 237, options [nop,nop,TS val 2873181210 ecr 3721073462], length 0
20:36:06.548200 IP 10.xxx.xxx.200.http > maint-linux.57240: Flags [F.], seq 227, ack 614, win 236, options [nop,nop,TS val 3721073462 ecr 2873181188], length 0
20:36:06.548323 IP maint-linux.57240 > 10.xxx.xxx.200.http: Flags [F.], seq 614, ack 228, win 237, options [nop,nop,TS val 2873181210 ecr 3721073462], length 0
20:36:06.569492 IP 10.xxx.xxx.200.http > maint-linux.57240: Flags [.], ack 615, win 236, options [nop,nop,TS val 3721073483 ecr 2873181210], length 0
```

结论
原因2个
1. s3网关返回内容和直连s3服务返回内容不一致 导致harbor解析出现异常
2. harbor本身的默认配置 在连s3服务时 会触发  https://github.com/goharbor/harbor/issues/5656

---
解决 

- 从s3网关切到s3服务

> 并将xml返回不符合预期, 反馈到存储团队, 跟进存储团队修复

- s3配置

```yaml
storage:
  redirect:
    disable: false -> true
```



##### tips: pod抓包
> 针对场景： 非root运行时需要通过sidecar来抓包
> 弊端: 需要重启pod

```yaml
    - name: tcpdump
      image: http://artifacts.xxxx.com/nicolaka/netshoot:latest
      command:
        - /bin/sleep
        - infinity
      resources:
        limits:
          cpu: 500m
          memory: 1Gi
        requests:
          cpu: 500m
          memory: 1Gi
      lifecycle: {}
      imagePullPolicy: IfNotPresent
      securityContext:
        capabilities:
          add:
            - NET_RAW
            - NET_ADMIN
        runAsUser: 0
        runAsNonRoot: false
```