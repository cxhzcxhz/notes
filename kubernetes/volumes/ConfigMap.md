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
nginx-config中的key键nginx_port、server_name被当作文件名，注入到挂载目录/etc/nginx/config.d/中，文件内容就是value值。

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

### 下面实验将configmap类型的资源nginx-www的键值对，在创建pod时，将键值对以文件形式作为虚拟主机使用，工作在我们定义好的主机名和端口。

创建pod配置文件，并应用：

        [root@docker1:~/mainfests/configmap ]# vim pod-configmap-3.yaml 
        apiVersion: v1
        kind: Pod
        metadata:
           name: pod-cm-3
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
               - name: nginxconf
                 mountPath: /etc/nginx/conf.d/
                 readOnly: true
           volumes:
           - name: nginxconf
             configMap:
                name: nginx-www
                
        [root@docker1:~/mainfests/configmap ]# kubectl  apply -f pod-configmap-3.yaml 
        pod/pod-cm-3 created
        
        [root@docker1:~ ]# kubectl  get pods -o wide
        NAME                             READY   STATUS    RESTARTS   AGE   IP            NODE      NOMINATED NODE   READINESS GATES
        myapp-deploy-6b56d98b6b-2fzxd    1/1     Running   0          28h   10.244.1.60   docker2   <none>           <none>
        myapp-deploy-6b56d98b6b-fd7ps    1/1     Running   0          28h   10.244.2.54   docker3   <none>           <none>
        myapp-deploy-6b56d98b6b-wcwd2    1/1     Running   0          28h   10.244.1.61   docker2   <none>           <none>
        pod-cm-3                         1/1     Running   0          19m   10.244.2.57   docker3   <none>           <none>
              
        查看configmap中的键值对信息：
        [root@docker1:~ ]# kubectl  describe cm/nginx-www
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

        查看pod中，注入的文件信息，与configmap的nginx-www键值对一样的信息：
        [root@docker1:~/mainfests/configmap ]# kubectl  exec -it pod-cm-3 -- /bin/sh
        / # cd /etc/nginx/conf.d/..
        ..2019_05_31_06_16_02.042377696/  ..data/
        / # cd /etc/nginx/conf.d/
        /etc/nginx/conf.d # ls
        www.conf
        /etc/nginx/conf.d # cat www.conf 
        server {
            server_name myapp.magedu.com;
            listen 80;
            root /data/web/html/;
        }
        /etc/nginx/conf.d #  创建工作目录及主页文件操作：
        /etc/nginx/conf.d # mkdir -p /data/web/html/
        /etc/nginx/conf.d # cd /data/web/html/
        /data/web/html # cat > index.html << end
        > this is cm/nginx-www index
        > end
        /data/web/html #
        
访问pod，测试结果：

        [root@docker2 ~]# curl 10.244.2.57
        this is cm/nginx-www index
 直接可以工作了。
 
 现在修改cm/nginx-www的listen值为8099，看pod中的值是否会更随发生变化：
 
        
        [root@docker1:~ ]# kubectl  edit cm/nginx-www
        # Please edit the object below. Lines beginning with a '#' will be ignored,
        # and an empty file will abort the edit. If an error occurs while saving this file will be
        # reopened with the relevant failures.
        #
        apiVersion: v1
        data:
          www.conf: "server {\n\tserver_name myapp.magedu.com;\n\tlisten 8099;\n\troot /data/web/html/;\n}\n"
        kind: ConfigMap
        metadata:
          creationTimestamp: "2019-05-30T10:08:45Z"
          name: nginx-www
          namespace: default
          resourceVersion: "586719"
          selfLink: /api/v1/namespaces/default/configmaps/nginx-www
          uid: e8c582a9-82c2-11e9-89e1-000c29d2af8d
        "/tmp/kubectl-edit-abnpd.yaml" 15L, 572C written
        configmap/nginx-www edited
  
查看pod中，www.conf文件的内容，时候发生改变；（需要有个等待过程）
        
        [root@docker1:~/mainfests/configmap ]# kubectl  exec -it pod-cm-3 -- /bin/sh
        / # cat /etc/nginx/conf.d/www.conf 
        server {
            server_name myapp.magedu.com;
            listen 8099;
            root /data/web/html/;
        }
        / # 
          
但是服务监听的端口，不会自动修改为8099，需要将服务重载后生效。 

        / # netstat -tnlp
        Active Internet connections (only servers)
        Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
        tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      1/nginx: master pro

重载服务，使用新的配置：

        / # nginx -s reload
        2019/05/31 06:54:07 [notice] 40#40: signal process started
        / # netstat -tnlp
        Active Internet connections (only servers)
        Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
        tcp        0      0 0.0.0.0:8099            0.0.0.0:*               LISTEN      1/nginx: master pro

再次测试访问pod，测试结果：

        [root@docker2 ~]# curl 10.244.2.57
        curl: (7) Failed connect to 10.244.2.57:80; Connection refused
        [root@docker2 ~]# curl 10.244.2.57:8099
        this is cm/nginx-www index

#### 修改cm类型的资源，pod的值会发生改变。如果是静态文件内容，无所谓了，直接生效，如果是主进程的配置文件发生变化，需要重载配置文件才能生效。

#### 注入部分键值对到pods中。
我们上述实验中，都是将cm类型资源nginx-config、nginx-www全部的键值对，都注入到pod中。
下面实验，将部分键值对注入到pod中。

