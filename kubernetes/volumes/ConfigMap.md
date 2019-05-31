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
     
##### 示例一，env方式注入变量到pod。创建一主进程为nginx的镜像(ikubernetes/myapp:v1)启动的pod，通过env方式将上述nginx-config资源定义的key和value，注入到pod的容器中。

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
        ...
结果显示，NGINX_SERVER_PORT和NGINX_SERVER_NAME变量注入到pod中。

如果修改configmap中变量的值，pod中注入的变量值，会发生改变吗？

手动修改nginx-config中变量NGINX_SERVER_PORT的值为“8080”，并查看pod中的变量值是否发生变化：
    
        [root@docker1:~ ]# kubectl  edit cm/nginx-config
        # Please edit the object below. Lines beginning with a '#' will be ignored,
        # and an empty file will abort the edit. If an error occurs while saving this file will be
        # reopened with the relevant failures.
        #
        apiVersion: v1
        data:
          nginx_port: "8080"
          server_name: myapp.magedu.com
        kind: ConfigMap
        metadata:
          creationTimestamp: "2019-05-30T09:50:01Z"
          name: nginx-config
          namespace: default
          resourceVersion: "684544"
          selfLink: /api/v1/namespaces/default/configmaps/nginx-config
          uid: 4aecdd9e-82c0-11e9-89e1-000c29d2af8d
        
        查看pod中变量值，是否发生变化？
        [root@docker1:~ ]# kubectl  exec -it pod-cm-1 -- printenv
        PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        HOSTNAME=pod-cm-1
        TERM=xterm
        NGINX_SERVER_NAME=myapp.magedu.com
        NGINX_SERVER_PORT=80
        MYAPP_PORT_80_TCP=tcp://10.107.183.224:80
        TOMCAT_PORT_8009_TCP_PORT=8009
        ...
        结果显示还是80，并没有修改为8080.
###### 说明env方式注入的变量是在pod容器创建时注入的，pod容器中变量不会跟随configmap中数据的修改，而发生变化。

##### 示例二，configmap方式注入变量到pod。创建一主进程为nginx的镜像(ikubernetes/myapp:v1)启动的pod，通过configmap方式将上述nginx-config资源定义的key和value，注入到pod的容器中。      

创建并应用pod配置文件示例；

        [root@docker1:~/mainfests/configmap ]# vim pod-configmap-2.yaml 
        apiVersion: v1
        kind: Pod
        metadata:
           name: pod-cm-2
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
             volumeMounts:
               - name: nginxconf                     #要挂载的volume为nginxconf这个存储卷。
                 mountPath: /etc/nginx/config.d/     #定义的挂载点目录，将nginxconf存储卷挂载到指定目录。将nginx-config资源的键值对当作文件，挂载到此目录中。
                 readOnly: true                      #只读挂载，内容不能被pod修改。
           volumes:
           - name: nginxconf
             configMap:                              #使用configmap类型的资源作为存储卷           
                name: nginx-config                   #使用哪个configmap，这使用nginx-config存储卷
        
        [root@docker1 configmap]# kubectl  apply -f pod-configmap-2.yaml 
        pod/pod-cm-2 created

###### pod在创建时，使用存储卷的方式，引用nginx-config这个configmap中的数据，将key为文件名，value为文件内容的方式，注入到挂载点中。
        [root@docker1:~/mainfests/configmap ]# kubectl  get cm/nginx-config -o yaml
        apiVersion: v1
        data:
          nginx_port: "8088"
          server_name: myapp.magedu.com
        kind: ConfigMap
        metadata:
          creationTimestamp: "2019-05-30T09:50:01Z"
          name: nginx-config
          namespace: default
          resourceVersion: "694740"
          selfLink: /api/v1/namespaces/default/configmaps/nginx-config
          uid: 4aecdd9e-82c0-11e9-89e1-000c29d2af8d
nginx-config中的键值对：
      nginx_port: "8088"
      server_name: myapp.magedu.com
        
        [root@docker1:~/mainfests/configmap ]# kubectl  exec -it pod-cm-2 -- /bin/sh
        / # cd /etc/nginx/config.d/
        /etc/nginx/config.d # ls
        nginx_port   server_name
        /etc/nginx/config.d # cat server_name 
        myapp.magedu.com/etc/nginx/config.d # 
        /etc/nginx/config.d # cat nginx_port 
        80/etc/nginx/config.d # 
nginx-config中的key键，当作文件名，注入到挂载目录/etc/nginx/config.d/中，文件内容就是value值。

修改nginx-config中变量的值，pod中变量的值，是否发生变化？ 

        [root@docker1:~/mainfests/configmap ]# kubectl  edit cm/nginx-config
        # Please edit the object below. Lines beginning with a '#' will be ignored,
        # and an empty file will abort the edit. If an error occurs while saving this file will be
        # reopened with the relevant failures.
        #
        apiVersion: v1
        data:
          nginx_port: "8088"
          server_name: myapp.magedu.com
        kind: ConfigMap
        metadata:
          creationTimestamp: "2019-05-30T09:50:01Z"
          name: nginx-config
          namespace: default
          resourceVersion: "694704"
          selfLink: /api/v1/namespaces/default/configmaps/nginx-config
          uid: 4aecdd9e-82c0-11e9-89e1-000c29d2af8d
          
        [root@docker1:~/mainfests/configmap ]# kubectl  exec -it pod-cm-2 -- cat /etc/nginx/config.d/nginx_port
        8088 
##### 实验结果显示，修改configmap类型nginx-config资源的键值对，pod中注入的变量值也会跟着一起发生改变。
