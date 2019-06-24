### ingress也是k8s的标准资源。
定义ingress类型资源使用字段，apiVersion、kind、metadata、spec、status

重要字段spec，使用的字段有：

      [root@docker1:~/mainfests/ingress ]# kubectl  explain ingress.spec
      backend：定义后端有哪些主机，就是使用seriveName引用service类型资源匹配到的pods。
      rules：定义规则列表，通过定义的规则，将流量导入到指定的service资源匹配到的pods上。
      tls ：定义https访问时使用的字段，需要先创建secret类型资源，封装证书和key，通过secretName字段引用创建secret类型资源，即可将ssl证书及私钥，注入到controller中。


示例1、创建ingress类型资源ingress-myapp，定义基于主机名的虚拟主机，将规则注入到controller中。
#### 创建ingress类型资源之前，我们先准备好service类型资源及匹配到的pods资源，创建完成。
创建service类型资源myapp，及使用deployment控制器创建的3个pods，配置文件如下：

      [root@docker1:~/mainfests/ingress ]# cat deploy-demo.yaml 
      apiVersion: v1
      kind: Service
      metadata:
         name: myapp
         namespace: default
      spec:
         selector:
            app: myapp
            release: canary
         ports:
         - name: http
           targetPort: 80
           port: 80

      ---
      apiVersion: apps/v1
      kind: Deployment
      metadata:
          name: myapp-deploy
          namespace: default
      spec:
          replicas: 3 
          selector:
              matchLabels:
                  app: myapp
                  release: canary
          template:
              metadata:
                  labels:
                      app: myapp
                      release: canary
              spec:
                  containers:
                  - name: myapp
                    image: ikubernetes/myapp:v2
                    ports:
                    - name: http
                      containerPort: 80
      
      [root@docker1:~/mainfests/ingress ]# kubectl apply -f deploy-demo.yaml
      [root@docker1:~/mainfests/ingress ]# kubectl  get svc
      NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
      kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP             31d
      myapp        ClusterIP   10.102.67.125    <none>        80/TCP              5h50m
      
      [root@docker1:~/mainfests/ingress ]# kubectl  get pods
      NAME                             READY   STATUS    RESTARTS   AGE
      myapp-deploy-6b56d98b6b-8xm59    1/1     Running   0          3h26m
      myapp-deploy-6b56d98b6b-qq9rr    1/1     Running   0          3h26m
      myapp-deploy-6b56d98b6b-qxbs8    1/1     Running   0          3h26m
上述结果显示，service及pods准备完成。

#### 创建service-nodeport.yanl配置文件，创建service类型资源ingress-nginx，在ingress-nginx名称空间，使用type: NodePort 字段指令，监听在node节点的指定端口。（可以匹配到controller pods，并引入外部流量到controller pods，而pods中有使用ingress类型资源生成的虚拟主机）

      [root@docker1:~/mainfests/ingress ]# cat service-nodeport.yaml 
      apiVersion: v1
      kind: Service
      metadata:
        name: ingress-nginx
        namespace: ingress-nginx
        labels:
          app.kubernetes.io/name: ingress-nginx
          app.kubernetes.io/part-of: ingress-nginx
      spec:
        type: NodePort
        ports:
          - name: http
            port: 80
            targetPort: 80
            nodePort: 30080
            protocol: TCP
          - name: https
            port: 443
            targetPort: 443
            nodePort: 30443
            protocol: TCP
        selector:
          app.kubernetes.io/name: ingress-nginx
          app.kubernetes.io/part-of: ingress-nginx

      [root@docker1:~/mainfests/ingress ]# kubectl  get svc -n ingress-nginx
      NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
      ingress-nginx   NodePort   10.101.95.135   <none>        80:30080/TCP,443:30443/TCP   4h38m


