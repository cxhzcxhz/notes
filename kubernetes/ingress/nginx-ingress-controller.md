### 部署nginx controller

    service资源只能实现4层代理（流量入口）。
    ingress资源可以实现4、7层代理（流量入口）。
使用ingress资源，实现7层代理访问，大概分几步实现：（参考）

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
