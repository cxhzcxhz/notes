### ConfigMap是一种kubernetes中标准资源类型。可以简写为cm。
ConfigMap可以让配置文件与镜像解耦，镜像中的配置文件，可以通过configmap注入到镜像中。

    ConfigMap相当于一些列配置文件的集合，可以注入到pod的容器中使用。
      注入方式：
        1、直接将configmap当存储卷使用。
        2、使用env的valueFrom方式，引用configmap中的数据（键值对）。