示例：创建pod配置文件，只将key为nginx_port的值，注入到pod中。


        [root@docker1:~/mainfests/configmap ]# vim pod-configmap-2-portion.yaml
        apiVersion: v1
        kind: Pod
        metadata:
           name: pod-cm-2-portion
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
               - name: nginxconf
                 mountPath: /etc/nginx/config.d/
           volumes:
           - name: nginxconf
             configMap:
                name: nginx-config           #使用cm类型的nginx-config资源的键值对
                items:                       #用来指定我们指定的key。
                  - key: nginx_port          #使用nginx-config的具体的键（key）
                    path: port              #注入到pod中的文件名
                                
        [root@docker1:~/mainfests/configmap ]# kubectl  apply -f pod-configmap-2-portion.yaml 
        pod/pod-cm-2-portion created
        [root@docker1:~/mainfests/configmap ]# kubectl  get pods
        NAME                             READY   STATUS    RESTARTS   AGE
        myapp-deploy-6b56d98b6b-2fzxd    1/1     Running   0          29h
        myapp-deploy-6b56d98b6b-fd7ps    1/1     Running   0          29h
        myapp-deploy-6b56d98b6b-wcwd2    1/1     Running   0          29h
        pod-cm-2-portion                 1/1     Running   0          6s
        pod-cm-3                         1/1     Running   0          68m
        pod-vol-pvc                      1/1     Running   0          27h
        tomcat-deploy-8475677b49-9k776   1/1     Running   0          29h
        tomcat-deploy-8475677b49-gvb9c   1/1     Running   0          29h
        tomcat-deploy-8475677b49-p7fd4   1/1     Running   1          2d1h
        
        [root@docker1:~/mainfests/configmap ]# kubectl exec -it pod-cm-2-portion -- /bin/sh
        / # cd /etc/nginx/config.d/
        /etc/nginx/config.d # ls
        port
        /etc/nginx/config.d # cat port
        8088/etc/nginx/config.d # 
        进入pod中查看，只有一个文件port，文件内容为nginx-config资源中的nginx_port（key）键的值（value）。
 
#### 以键值对作为虚拟主机文件，注入键值对到pods中。 

比如在实际应用中，
可以将path：指定为www.XXX.com.conf某虚拟机主的配置文件名，
而key指定为是这个虚拟主机的时间配置文件内容，而挂载点是默认的/etc/nginx/conf.d/，启动的pod时，直接一个虚拟主机就创建成功了。

示例，直接将nginx-www的键值对，作为虚拟主机，注入到pod中。
 
        [root@docker1:~/mainfests/configmap ]# vim pod-configmap-3-portion.yaml.yaml
        apiVersion: v1
        kind: Pod
        metadata:
           name: pod-cm-3-portion
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
               - name: nginxconf
                 mountPath: /etc/nginx/conf.d/
           volumes:
           - name: nginxconf
             configMap:
                name: nginx-www                   #使用的具体的configmap类型nginx-www资源
                items:                            #items支持部分导入键值对
                  - key: www.conf                 #使用的key
                    path: myapp.magedu.com.conf   #注入的文件名，就是虚拟主机配置文件，直接可以使用。
        ~                                                                                                               
        "pod-configmap-3-portion.yaml.yaml" 24L, 498C written                                         
        [root@docker1:~/mainfests/configmap ]# 
        [root@docker1:~/mainfests/configmap ]# 
        [root@docker1:~/mainfests/configmap ]# kubectl  apply -f pod-configmap-3-portion.yaml.yaml 
        pod/pod-cm-3-portion created
        [root@docker1:~/mainfests/configmap ]# kubectl  get pods
        NAME                             READY   STATUS    RESTARTS   AGE
        myapp-deploy-6b56d98b6b-2fzxd    1/1     Running   0          29h
        myapp-deploy-6b56d98b6b-fd7ps    1/1     Running   0          29h
        myapp-deploy-6b56d98b6b-wcwd2    1/1     Running   0          29h
        pod-cm-3                         1/1     Running   0          92m
        pod-cm-3-portion                 1/1     Running   0          2s
        pod-vol-pvc                      1/1     Running   0          28h
        tomcat-deploy-8475677b49-9k776   1/1     Running   0          29h
        tomcat-deploy-8475677b49-gvb9c   1/1     Running   0          29h
        tomcat-deploy-8475677b49-p7fd4   1/1     Running   1          2d1h
        
        进入pod中，查看注入的虚拟主机配置文件myapp.magedu.com.conf。
        [root@docker1:~/mainfests/configmap ]# kubectl  exec -it pod-cm-3-portion -- /bin/sh
        / # cd /etc/nginx/
        /etc/nginx # ls
        conf.d                  koi-utf                 nginx.conf              uwsgi_params.default
        fastcgi.conf            koi-win                 nginx.conf.default      win-utf
        fastcgi.conf.default    mime.types              scgi_params
        fastcgi_params          mime.types.default      scgi_params.default
        fastcgi_params.default  modules                 uwsgi_params
        /etc/nginx # cd conf.d/
        /etc/nginx/conf.d # ls
        myapp.magedu.com.conf
        /etc/nginx/conf.d # cat myapp.magedu.com.conf 
        server {
            server_name myapp.magedu.com;
            listen 8099;
            root /data/web/html/;
        }
        /etc/nginx/conf.d #

实现了直接将cm/nginx-www资源中的键值对，注入到pod中，生成虚拟主机配置文件。