#### 创建ingress类型资源ingress-myapp，内容如下：

      [root@docker1:~/mainfests/ingress ]# cat ingress-myapp.yaml 
      apiVersion: extensions/v1beta1
      kind: Ingress     # 类型为ingress
      metadata:
          name: ingress-myapp   # ingress名称为：ingress-myapp
          namespace: default
          annotations:
              kubernetes.io/ingress.class: "nginx"  #使用annotations字段，标明使用nginx ingress controller。
      spec:
          rules:    # 定义规则
          - host: myapp.magedu.com  # 基于主机名的虚拟主机。
            http:
              paths:
              - path:  # 这里可以定义基于url的虚拟主机。
                backend:  # 定义后端主机，就是匹配到的pods数量。
                  serviceName: myapp   # 使用service类型资源名称为myapp的资源，匹配后端pods数量，并监听pods数量是否变化，并将最新结果通知给                                              ingress类型资源ingress-myapp，ingress-myapp立即将pods最新地址，注入到ingress controller中，并通知                                            ingress controller重载配置文件，使用最新配置。
                  servicePort: 80
      
      [root@docker1:~/mainfests/ingress ]# kubectl  apply -f ingress-myapp.yaml
      [root@docker1:~/mainfests/ingress ]# kubectl  get ing
      NAME                 HOSTS               ADDRESS   PORTS     AGE
      ingress-myapp        myapp.magedu.com              80        4h41m
      
