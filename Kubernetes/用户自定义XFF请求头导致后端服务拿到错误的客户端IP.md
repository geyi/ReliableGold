# 用户自定义XFF请求头导致后端服务拿到错误的客户端IP

## **原因**
问题的根本原因是ingress控制器默认配置为信任所有的请求IP，导致Nginx会将把XFF请求头最左边的IP地址作为`$remote_addr`。相关配置如下（已省略无关的配置）：
```
http {
  map '' $the_real_ip {

    default          $remote_addr;

  }

  real_ip_header      X-Forwarded-For;

  real_ip_recursive   on;

  set_real_ip_from    0.0.0.0/0;


  server {
    listen       80;
    server_name  ingress.domain.com;

    location / {
      proxy_set_header X-Forwarded-For        $the_real_ip;
      proxy_pass http://upstream_balancer;
    }
  }
}
```

## **问题分析与解决方案**
### **核心逻辑**
1. **`ngx_http_realip_module` 模块的触发条件**  
   - `ngx_http_realip_module` 模块仅当 **直接连接的客户端 IP**（即上游代理的 IP）在 `set_real_ip_from` 的信任范围内时，才会生效。
   - 如果直接连接的客户端 IP **未被信任**，即使请求头中包含 `X-Forwarded-For` 等信息，Nginx 也会 **忽略这些头**，`$remote_addr` 保持为直接连接的客户端 IP。

2. **关键配置指令**
   | 指令                | 作用                                                                          |
   |---------------------|-------------------------------------------------------------------------------|
   | `set_real_ip_from`  | 用于定义可信任的IP地址或网段（只有来自可信任IP的请求才会触发`ngx_http_realip_module`模块的逻辑） |
   | `real_ip_header`    | 如果请求的IP是可信任的，Nginx会将客户端地址和可选端口更改为`real_ip_header`指令定义的请求头（如 `X-Forwarded-For` 或 `X-Real-IP`）中发送的地址和端口 |
   | `real_ip_recursive` | 该指令的作用是控制Nginx如何从一个IP链中选择真实的客户端IP。在关闭状态下（默认配置），Nginx会从`real_ip_header`指定的请求头（如X-Forwarded-For）中取最右边的IP地址作为客户端IP。而开启后，Nginx会从右往左遍历IP链，跳过所有在`set_real_ip_from`中定义的信任IP，直到找到第一个不在信任列表中的IP，将其作为真实客户端IP |
   | `$proxy_add_x_forwarded_for` | “X-Forwarded-For”客户端请求头字段会被附加上`$remote_addr`变量，并用逗号分隔。如果客户端请求标头中不存在“X-Forwarded-For”字段，则`$proxy_add_x_forwarded_for`变量等于`$remote_addr`变量 |

> 官方说明可以参考：https://nginx.org/en/docs/http/ngx_http_realip_module.html

### **通过以下示例对问题进行分析**
第一个Nginx IP地址为192.168.249.157，配置如下（已省略无关的配置）：
```
http {
  set_real_ip_from 192.168.228.0/24;
  set_real_ip_from 127.0.0.1;

  real_ip_header X-Forwarded-For;

  server {
    listen       80;
    server_name  your.domain.com;

    proxy_set_header X-Forwarded-For $remote_addr;

    location ^~ /path/ {
      client_max_body_size 8m;
      proxy_set_header X-Forwarded-Host $host;
      proxy_set_header X-Forwarded-Server $host;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_pass http://ingress.domain.com/server/;
    }
  }
}
```

第二个Nginx（ingress controller） IP地址为10.240.1.173，配置如下：
```
http {
  map '' $the_real_ip {

    default          $remote_addr;

  }

  real_ip_header      X-Forwarded-For;

  real_ip_recursive   on;

  set_real_ip_from    0.0.0.0/0;


  server {
    listen       80;
    server_name  ingress.domain.com;

    location / {
      proxy_set_header X-Forwarded-For        $the_real_ip;
      proxy_pass http://upstream_balancer;
    }
  }
}
```

假设客户端的IP地址为10.240.20.49，设置了请求头`X-Forwarded-For: 127.0.0.1`
1. 客户端发送X-Forwarded-For: 127.0.0.1，IP为10.240.20.49到第一个Nginx。

2. 第一个Nginx的real_ip模块检查客户端IP10.240.20.49是否在set_real_ip_from中。由于不在，$remote_addr保持为10.240.20.49。

3. 在location中，X-Forwarded-For被设置为$proxy_add_x_forwarded_for，即原始头127.0.0.1加上客户端的IP，成为127.0.0.1, 10.240.20.49。

4. 第二个Nginx接收到请求，X-Forwarded-For头为127.0.0.1, 10.240.20.49，其来源IP是第一个Nginx的192.168.249.157。

5. 第二个Nginx的real_ip模块处理X-Forwarded-For头。由于set_real_ip_from是0.0.0.0/0，所有IP都被信任。real_ip_recursive on会从右到左寻找第一个不在可信列表中的IP，但因为所有IP都被信任，最终会以127.0.0.1为客户端IP，即X-Forwarded-For被设置为127.0.0.1。

以下是各个节点日志中输出的`$remote_addr`和`$http_x_forwarded_for`
| Nginx/Service   | remote_addr   | http_x_forwarded_for    |
|-----------------|---------------|-------------------------|
| 192.168.249.157 | 10.240.20.49  | 127.0.0.1               |
| 10.240.1.173    | 127.0.0.1     | 127.0.0.1, 10.240.20.49 |
| Service         | 10.244.50.128 | 127.0.0.1               |

通过上述分析可知，ingress控制器中错误的将`set_real_ip_from`设置为过大的范围（如0.0.0.0/0），这会导致所有IP都被信任，`real_ip_recursive`无法正确工作，从而无法提取真实IP。正确的做法是只信任上游的代理IP，这样Nginx可以正确跳过这些信任的代理IP，找到客户端的真实IP。

### **更改ingress控制器的配置**
1. 查找对应的ConfigMap
   ```shell
   [kuaidi@k8s-dev-master ~]$ sudo kubectl get configmap -n ingress-nginx 
   NAME                              DATA   AGE
   ingress-controller-leader-nginx   2      4y24d
   istio-ca-root-cert                1      3y317d
   nginx-configuration               0      4y24d
   tcp-services                      0      4y24d
   udp-services                      0      4y24d
   ```
2. 编辑ConfigMap，添加proxy-real-ip-cidr配置
   ```shell
   [kuaidi@k8s-dev-master ~]$ sudo kubectl edit configmap nginx-configuration -n ingress-nginx
   configmap/ingress-controller-leader-nginx edited
   ```
   ```yaml
   apiVersion: v1
   data:
     # 添加或修改以下配置，可以指定多个 CIDR，用逗号分隔
     proxy-real-ip-cidr: 192.168.249.0/24
   ```
   > ConfigMap配置可以参考：https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#proxy-real-ip-cidr
3. ~~重新启动ingress控制器~~（修改配置后自动生效）
   ```shell
   [kuaidi@k8s-dev-master ~]$ sudo kubectl rollout restart deployment nginx-ingress-controller -n ingress-nginx
   deployment.apps/nginx-ingress-controller restarted
   ```