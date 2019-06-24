### ingress解析

Ingress 是从Kubernetes集群外部访问集群内部服务的入口。将集群外部的http或https流量导入集群中的service中。转发路由是有定义的ingress类型资源时的rules规则设定的。

外部访问流程：

        internet
            |
       [ Ingress ]
       --|-----|--
       [ Services ]
