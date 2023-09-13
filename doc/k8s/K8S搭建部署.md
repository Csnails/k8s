# 一、kube-prometheus搭建部署



## 1.1、环境说明

| 组件            | 版本                                                         |
| --------------- | ------------------------------------------------------------ |
| k8s集群         | v1.20.2                                                      |
| kube-prometheus | release-0.7  根据k8s的版本进行选择合适的版本下载，下载地址：[`kube-prometheus`](https://github.com/prometheus-operator/kube-prometheus) |

## 1.2、安装部署

```shell
# 非联网状态下，手动上传镜像文件，先执行上述内容。联网状态下忽略。
# 切记注意要整个节点导入镜像，不然分配到节点无镜像容器启动失败
docker save -o 文件名 镜像名称：版本
docker load < 文件名


# 联网状态下
unzip kube-prometheus-release-0.7.zip 
cd kube-prometheus/manifests/setup/ && kubectl apply -f .
cd kube-prometheus/manifests/ && kubectl apply -f .
```

## 1.3、查看启动情况

```shell
# crd
[root@master1 perometheus]# kubectl get crd|grep   coreos
alertmanagerconfigs.monitoring.coreos.com             2023-09-05T01:06:13Z
alertmanagers.monitoring.coreos.com                   2023-09-05T01:06:13Z
podmonitors.monitoring.coreos.com                     2023-09-05T01:06:13Z
probes.monitoring.coreos.com                          2023-09-05T01:06:13Z
prometheuses.monitoring.coreos.com                    2023-09-05T01:06:13Z
prometheusrules.monitoring.coreos.com                 2023-09-05T01:06:13Z
servicemonitors.monitoring.coreos.com                 2023-09-05T01:06:14Z
thanosrulers.monitoring.coreos.com                    2023-09-05T01:06:14Z

# pod
[root@master1 perometheus]# kubectl  get pod -n monitoring
NAME                                  READY   STATUS    RESTARTS   AGE
alertmanager-main-0                   2/2     Running   0          18m
alertmanager-main-1                   2/2     Running   0          18m
alertmanager-main-2                   2/2     Running   0          18m
grafana-f8cd57fcf-pthsp               1/1     Running   0          88m
kube-state-metrics-587bfd4f97-vqqnm   3/3     Running   0          88m
node-exporter-7c2j7                   2/2     Running   0          88m
node-exporter-8n6jc                   2/2     Running   0          88m
node-exporter-dqlbv                   2/2     Running   0          88m
node-exporter-gmfp7                   2/2     Running   0          88m
node-exporter-gpw5k                   2/2     Running   0          88m
node-exporter-tpndk                   2/2     Running   0          88m
node-exporter-vwvwn                   2/2     Running   0          88m
node-exporter-z4hcd                   2/2     Running   0          88m
prometheus-adapter-69b8496df6-hllj6   1/1     Running   0          88m
prometheus-k8s-0                      2/2     Running   0          18m
prometheus-k8s-1                      2/2     Running   1          18m
prometheus-operator-db49d9679-fjk26   2/2     Running   0          28m

# service
[root@master1 perometheus]# kubectl get  svc -n monitoring
NAME                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
alertmanager-main     ClusterIP   10.0.210.34    <none>        9093/TCP            30m
grafana               ClusterIP   10.0.140.22    <none>        3000/TCP            30m
kube-state-metrics    ClusterIP   None           <none>        8443/TCP,9443/TCP   30m
node-exporter         ClusterIP   None           <none>        9100/TCP            30m
prometheus-adapter    ClusterIP   10.0.96.155    <none>        443/TCP             30m
prometheus-k8s        ClusterIP   10.0.228.214   <none>        9090/TCP            30m
prometheus-operator   ClusterIP   None           <none>        8443/TCP            30m

```

## 1.4、修改svc类型

关键词：`type: NodePort`    `nodePort: 39090`



## 1.5、安装后告警修复[`Prometheus`](http://集群:9090/)

+ **KubeControllerManagerDown**
+ **KubeSchedulerDown**

```shell
# 查看
[root@master1 manifests]# netstat -ulntp|grep contro
tcp        0      0 127.0.0.1:10257         0.0.0.0:*               LISTEN      780/kube-controller 
tcp6       0      0 :::10252                :::*                    LISTEN      780/kube-controller 
[root@master1 manifests]# kubectl get servicemonitor -n monitoring  kube-controller-manager -oyaml  最后几行是关键
[root@master1 manifests]# kubectl get svc  -n kube-system
NAME             TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                        AGE
kube-dns         ClusterIP   10.0.0.2      <none>        53/UDP,53/TCP,9153/TCP         147d
kubelet          ClusterIP   None          <none>        10250/TCP,10255/TCP,4194/TCP   5h33m
metrics-server   ClusterIP   10.0.127.64   <none>        443/TCP                        147d
[root@master1 manifests]# kubectl get svc  -n kube-system -l k8s-app=kube-controller-manager
No resources found in kube-system namespace.

# 修复此问题的方法
## 修改kube-controller监听地址  /usr/lib/systemd/system/kube-controller-manager.service或这个文件里面配置的conf
## 创建对应的svc和ep
```

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: kube-controller-manager
  labels:
    k8s-app: kube-controller-manager
spec:
#  selector:
#    component: kube-controller-manager
  type: ClusterIP
  clusterIP: None
  ports:
  - name: http-metrics
    port: 10252
    targetPort: 10252
    protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: kube-scheduler
  labels:
    k8s-app: kube-scheduler
spec:
#  selector:
#    component: kube-scheduler
  type: ClusterIP
  clusterIP: None
  ports:
  - name: http-metrics
    port: 10251
    targetPort: 10251
    protocol: TCP

```

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    k8s-app: kube-controller-manager
  name: kube-controller-manager
  namespace: kube-system
subsets:
- addresses:
  - ip: 192.168.20.20 # change to your kube-controller-manager ip
  - ip: 192.168.20.21 # change to your kube-controller-manager ip
  - ip: 192.168.20.22 # change to your kube-controller-manager ip
  ports:
  - name: http-metrics
    port: 10252
    protocol: TCP
---
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    k8s-app: kube-scheduler
  name: kube-scheduler
  namespace: kube-system
subsets:
- addresses:
  - ip: 192.168.20.20 # change to your kube-scheduler ip 
  - ip: 192.168.20.21 # change to your kube-scheduler ip
  - ip: 192.168.20.22 # change to your kube-scheduler ip
  ports:
  - name: http-metrics
    port: 10251
    protocol: TCP

```

# 二、ELK搭建部署

## 2.1、环境说明

| 组件 | 版本  | namespace    | 映射端口      |
| ---- | ----- | ------------ | ------------- |
| ELK  | 7.9.3 | kube-logging | Kibana：48135 |

yaml文件获取：https://github.com/Csnails/k8s.git

## 2.2、准备工作

```shell
# 由于环境是非联网状态，需要手动上传镜像到相关节点
# 上传包到master
# 分发到相关节点
for i in `seq 11 18`;do scp -P 22622 k8s_es.7.9.3.tar.gz   root@192.168.83.$i:~ ;done
tar -zxvf k8s_es.7.9.3.tar.gz && rm -rf k8s_es.7.9.3.tar.gz
cd /root/es/images
docker load < alpine.tar   && docker load < busybox.tar  &&  docker load < elasticsearch.tar    &&   docker load < filebeat.tar    &&   docker load < kibana.tar    &&  docker load < logstash.tar
```

## 2.3、安装ELK

+ 注意点：

  - es.yaml                    文件内容：nodeSelector可以指定node启动po，如不指定需将此内容删除，否则容器无法正常运行

  - logstash.yaml          注意两个ConfigMap的内容，按实际修改

  - filebeat.yaml            ConfigMap内容修改

  - kibana.yaml             资源Ingress按情况需要，不涉及，进行删除
  
  - namespace.yaml     无注意点，直接创建。

```shell
# filebeat.yaml  logstash.yaml 按具体日志格式修改
cd /root/es/yaml/ && kubectl apply -f .

# 查看
[root@master1 yaml]# kubectl get pod,svc,ep,cm -n kube-logging 
NAME                            READY   STATUS    RESTARTS   AGE
pod/elasticsearch-logging-0     1/1     Running   0          9m15s
pod/filebeat-7tjst              1/1     Running   0          3m9s
pod/filebeat-9f6tt              1/1     Running   0          3m9s
pod/filebeat-bg8sk              1/1     Running   0          3m9s
pod/filebeat-gdk8r              1/1     Running   0          3m9s
pod/filebeat-ggrg9              1/1     Running   0          3m9s
pod/filebeat-hbrwt              1/1     Running   0          3m9s
pod/filebeat-lbdhv              1/1     Running   0          3m9s
pod/filebeat-rsrlb              1/1     Running   0          3m9s
pod/kibana-6d5c8c6778-jfbbp     1/1     Running   0          80s
pod/logstash-86c7c65955-jzb8k   1/1     Running   0          6m51s

NAME                            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
service/elasticsearch-logging   ClusterIP   10.0.50.200   <none>        9200/TCP         9m15s
service/kibana                  NodePort    10.0.29.9     <none>        5601:48135/TCP   80s
service/logstash                ClusterIP   None          <none>        5044/TCP         6m51s

NAME                              ENDPOINTS           AGE
endpoints/elasticsearch-logging   10.96.180.15:9200   9m15s
endpoints/kibana                  10.96.104.42:5601   80s
endpoints/logstash                10.96.139.9:5044    6m51s

NAME                         DATA   AGE
configmap/filebeat-config    1      3m9s
configmap/kibana-config      1      80s
configmap/kube-root-ca.crt   1      28m
configmap/logstash-conf      1      6m51s
configmap/logstash-yml       1      6m51s


```

## 2.4、访问ELK

http://192.168.83.18:48135/app/home#/

左上角菜单----->Stack Management----->索引模式----->创建索引模式



# 三、Sonarqube搭建部署

## 3.1、环境说明

| 组件       | 版本 | 备注            |
| ---------- | ---- | --------------- |
| PostgreSQL |      |                 |
| SonarQube  |      | 端口映射：30003 |

## 3.2、环境准备

### 3.2.1、上传image和yaml文件

测试环境未联网，所以离线上传image

yaml文件获取：https://github.com/Csnails/k8s.git。该GitHub存放的是本人修改的yaml，如不可用可自行网路搜索其他可以的成品yaml

### 3.2.2、修改yaml，使其落到指定节点

`xxx.spec.nodeName`指定节点。使用上述github的yaml文件需要修改下nodeName

```shell
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres-sonar
  template:
    metadata:
      labels:
        app: postgres-sonar
    spec:
      nodeName: node1
      containers:

```

## 3.2、安装PostgreSQL

```shell
[root@master1 sonarQube]# sed -i 's/\xc2\xa0/ /g' PostgreSQL.yaml  && sed -i 's/\s\+$//' PostgreSQL.yaml
[root@master1 sonarQube]# sed -i 's/\xc2\xa0/ /g' SonarQube.yaml  && sed -i 's/\s\+$//' SonarQube.yaml
[root@master1 sonarQube]# kubectl apply  -f PostgreSQL.yaml 
deployment.apps/postgres-sonar created
service/postgres-sonar created

[root@master1 sonarQube]# kubectl get pod
NAME                             READY   STATUS    RESTARTS   AGE
postgres-sonar-df49fb679-bklqr   1/1     Running   0          11s
```

## 3.3、安装SonarQube

```shell
[root@master1 sonarQube]# kubectl apply  -f SonarQube.yaml 
deployment.apps/sonarqube created
service/sonarqube created
[root@master1 sonarQube]# kubectl get pod -owide
NAME                             READY   STATUS    RESTARTS   AGE     IP              NODE    NOMINATED NODE   READINESS GATES
postgres-sonar-df49fb679-bklqr   1/1     Running   0          3m16s   10.96.166.129   node1   <none>           <none>
sonarqube-6575b5d48b-pjv9q       1/1     Running   0          73s     10.96.166.132   node1   <none>           <none>

```

## 3.4、访问检查

| 地址                                | 用户名 | 密码                           |
| ----------------------------------- | ------ | ------------------------------ |
| http://192.168.83.18:30003/projects | admin  | 默认密码admin，修改为Sh******* |

（可选）安装中文插件：导航栏-Adminstration --->Marketplace  ---> Plugins  --->搜索（Chiness Pack）



# 四、jenkins搭建部署

## 4.1、环境说明

| 组件              | 版本        |      |
| ----------------- | ----------- | ---- |
| jenkins           | 2.392-jdk11 |      |
| apache-maven      | 3.9.0       |      |
| sonar-scanner-cli | 4.8.0.2856  |      |
|                   |             |      |

> yaml文件获取：https://github.com/Csnails/k8s.git。需要手动生成Jenkins镜像，在GitHub/yaml/jenkins/jenkins-maven/  所需构建的dockefile

## 4.2、环境准备

```shell
# 由于环境是非联网状态，需要手动上传镜像到相关节点
# 上传包到master
# 分发到相关节点
for i in `seq 11 18`;do scp -P 22622 jenkins-images.tar.gz    root@192.168.83.$i:~ ;done
tar -zxvf jenkins-images.tar.gz  && rm -rf jenkins-images.tar.gz

cd /root/jenkins/images && docker load < jenkins.tar   && rm -rf /root/jenkins/
```

## 4.3、安装jenkins

```shell
# 查看日志获取密码信息 e9bb9a0d95994376a5944bd5f7cf6ac9
[root@master1 manifests]# cd /root/k8s-jenkins/manifests && kubectl apply  -f .
[root@master1 manifests]# kubectl logs -f pod/jenkins-74fb9646d8-dlzgt -n kube-devops 
[root@master1 manifests]# kubectl get svc,pods -n kube-devops 
NAME                      TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
service/jenkins-service   NodePort   10.0.53.147   <none>        8080:45753/TCP   8h

NAME                           READY   STATUS    RESTARTS   AGE
pod/jenkins-74fb9646d8-dlzgt   1/1     Running   0          8h


```

访问：http://192.168.83.18:45753/login?from=%2F ---> 输入密码 --->选择插件来安装--->等待--->进入页面--->设置--->修改密码passwd



## 4.4、插件安装

+ Build Authorization Token Root

+ Gitlab

+ SonarQube Scanner

+ Node and Label parameter

+ Kubernetes

## 4.5、常用配置

### 4.5.1、配置Sonarqube

#### 4.5.1.1、获取Sonarqube的token

首页 ---> 右上角用户图标--->我的账户--->安全--->generate tokens--->输入信息（比如Jenkins）--->点击generate--->记录token（7560a1930c2204e138caebc90dc7ffb2fc5046ad）

![image-20230913093724929](C:\Users\liubaoyi\Desktop\软件-崔\K8S\images\image-20230913093724929.png)

![image-20230913093619131](C:\Users\liubaoyi\Desktop\软件-崔\K8S\images\image-20230913093619131.png)

#### 4.5.1.2 Jenkins配置Sonarqube

配置凭证：Dashboard ---->系统管理  ---->凭证  ----> 系统  ----> 全局凭证  ----> NEW credentials （类型选择：Secret test；Secret：上步骤生成的token）

![image-20230913094921864](C:\Users\liubaoyi\Desktop\软件-崔\K8S\images\image-20230913094921864.png)

```shell
[root@master1 k8s-jenkins]# kubectl get svc --all-namespaces|grep son
default         postgres-sonar            ClusterIP   None           <none>        5432/TCP                        40h
default         sonarqube                 NodePort    10.0.162.147   <none>        9000:30003/TCP                  40h
记录sonarqube所在的空间，便于配置sonarqube

```

配置sonarqube：Dashboard ---->系统管理 ---->Configure System ---->SonarQube（name:自定义；server url:http://svc的name.命名空间:端口；选择先去添加的token）

![image-20230913100914171](C:\Users\liubaoyi\Desktop\软件-崔\K8S\images\image-20230913100914171.png)

查看sonarqube所在的空间，然后配置格式：svc名称.命名空间:端口

## 4.5.2、kubernetes

配置kubernets：Dashboard ---->系统管理 ---->节点管理 ---->Configure Clouds---->配置参数

前提：Jenkins部署在k8s里面

- Kubernetes 地址：https://kubernetes.default

- 禁用 HTTPS 证书检查：勾选

- Jenkins 地址：http://jenkins-service.kube-devops:8080

![image-20230913154919087](C:\Users\liubaoyi\Desktop\软件-崔\K8S\images\image-20230913154919087.png)



































































# 附录：常用命令

```shell

kubectl  apply -f prometheus-operator-deployment.yaml 

kubectl  delete  -f prometheus-operator-deployment.yaml 

kubectl describe  po prometheus-operator-db49d9679-rhtk9   -n monitoring

kubectl replace  -f prometheus-operator-deployment.yaml 

kubectl  edit deploy prometheus-operator -n monitoring

kubectl  delete  pod prometheus-operator-7649c7454f-qgd5p  -n monitoring
```



```shell
curl prometheus-k8s的clusterIP/metrics    ## 查看数据
```

grafana  URL 密码忘记处理方法 `grafana-cli  admin reset-admin-password 新密码 `