查看nginx ingress controller中注入的配置文件信息，查看是否有我们注入主机名为myapp.magedu.com的虚拟主机，命令：
      
      [root@docker1:~/mainfests/ingress ]# kubectl  exec -ti nginx-ingress-controller-689498bc7c-rqfxj  -n ingress-nginx --  cat nginx.conf|grep -C 20 "myapp.magedu.com"

	## start server myapp.magedu.com
	server {
		server_name myapp.magedu.com ;
		
		listen 80;
		
		set $proxy_upstream_name "-";
		set $pass_access_scheme $scheme;
		set $pass_server_port $server_port;
		set $best_http_host $http_host;
		set $pass_port $pass_server_port;
		
		location / {
			
			set $namespace      "default";
			set $ingress_name   "ingress-myapp";
			set $service_name   "myapp";
			set $service_port   "80";
			set $location_path  "/";
			
			rewrite_by_lua_block {
				lua_ingress.rewrite({
					force_ssl_redirect = false,
			--
  			proxy_buffer_size                       4k;
			proxy_buffers                           4 4k;
			proxy_request_buffering                 on;
			
			proxy_http_version                      1.1;
			
			proxy_cookie_domain                     off;
			proxy_cookie_path                       off;
			
			# In case of errors try the next upstream server before returning an error
			proxy_next_upstream                     error timeout;
			proxy_next_upstream_tries               3;
			
			proxy_pass http://upstream_balancer;
			
			proxy_redirect                          off;
			
		}
		
	}
	## end server myapp.magedu.com
结果显示，在nginx ingress controller的pods中，已经注入了“myapp.magedu.com”虚拟主机。

在集群外部访问service类型资源ingress-nginx，监听的node节点的port端口30080，访问测试，命令：http://myapp.magedu.com:30080/hostname.html

结果如图所示：就是后端pods的名称。

      https://github.com/cxhzcxhz/notes/blob/master/kubernetes/ingress/images/myapp.magedu.com1.png
      https://github.com/cxhzcxhz/notes/blob/master/kubernetes/ingress/images/myapp.magedu.com2.png
      https://github.com/cxhzcxhz/notes/blob/master/kubernetes/ingress/images/myapp.magedu.com3.png
      我们系统中的pods信息，命令；
      [root@docker1:~/mainfests/ingress ]# kubectl  get pods
      NAME                             READY   STATUS    RESTARTS   AGE
      myapp-deploy-6b56d98b6b-8xm59    1/1     Running   0          4h3m
      myapp-deploy-6b56d98b6b-qq9rr    1/1     Running   0          4h3m
      myapp-deploy-6b56d98b6b-qxbs8    1/1     Running   0          4h3m
      
与我们在集群外部访问，得到的结果一致。     



#### 示例2，创建ingress类型资源ingress-tomcat配置文件，创建基于主机名“tomcat.magedu.com”的虚拟主机，并使用service类型资源ingress-nginx监听的30080端口，通过集群外部访问测试。

提前准备需要的service及pods后端资源，创建service类型资源tomcat，并使用deployment控制器创建3个pods，提供tomcat主页服务。
配置文件为：
	
	[root@docker1:~/mainfests/ingress ]# cat tomcat-deploy.yaml 
	apiVersion: v1
	kind: Service
	metadata:
	   name: tomcat 
	   namespace: default
	   labels:
	     app: tomcat
	     release: canary
	spec:
	   selector:
	      app: tomcat
	      release: canary
	   ports:
	   - name: http
	     targetPort: 8080
	     port: 8080
	   - name: ajp
	     targetPort: 8009
	     port: 8009

	---
	apiVersion: apps/v1
	kind: Deployment
	metadata:
	    name: tomcat-deploy
	    namespace: default
	spec:
	    replicas: 3
	    selector:
		matchLabels:
		    app: tomcat
		    release: canary
	    template:
		metadata:
		    labels:
			app: tomcat
			release: canary
		spec:
		    containers:
		    - name: tomcat
		      image: tomcat:8.5.38-alpine 
		      ports:
		      - name: http
			containerPort: 8080
		      - name: ajp
			containerPort: 8009
	
	[root@docker1:~ ]# kubectl  get svc
	NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
	kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP             31d
	myapp        ClusterIP   10.102.67.125    <none>        80/TCP              6h41m
	tomcat       ClusterIP   10.108.213.128   <none>        8080/TCP,8009/TCP   3h40m
	
	[root@docker1:~ ]# kubectl  get pods
	NAME                             READY   STATUS    RESTARTS   AGE
	myapp-deploy-6b56d98b6b-8xm59    1/1     Running   0          4h17m
	myapp-deploy-6b56d98b6b-qq9rr    1/1     Running   0          4h17m
	myapp-deploy-6b56d98b6b-qxbs8    1/1     Running   0          4h17m
	tomcat-deploy-8475677b49-npl96   1/1     Running   0          3h41m
	tomcat-deploy-8475677b49-w6t59   1/1     Running   0          3h41m
	tomcat-deploy-8475677b49-zzl5v   1/1     Running   0          3h41m
结果显示，service类型资源tomcat及pods已经running。

下面创建ingress类型资源ingress-tomcat，将主机名为“tomcat.magedu.com”的虚拟主机注入到nginx ingress controller中，并使用service类型资源ingress-nginx监听的30080端口，在集群外部访问测试。

ingress-tomcat配置文件：

	[root@docker1:~/mainfests/ingress ]# cat ingress-tomcat.yaml 
	apiVersion: extensions/v1beta1
	kind: Ingress
	metadata:
	    name: ingress-tomcat         # ingress名称
	    namespace: default
	    annotations:
		kubernetes.io/ingress.class: "nginx"
	spec:
	    rules:
	    - host: tomcat.magedu.com    # 注入的主机名
	      http:
		paths:
		- path:
		  backend:
		    serviceName: tomcat  # 使用service类型资源为tomcat，对后端pods匹配。
		    servicePort: 8080
	
	[root@docker1:~/mainfests/ingress ]# kubectl  get ing
	NAME                 HOSTS               ADDRESS   PORTS     AGE
	ingress-myapp        myapp.magedu.com              80        5h20m
	ingress-tomcat       tomcat.magedu.com             80        3h49m
	
	[root@docker1:~/mainfests/ingress ]# kubectl  get  svc -n ingress-nginx
	NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
	ingress-nginx   NodePort   10.101.95.135   <none>        80:30080/TCP,443:30443/TCP   5h38m
	
查看nginx ingress controllor pods中，是否有ingress-tomcat注入的主机名为“tomcat.magedu.com”虚拟主机，命令：

	[root@docker1:~/mainfests/ingress ]# kubectl  exec nginx-ingress-controller-689498bc7c-rqfxj -n ingress-nginx -- cat nginx.conf | grep -C 20 "tomcat.magedu.com"
	
	## start server tomcat.magedu.com
	server {
		server_name tomcat.magedu.com ;
		
		listen 80;
		
		set $proxy_upstream_name "-";
		set $pass_access_scheme $scheme;
		set $pass_server_port $server_port;
		set $best_http_host $http_host;
		set $pass_port $pass_server_port;
		
		listen 443  ssl http2;
		
		# PEM sha: be786d72a213ca0259b8617b02fa78a82a0a03fc
		ssl_certificate                         /etc/ingress-controller/ssl/default-fake-certificate.pem;
		ssl_certificate_key                     /etc/ingress-controller/ssl/default-fake-certificate.pem;
		
		ssl_certificate_by_lua_block {
			certificate.call()
		}
		
		location / {
			--
			proxy_buffer_size                       4k;
			proxy_buffers                           4 4k;
			proxy_request_buffering                 on;
			
			proxy_http_version                      1.1;
			
			proxy_cookie_domain                     off;
			proxy_cookie_path                       off;
			
			# In case of errors try the next upstream server before returning an error
			proxy_next_upstream                     error timeout;
			proxy_next_upstream_tries               3;
			
			proxy_pass http://upstream_balancer;
			
			proxy_redirect                          off;
			
		}
		
	}
	## end server tomcat.magedu.com
结果显示，tomcat.magedu.com已经注入到nginx ingress controller中。

集群外部访问测试，命令：http://tomcat.magedu.com:30080/

结果如图所示：https://github.com/cxhzcxhz/notes/blob/master/kubernetes/ingress/images/tomcat.magedu.com.png


#### 示例3，创建ingress类型资源ingress-tomcat-tls配置文件，创建基于主机名“tomcat.magedu.com”的虚拟主机，并使用service类型资源ingress-nginx监听的30443端口，实现https集群外部访问测试。

自建证书操作：
	
	创建私钥：
	[root@docker1:~/mainfests/ingress ]# openssl genrsa -out tls.key 2048
	创建证书：
	[root@docker1:~/mainfests/ingress ]# openssl  req -new -x509 -key tls.key  -out tls.crt -subj /C=CN/ST=Beijing/L=Beijing/O=Devops/CN=tomcat.magedu.com
	创建secret类型的资源，secret类型资源可以注入pod中。
	[root@docker1:~/mainfests/ingress ]# kubectl  create secret  tls tomcat-ingress-secret --cert=tls.crt  --key=tls.key
	secret/tomcat-ingress-secret created
	
	[root@docker1:~/mainfests/ingress ]# kubectl  get secret
	NAME                       TYPE                                  DATA   AGE
	admin-token-dh956          kubernetes.io/service-account-token   3      13d
	def-ns-admin-token-cxp4k   kubernetes.io/service-account-token   3      6d9h
	default-token-dh66j        kubernetes.io/service-account-token   3      31d
	tomcat-ingress-secret      kubernetes.io/tls                     2      28d
	
	[root@docker1:~/mainfests/ingress ]# kubectl  describe secret tomcat-ingress-secret
	Name:         tomcat-ingress-secret
	Namespace:    default
	Labels:       <none>
	Annotations:  <none>

	Type:  kubernetes.io/tls

	Data
	====
	tls.crt:  1294 bytes
	tls.key:  1675 bytes
	
创建ingress类型资源ingress-tomcat-tls，注入主机名为“tomcat.magedu.com”虚拟主机到nginx ingress controller中。
配置文件：

	[root@docker1:~/mainfests/ingress ]# cat ingress-tomcat-tls.yaml 
	apiVersion: extensions/v1beta1
	kind: Ingress
	metadata:
	    name: ingress-tomcat-tls
	    namespace: default
	    annotations:
		kubernetes.io/ingress.class: "nginx"
	spec:
	    tls:
	    - hosts: 
	      - tomcat.magedu.com
	      secretName: tomcat-ingress-secret
	    rules:
	    - host: tomcat.magedu.com
	      http:
		paths:
		- path:
		  backend:
		    serviceName: tomcat   #还是使用先前的service类型资源tomcat，通过tomcat匹配后端的pods。
		    servicePort: 8080

查看nginx ingress controllor pods中，是否有ingress-tomcat-tls注入的主机名为“tomcat.magedu.com”虚拟主机，是否监听在443端口，命令：

	[root@docker1:~/mainfests/ingress ]# kubectl  exec -ti nginx-ingress-controller-689498bc7c-rqfxj  -n ingress-nginx --  /bin/sh
	$ cat nginx.conf
	## start server tomcat.magedu.com
	server {
		server_name tomcat.magedu.com ;
		
		listen 80;                 #监听80端口
		
		set $proxy_upstream_name "-";
		set $pass_access_scheme $scheme;
		set $pass_server_port $server_port;
		set $best_http_host $http_host;
		set $pass_port $pass_server_port;
		
		listen 443  ssl http2;     #监听443端口
		
		# PEM sha: be786d72a213ca0259b8617b02fa78a82a0a03fc
		ssl_certificate                         /etc/ingress-controller/ssl/default-fake-certificate.pem;
		ssl_certificate_key                     /etc/ingress-controller/ssl/default-fake-certificate.pem;
		
		ssl_certificate_by_lua_block {
			certificate.call()
		}

结果显示在nginx ingress controller中已经监听了443端口。

集群外部访问测试，命令为：https://tomcat.magedu.com:30443/

结果如图：https://github.com/cxhzcxhz/notes/blob/master/kubernetes/ingress/images/https-tomcat.magedu.com.png






