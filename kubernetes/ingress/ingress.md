### ingress解析

Ingress 是从Kubernetes集群外部访问集群内部服务的入口。将集群外部的http或https流量导入集群中的service中。转发路由是有定义的ingress类型资源时的rules规则设定的。

        外部访问流程：

                internet
                    |
               [ Ingress ]
               --|-----|--
               [ Services ]
ingress类型的资源可以配置为，可提供具体某服务的虚拟主机（常见的有基于URLs、基于主机名称的虚拟主机）。

ingress不会暴露任意端口或协议。可以使用Service.Type=NodePort or Service.Type=LoadBalancer，暴露到集群边界外。

### 先决条件
在使用ingresss类型的资源之前，必须需要有个ingress controllerl来实现ingress，单纯的创建ingresss资源没有任何意义。
Ingress Controller是一个守护进程，部署为Kubernetes Pod，它监视apiserver的/ingresses端点以更新Ingress资源。它的工作是满足对Ingress的要求。
与其他类型的控制器不同，其他类型的控制器是作为kube-controller-manager二进制文件的一部分运行，在集群启动时自动启动。

#### 这里我们选择nginx ingress  controller。
部署nginx controller方法：
        
        首先在mater节点上，部署一个ingress controller，建议使用通用部署命令：
        kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
查看创建nginx controller pods信息，命令：

        [root@docker1:~ ]# kubectl  get pods -n ingress-nginx
        NAME                                        READY   STATUS    RESTARTS   AGE
        nginx-ingress-controller-689498bc7c-rqfxj   1/1     Running   0          146m
