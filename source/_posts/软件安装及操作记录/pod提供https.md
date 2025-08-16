---
title: Pod 提供 https 服务
---
创建tls目录后续存放私钥文件、证书

```bash
[root@master ~]# mkdir tls
```

生成私钥

```
[root@master ~]# openssl genrsa -out tls/cert.key 2048
```

`openssl genrsa`：使用 OpenSSL 工具生成 RSA 类型的私钥。
`-out tls/cert.key`：将生成的私钥保存为 tls/cert.key 文件。
2048：表示密钥长度为 2048 位，这是目前推荐的安全强度。

生成自签名证书（Self-signed Certificate）

```bash
[root@master ~]# openssl req -new -x509 -key tls/cert.key -out tls/cert.crt \
                       -subj "/C=CN/ST=BJ/L=BJ/O=Tedu/OU=NSD/CN=localhost"
```

🔹 openssl req
表示你要生成一个 X.509 证书请求（CSR）或直接生成证书。
🔹 -new
表示生成一个新的证书请求（这里配合 -x509 使用，直接生成证书）。
🔹 -x509
表示直接输出一个 自签名证书，而不是 CSR（证书签名请求）。
🔹 -key tls/cert.key
指定使用的私钥文件，即上一步生成的 cert.key。
🔹 -out tls/cert.crt
输出证书文件路径为 tls/cert.crt。
🔹 -subj "/C=CN/ST=BJ/L=BJ/O=Tedu/OU=NSD/CN=localhost"

| 字段 | 含义                        | 示例                      |
| ---- | --------------------------- | ------------------------- |
| `C`  | Country（国家代码）         | CN = China                |
| `ST` | State（省份）               | BJ = Beijing              |
| `L`  | Locality（城市）            | BJ = Beijing              |
| `O`  | Organization（组织名）      | Tedu（公司或单位名称）    |
| `OU` | Organizational Unit（部门） | NSD（网络服务部）         |
| `CN` | Common Name（通用名）       | localhost（域名或主机名） |

创建secret、pod

```bash
[root@master ~]# kubectl create secret generic webcert --from-file=tls
# 导出 secret 资源对象文件
[root@master ~]# kubectl get secrets webcert -o yaml |tee webcert.yaml
[root@master ~]# vim web.yaml
  ---
  apiVersion: v1
  kind: Pod
  metadata:
  name: web
  spec:
  terminationGracePeriodSeconds: 0
  restartPolicy: Always
  containers:
  - name: nginx
    image: myos:nginx
  [root@master ~]# kubectl apply –f web.yaml
  pod/web created
```

修改nginx.conf启用https

```bash
[root@master ~]# kubectl cp web:/usr/local/nginx/conf/nginx.conf nginx.conf
[root@master ~]# vim nginx.conf
... ...
    server {
        listen       443 ssl;
        server_name  localhost;
        ssl_certificate      tls/cert.crt;  #指定证书文件路径
        ssl_certificate_key  tls/cert.key;  #指定私钥文件路径
        ssl_session_cache    shared:SSL:1m; #共享内存缓存区大小
        ssl_session_timeout  5m;  #SSL会话缓存的有效时间
        ssl_ciphers  HIGH:!aNULL:!MD5; #使用高强度加密算法 ！禁用
        ssl_prefer_server_ciphers  on; #优先选择加密套件
        location / {
            root   html;
            index  index.html index.htm;
        }
    }
... ...
```

nginx.conf、tls做临时挂载

```bash
# 使用命令创建 configMap
[root@master ~]# kubectl create configmap nginx-ssl --from-file=nginx.conf
configmap/nginx-ssl created
[root@master ~]# vim web.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  terminationGracePeriodSeconds: 0
  restartPolicy: Always
  volumes:
  - name: webconf
    configMap:
      name: nginx-ssl
  - name: ssl
    secret:
      secretName: webcert
      items:
      - key: cert.key
        path: cert.key
        mode: 0400
      - key: cert.crt
        path: cert.crt
        mode: 0444
  containers:
  - name: nginx
    image: myos:nginx
    volumeMounts:
    - name: webconf
      subPath: nginx.conf
      mountPath: /usr/local/nginx/conf/nginx.conf
    - name: ssl   ## 将secret中的所有挂载到tls
      mountPath: /usr/local/nginx/conf/tls
      readOnly: true
[root@master ~]# kubectl replace --force -f web.yaml
pod "web" deleted
pod/web created
[root@master ~]# kubectl get pods -o wide
NAME   READY   STATUS    RESTARTS   AGE     IP              NODE
Web    1/1     Running   0           8s      10.244.1.19    node-0001
[root@master ~]# curl -k https://10.244.1.19
Nginx is running !  #-k 忽略ssl验证
```
