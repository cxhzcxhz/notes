### 部署nginx controller

    service资源只能实现4层代理（流量入口）。
    ingress资源可以实现4、7层代理（流量入口）。
使用ingress资源，实现7层代理访问，大概分几步实现：
（图片：https://github.com/cxhzcxhz/notes/blob/master/kubernetes/ingress/images/ingress1.png
       https://github.com/cxhzcxhz/notes/blob/master/kubernetes/ingress/images/ingress2.png）

    1、部署nginx ingress controller控制器。
    2、编写ingress类型的资源配置文件，定义使用基于主机名或url方式，提供前端访问入口（具体提供服务的虚拟主机配置文件）。
    3、创建service类型的资源，使用type为NodePort的字段定义，匹配到ingress controller pods。帮助ingress controller接入外部流量。
       （也可在创建nginx controller pods时，使用hostNetwork: true，让nginx controller共享主机网络名称空间。
        参考链接：https://kubernetes.github.io/ingress-nginx/deploy/baremetal/ ）
    4、创建service类型的资源，此service负责使用标签选择器，选择到符合要求的pods数量。
       如果后端pods数量发生变化，则service上会立即生成对象规则，并及时通知给ingress资源，ingress资源会立即将pods的地址注入到nginx controller pods中，并触发主进程重载，使用最新的配置文件。
    5、nginx ingress controller 根据ingress资源注入的配置文件，基于主机名或url，提供前端访问入口，然后使用service类型的资源，匹配后端实际的pods。
    
部署nginx controller方法：
        
        首先在mater节点上，部署一个ingress controller，建议使用通用部署命令：
        kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
查看创建nginx controller pods信息，命令：

        [root@docker1:~ ]# kubectl  get pods -n ingress-nginx
        NAME                                        READY   STATUS    RESTARTS   AGE
        nginx-ingress-controller-689498bc7c-rqfxj   1/1     Running   0          146m
查看pods的详细信息，命令；
        
        [root@docker1:~/mainfests/ingress ]# kubectl  describe pods/nginx-ingress-controller-689498bc7c-rqfxj  -n ingress-nginx
        Name:               nginx-ingress-controller-689498bc7c-rqfxj
        Namespace:          ingress-nginx
        Priority:           0
        PriorityClassName:  <none>
        Node:               docker2/192.168.12.174
        Start Time:         Mon, 24 Jun 2019 14:14:46 +0800
        Labels:             app.kubernetes.io/name=ingress-nginx
                            app.kubernetes.io/part-of=ingress-nginx
                            pod-template-hash=689498bc7c
        Annotations:        prometheus.io/port: 10254
                            prometheus.io/scrape: true
        Status:             Running
        IP:                 10.244.1.125
        Controlled By:      ReplicaSet/nginx-ingress-controller-689498bc7c
        Containers:
          nginx-ingress-controller:
            Container ID:  docker://36e75d5898809494be9c4c5033c6f1b11968b138216e73b5c717cf2ac4912d58
            Image:         quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.24.1
            Image ID:      docker-pullable://quay.io/kubernetes-ingress-controller/nginx-ingress-controller@sha256:76861d167e4e3db18f2672fd3435396aaa898ddf4d1128375d7c93b91c59f87f
            Ports:         80/TCP, 443/TCP
            Host Ports:    0/TCP, 0/TCP
            Args:
              /nginx-ingress-controller
              --configmap=$(POD_NAMESPACE)/nginx-configuration
              --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
              --udp-services-configmap=$(POD_NAMESPACE)/udp-services
              --publish-service=$(POD_NAMESPACE)/ingress-nginx
              --annotations-prefix=nginx.ingress.kubernetes.io
            State:          Running
              Started:      Mon, 24 Jun 2019 14:14:48 +0800
            Ready:          True
            Restart Count:  0
            Liveness:       http-get http://:10254/healthz delay=10s timeout=10s period=10s #success=1 #failure=3
            Readiness:      http-get http://:10254/healthz delay=0s timeout=10s period=10s #success=1 #failure=3
            Environment:
              POD_NAME:       nginx-ingress-controller-689498bc7c-rqfxj (v1:metadata.name)
              POD_NAMESPACE:  ingress-nginx (v1:metadata.namespace)
            Mounts:
              /var/run/secrets/kubernetes.io/serviceaccount from nginx-ingress-serviceaccount-token-lcs8j (ro)
        Conditions:
          Type              Status
          Initialized       True 
          Ready             True 
          ContainersReady   True 
          PodScheduled      True 
        Volumes:
          nginx-ingress-serviceaccount-token-lcs8j:
            Type:        Secret (a volume populated by a Secret)
            SecretName:  nginx-ingress-serviceaccount-token-lcs8j
            Optional:    false
        QoS Class:       BestEffort
        Node-Selectors:  <none>
        Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                         node.kubernetes.io/unreachable:NoExecute for 300s
        Events:          <none>
