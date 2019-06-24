### ingress也是k8s的标准资源。
定义ingress类型资源使用字段，apiVersion、kind、metadata、spec、status
重要字段spec，使用的字段有：
[root@docker1:~/mainfests/ingress ]# kubectl  explain ingress.spec
      backend：定义后端有哪些主机，就是使用seriveName引用service类型资源匹配到的pods。
      rules：定义规则列表，通过定义的规则，将流量导入到指定的service资源匹配到的pods上。
      tls ：定义https访问时使用的字段，需要先创建secret类型资源，封装证书和key，通过secretName字段引用创建secret类型资源，即可将ssl证书及私钥，注入到controller中。
      

