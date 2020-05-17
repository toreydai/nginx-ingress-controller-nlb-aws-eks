# nginx-ingress-controller-nlb-aws-eks
本项目用于在AWS EKS上部署Nginx Ingress Controller，并通过AWS NLB暴露Ningx Ingress。
本项目中所使用的文件均存放于resources目录下，使用时Git clone到本地即可。
参考来源：https://github.com/cornellanthony/nlb-nginxIngress-eks


前提条件：

```
EKS Node Group安全组允许TCP 10254入站
```

设置别名

```
alias k="kubectl"
```

在集群中创建必须的资源

```
k create -f resources/mandatory.yaml

# 查看Deployment
k get deployment -n ingress-nginx
# 参考输出
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
nginx-ingress-controller   1/1     1            1           11s

# 查看Pod
k get pods -n ingress-nginx
# 参考输出
NAME                                        READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-544b9d5c58-r9zr8   1/1     Running   0          18s

k describe pod <nginx-ingress-controller-xxx> -n ingress-nginx
# 参考输出
Events:
  Type    Reason     Age   From                                                        Message
  ----    ------     ----  ----                                                        -------
  Normal  Scheduled  33s   default-scheduler                                           Successfully assigned ingress-nginx/nginx-ingress-controller-544b9d5c58-r9zr8 to ip-192-168-64-244.cn-northwest-1.compute.internal
  Normal  Pulled     32s   kubelet, ip-192-168-64-244.cn-northwest-1.compute.internal  Container image "048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/quay/kubernetes-ingress-controller/nginx-ingress-controller:0.32.0" already present on machine
  Normal  Created    32s   kubelet, ip-192-168-64-244.cn-northwest-1.compute.internal  Created container nginx-ingress-controller
  Normal  Started    32s   kubelet, ip-192-168-64-244.cn-northwest-1.compute.internal  Started container nginx-ingress-controller
```

为Nginx Ingress Controller创建NLB

```
k create -f resources/nlb-service.yaml

# 查看nlb-service状态
k get svc -n ingress-nginx
# 参考输出
NAME            TYPE           CLUSTER-IP       EXTERNAL-IP                                                                             PORT(S)                      AGE
ingress-nginx   LoadBalancer   10.100.180.140   xxx.elb.cn-northwest-1.amazonaws.com.cn   80:31943/TCP,443:30260/TCP   28s
```

创建服务资源：

```
k create -f resources/apple.yaml
k create -f resources/banana.yaml

# 查看app & banana Pod
k get pods
# 参考输出
NAME                        READY   STATUS    RESTARTS   AGE
apple-app                   1/1     Running   0          30m
banana-app                  1/1     Running   0          27m

# 查看app & banana Service
k get svc
# 参考输出
NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
apple-service       ClusterIP   10.100.30.73     <none>        5678/TCP       32m
banana-service      ClusterIP   10.100.192.204   <none>        5678/TCP       28m
```
创建SSL和Secret

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=test.com/O=test.com"

k create secret tls tls-secret --key tls.key --cert tls.crt
```
创建Ingress资源

```
k create -f resources/example-ingress.yaml

# 查看Ingress状态
k get ingress
# 参考输出
example-ingress   test.com   xxx-e46740de02f66714.elb.cn-northwest-1.amazonaws.com.cn   80, 443   24m
```
测试Ningx Ingress

```
# 获取NLB DNS
NLB=$(kubectl get service ingress-nginx -n ingress-nginx -o json | jq -r '.status.loadBalancer.ingress[].hostname')

# 访问apple 
curl -i -k -H "Host:test.com" ${NLB}/apple
# 参考输出
HTTP/1.1 200 OK
Server: nginx/1.17.10
Date: Sun, 17 May 2020 13:33:57 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 6
Connection: keep-alive
X-App-Name: http-echo
X-App-Version: 0.2.3

apple

# 访问banana
curl -i -k -H "Host:test.com" ${NLB}/banana
# 参考输出
HTTP/1.1 200 OK
Server: nginx/1.17.10
Date: Sun, 17 May 2020 13:34:03 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 7
Connection: keep-alive
X-App-Name: http-echo
X-App-Version: 0.2.3

banana
```
清理环境

```
k delete -k resoures/
k delete secrest tls-secret
```
