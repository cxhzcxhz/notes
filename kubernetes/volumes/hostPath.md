### hostPath存储卷
#### 简单来说，就是把pod所在宿主机之上的，脱离pod中容器名称空间之外的，宿主机的文件系统目录与pod建立关联关系。所以删除了pod，数据还在本地宿主机上，只要新创建的pod能调度到该宿主机上来，就能使用之前的数据。
#### hostPath有2个子字段，path、type
####     path：用来表示宿主机上的路径信息
####     type：用来指定hostPath存储卷类型
####     type支持的字段值有:
    
#####       DirectoryOrCreate：如果path指定的目录路径不存在，则创建一个空目录，且权限为0755，与kubelet拥有相同的属主属组。
      
#####      Directory：指定path路径必须存在，否则启动失败。
      
#####      FileOrCreate：如果path指定的文件路径不存在，则创建一个空文件，且权限为0644，与kubelet拥有相同的属组属主。
      
#####      File：path指定的文件必须存在，否则启动报错。
      
#####      Socket：path指定的unix socket路径必须存在。
      
#####      CharDevice：path指定的路径必须是个字符设备。
      
#####      BlockDevice：path指定的路径必须是个块设备。


创建hostPath存储卷示例：


           [root@docker1:~/mainfests/volumes ]# cat  pod-hostpath-vol.yaml 
            apiVersion: v1
            kind: Pod
            metadata:
               name: pod-vol-hostpath
               namespace: default
            spec:
               containers:
               - name: myapp
                 image: ikubernetes/myapp:v1
                 volumeMounts:
                 - name: html
                   mountPath: /usr/share/nginx/html/
               volumes:
               - name: html
                 hostPath: 
                    path: /data/pod/volume1
                    type: Directory 
 
在node节点docker2、docker3上，提前准备好宿主机目录路径，并创建index.html文件。

        [root@docker2 ~]# cat  /data/pod/volume1/index.html
        docker2.magedu.com
        [root@docker3 ~]# cat /data/pod/volume1/index.html 
        docker3.magedu.com
如果创建的pod-vol-hostpath，被调度到不同节点，访问主页会有不同的内容。

查看创建的pod信息，并在集群任意节点访问pod的ip，是否为我们期望的内容，

    [root@docker1:~ ]# kubectl  get pods -o wide
    NAME                             READY   STATUS    RESTARTS   AGE   IP            NODE      NOMINATED NODE   READINESS GATES
    pod-vol-hostpath                 1/1     Running   0          16s   10.244.1.46   docker2   <none>           <none>
    tomcat-deploy-8475677b49-fc722   1/1     Running   1          24h   10.244.2.45   docker3   <none>           <none>
    tomcat-deploy-8475677b49-nrd5m   1/1     Running   1          24h   10.244.2.46   docker3   <none>           <none>
    tomcat-deploy-8475677b49-tbx42   1/1     Running   1          22h   10.244.2.47   docker3   <none>           <none>
    
    [root@docker1:~ ]# curl 10.244.1.46
    docker2.magedu.com

结果为：pod被调度到docker2节点，主页为docker2.magedu.com

### 使用hostPath存储卷，数据是保存到宿主机本地，即使删除掉现在的pod，新创建的pod如果能调度到此节点上来还是可以访问存储卷中的数据。实现了节点级别的数据持久存储。






