### ConfigMap是一种kubernetes中标准资源类型。可以简写为cm。
ConfigMap可以让配置文件与镜像解耦，镜像中的配置文件，可以通过configmap注入到镜像中。

ConfigMap相当于一些列配置文件的集合，可以注入到pod的容器中使用。

    注入方式：
      1、直接将configmap当存储卷使用。
      2、使用env的valueFrom方式，引用configmap中的数据（键值对）。
创建configmap，可以使用配置文件形式，也可以通过命令行创建。

    [root@docker1:~/mainfests/volumes ]# kubectl  create cm --help
基于文件创建configmap时，key默认就是文件名，value是文件内容。
基于目录创建configmap时，目录下的文件名就是value。 除常规文件之外的任何目录条目都将被忽略（例如子目录，符号链接，
设备，管道等）

###### 基于命令创建configmap：

    [root@docker1:~/mainfests/volumes ]# kubectl  create configmap nginx-config --from-literal=nginx_port=80 --from-literal=server_name=myapp.magedu.com
    configmap/nginx-config created
    [root@docker1:~/mainfests/volumes ]# kubectl  get cm
    NAME           DATA   AGE
    nginx-config   2      6s
    [root@docker1:~/mainfests/volumes ]# kubectl  describe cm/nginx-config
    Name:         nginx-config
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>

    Data
    ====
    nginx_port:
    ----
    80
    server_name:
    ----
    myapp.magedu.com
    Events:  <none>

通过上述的详细信息，可以看到创建的键值对，
    
    key为nginx_port，value为80
    key为server_name，value为myapp.magedu.com

###### 基于配置文件形式创建configmap：

    [root@docker1:~/mainfests/configmap ]# kubectl  create configmap nginx-www --from-file=./www.conf 
    configmap/nginx-www created
    
    [root@docker1:~/mainfests/configmap ]# kubectl  describe cm/nginx-www
    Name:         nginx-www
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>

    Data
    ====
    www.conf:
    ----
    server {
      server_name myapp.magedu.com;
      listen 80;
      root /data/web/html/;
    }
    Events:  <none>

也可以通过kubectl get cm/nginx-www -o yaml 形式查看：

    [root@docker1:~/mainfests/configmap ]# kubectl get cm/nginx-www -o yaml
    apiVersion: v1
    data:
      www.conf: "server {\n\tserver_name myapp.magedu.com;\n\tlisten 80;\n\troot /data/web/html/;\n}\n"
    kind: ConfigMap
    metadata:
      creationTimestamp: "2019-05-30T10:08:45Z"
      name: nginx-www
      namespace: default
      resourceVersion: "586719"
      selfLink: /api/v1/namespaces/default/configmaps/nginx-www
      uid: e8c582a9-82c2-11e9-89e1-000c29d2af8d

通过上述的详细信息，可以看到创建的键值对，
    
    key为www.conf，value为www.conf文件内容。
     
##### 示例，创建一主进程为nginx的镜像(ikubernetes/myapp:v1)启动的pod，通过env方式将上述nginx-config资源定义的key和value，注入到pod的容器的配置文件中。

创建pod配置文件示例：

        [root@docker1:~/mainfests/configmap ]# cat pod-configmap.yaml 
        apiVersion: v1
        kind: Pod                                           #创建一个pod
        metadata:
           name: pod-cm-1
           namespace: default
           labels:
             app: myapp
             tier: frontend
           annotations:
             magedu.com/created-by: "cluster admin"
        spec:
           containers:
           - name: myapp
             image: ikubernetes/myapp:v1
             env:                                           #使用env方式注入变量
               - name: NGINX_SERVER_PORT                    #注入的变量名
                 valueFrom:                                 #环境变量的值来源与，下面引用的  \
                   configMapKeyRef:                         #configmap中定义的值
                         name: nginx-config                 #引用名为nginx-config中的键值对
                         key: nginx_port                    #引用的key为nginx_port的value
               - name: NGINX_SERVER_NAME
                 valueFrom:
                   configMapKeyRef:
                        name: nginx-config
                        key: server_name

创建并查看容器中，是否有我们注入的变量及值：
        
        [root@docker1:~/mainfests/configmap ]# kubectl  apply -f pod-configmap.yaml 
        pod/pod-cm-1 created
        
        [root@docker1:~ ]# kubectl  exec -it pod-cm-1 -- printenv
        PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        HOSTNAME=pod-cm-1
        TERM=xterm
        NGINX_SERVER_NAME=myapp.magedu.com
        NGINX_SERVER_PORT=80
        MYAPP_PORT_80_TCP=tcp://10.107.183.224:80

        

