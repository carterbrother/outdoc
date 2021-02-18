# 云原生系列5 容器化日志方式之EFK

![image.png](https://cdn.nlark.com/yuque/0/2021/png/186661/1613655280771-de4e432f-7509-4f3a-a3a3-3701c3dfc42e.png#align=left&display=inline&height=590&margin=%5Bobject%20Object%5D&name=image.png&originHeight=590&originWidth=1108&size=303928&status=done&style=none&width=1108)


上图是EFK架构图，k8s环境下常见的日志采集方式。


# 日志需求


1. 集中采集微服务的日志，可以根据请求id追踪到完整的日志；





2. 统计请求接口的耗时，超出最长响应时间的，需要做报警，并针对性的进行调优；



3. 慢sql排行榜，并报警；



4. 异常日志排行榜，并报警；



5. 慢页面请求排行，并告警；





# k8s的日志采集


k8s本身不会为你做日志采集，需要自己做；


k8s的容器日志处理方式采用的 集群层级日志，即容器销毁，pod漂移，Node宕机不会对容器日志造成影响；


![image.png](https://cdn.nlark.com/yuque/0/2021/png/186661/1613657299647-cc138900-62e4-45bf-a0eb-fe595e8f30d0.png#align=left&display=inline&height=232&margin=%5Bobject%20Object%5D&name=image.png&originHeight=232&originWidth=696&size=19944&status=done&style=none&width=696)




容器的日志会输出到stdout,stderr,对应的存储在宿主机的目录中，即 /var/lib/docker/container ； 


## Node上通过日志代理转发


![image.png](https://cdn.nlark.com/yuque/0/2021/png/186661/1613657670743-95c2f7ab-f158-4026-ae35-f049d6f182ad.png#align=left&display=inline&height=617&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1234&originWidth=1806&size=132654&status=done&style=none&width=903)




在每个node上部署一个daemonset , 跑一个logging-agent收集日志，比如fluentd, 采集宿主机对应的数据盘上的日志，然后输出到日志存储服务或者消息队列；




优缺点分析：



| 对比 | 说明 |
| --- | --- |
| 优点 | 1.每个Node只需要部署一个Pod采集日志

2.对应用无侵入 |
| 缺点 | 应用输出的日志都必须直接输出到容器的stdout,stderr中 |







## Pod内部通过sidecar容器转发到日志服务


![image.png](https://cdn.nlark.com/yuque/0/2021/png/186661/1613657724977-619efc13-6b34-45e0-b9a8-db62235a1a2b.png#align=left&display=inline&height=365&margin=%5Bobject%20Object%5D&name=image.png&originHeight=730&originWidth=1608&size=69727&status=done&style=none&width=804)




通过在pod中启动一个sidecar容器，比如fluentd， 读取容器挂载的volume目录，输出到日志服务端；


日志输入源： 日志文件


日志处理： logging-agent ,比如fluentd


日志存储： 比如elasticSearch ,  kafka




优缺点分析：



| 对比 | 说明 |
| --- | --- |
| 优点 | 
1. 部署简单；


2. 对宿主机友好；
 |
| 缺点 | 
1. 消耗较多的资源；


2. 日志通过kubectl logs 无法看到
 |





示例：


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox
    args:
    - /bin/sh
    - -c
    - >
        i=0;
        while true;
        do
          echo "$i:$(data)" >> /var/log/1.log
          echo "$(data) INFO $i" >> /var/log/2.log
           i=$((i+1))
          sleep 1;
        done
    volumeMounts:
    - name: varlog
        mountPath: /var/log
  - name: count-agent
    image: k8s.gcr.io/fluentd-gcp:1.30
    env:
    - name: FLUENTD_ARGS
        value: -c /etc/fluentd-config/fluentd.conf
    valumeMounts:
    - name: varlog
        mountPath: /var/log
    - name: config-volume
        mountPath: /etc/fluentd-config
  volumes:
  - name: varlog
      emptyDir: {}
  - name: config-volume
      configMap:
        name: fluentd-config



```



## Pod内部通过sidecar容器输出到stdout


![image.png](https://cdn.nlark.com/yuque/0/2021/png/186661/1613657700097-0ecc804d-3025-4cd7-877b-d767f1ff1679.png#align=left&display=inline&height=566&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1132&originWidth=2128&size=141158&status=done&style=none&width=1064)




适用于应用容器只能把日志输出到文件，无法输出到stdout,stderr中的场景；


通过一个sidecar容器，直接读取日志文件，再重新输出到stdout,stderr中，


即可使用Node上通过日志代理转发的模式；


优缺点分析：



| 对比 | 说明 |
| --- | --- |
| 优点 | 只需耗费比较少的cpu和内存，共享volume处理效率比较高 |
| 缺点 | 宿主机上存在两份相同的日志，磁盘利用率不高 |









## 应用容器直接输出日志到日志服务


![image.png](https://cdn.nlark.com/yuque/0/2021/png/186661/1613657743964-188827e1-035a-4551-a38b-7263baabebd5.png#align=left&display=inline&height=408&margin=%5Bobject%20Object%5D&name=image.png&originHeight=816&originWidth=1774&size=73842&status=done&style=none&width=887)






适用于有成熟日志系统的场景，日志不需要通过k8s;






# EFK介绍


## fluentd


fluentd是一个统一日志层的开源数据收集器。


flentd允许你统一日志收集并更好的使用和理解数据；




![image.png](https://cdn.nlark.com/yuque/0/2020/png/186661/1608182349724-1ca56be8-bbf3-423a-8fce-6387995e0c55.png#align=left&display=inline&height=263&margin=%5Bobject%20Object%5D&name=image.png&originHeight=263&originWidth=1113&size=553573&status=done&style=none&width=1113)
# 
四大特征：



| **统一日志层**

fluentd隔断数据源，从后台系统提供统一日志层；

 | **简单灵活**
**
提供了500多个插件，连接非常多的数据源和输出源，内核简单； |
| --- | --- |
| **广泛验证**
**
5000多家数据驱动公司以来Fluentd
最大的客户通过它收集5万多台服务器的日志
 | **云原生**
**
是云原生CNCF的成员项目 |



![image.png](https://cdn.nlark.com/yuque/0/2020/png/186661/1608182861145-440046c5-c899-4fd2-869a-4cc13debadae.png#align=left&display=inline&height=341&margin=%5Bobject%20Object%5D&name=image.png&originHeight=341&originWidth=858&size=149222&status=done&style=none&width=858)




4大优势：



| **统一JSON日志**
**
**![image.png](https://cdn.nlark.com/yuque/0/2020/png/186661/1608183319900-34963ca5-8c77-437e-9cb9-c73f8aa735f5.png#align=left&display=inline&height=193&margin=%5Bobject%20Object%5D&name=image.png&originHeight=193&originWidth=307&size=20413&status=done&style=none&width=307)**
**
fluentd尝试采用JSON结构化数据，这就统一了所有处理日志数据的方面，收集，过滤，缓存，输出日志到多目的地，下行流数据处理使用Json更简单，因为它已经有足够的访问结构并保留了足够灵活的scemas；
 |
| --- |
| **插件化架构**
**
**![image.png](https://cdn.nlark.com/yuque/0/2020/png/186661/1608183520827-78b17a6e-8f27-4924-b0e7-d636cf8a73e7.png#align=left&display=inline&height=195&margin=%5Bobject%20Object%5D&name=image.png&originHeight=195&originWidth=341&size=36039&status=done&style=none&width=341)**
**fluntd 有灵活的插件体系允许社区扩展功能，500多个社区贡献的插件连接了很多数据源和目的地； 通过插件，你可以开始更好的使用你的日志** |
| **最小资源消耗**
**![image.png](https://cdn.nlark.com/yuque/0/2020/png/186661/1608183689638-66c1f788-4062-4227-a2b8-a3462dd9e3d4.png#align=left&display=inline&height=196&margin=%5Bobject%20Object%5D&name=image.png&originHeight=196&originWidth=346&size=38403&status=done&style=none&width=346)**
**c和ruby写的，需要极少的系统资源，40M左右的内存可以处理13k/时间/秒 ，如果你需要更紧凑的内存，可以使用Fluent bit ,更轻量的Fluentd** |
| **内核可靠**
**![image.png](https://cdn.nlark.com/yuque/0/2020/png/186661/1608183877490-fd26c801-0a31-430a-885e-ca9b217113d8.png#align=left&display=inline&height=185&margin=%5Bobject%20Object%5D&name=image.png&originHeight=185&originWidth=343&size=39138&status=done&style=none&width=343)**
**
Fluentd支持内存和基于文件缓存，防止内部节点数据丢失；
也支持robust失败并且可以配置高可用模式， 2000多家数据驱动公司在不同的产品中依赖Fluentd，更好的使用和理解他们的日志数据  |





使用fluentd的原因：



| **简单灵活**

10分钟即可在你的电脑上安装fluentd，你可以马上下载它，500多个插件打通数据源和目的地，插件也很好开发和部署； | **开源**
**
**基于Apache2.0证书  完全开源 **
** |
| --- | --- |
| **可靠高性能**

5000多个数据驱动公司的不同产品和服务依赖fluentd，更好的使用和理解数据，实际上，基于datadog的调查，是使用docker运行的排行top7的技术；

一些fluentd用户实时采集上千台机器的数据，每个实例只需要40M左右的内存，伸缩的时候，你可以节省很多内存 | **社区**
**
**fluentd可以改进软件并帮助其它人更好的使用**
 |



大公司使用背书： 微软 ， 亚马逊； pptv ;


![image.png](https://cdn.nlark.com/yuque/0/2020/png/186661/1608185712797-d197f26b-39ce-49ad-9946-8f78a9eb98db.png#align=left&display=inline&height=446&margin=%5Bobject%20Object%5D&name=image.png&originHeight=446&originWidth=450&size=67666&status=done&style=none&width=450)


![image.png](https://cdn.nlark.com/yuque/0/2020/png/186661/1608185691884-034ca0af-b1a9-4c0d-9416-96618dda5d13.png#align=left&display=inline&height=878&margin=%5Bobject%20Object%5D&name=image.png&originHeight=878&originWidth=523&size=101284&status=done&style=none&width=523)






可以结合elasticSearch + kibana来一起组成日志套件；
快速搭建EFK集群并收集应用的日志，配置性能排行榜；
![image.png](https://cdn.nlark.com/yuque/0/2020/png/186661/1608187144425-2b83cb8f-d350-4207-84eb-e29af4f6301c.png#align=left&display=inline&height=349&margin=%5Bobject%20Object%5D&name=image.png&originHeight=349&originWidth=778&size=38088&status=done&style=none&width=778)




## elasticsearch


Elasticsearch 是一个分布式、RESTful 风格的搜索和数据分析引擎，能够解决不断涌现出的各种用例。 作为 Elastic Stack 的核心，它集中存储您的数据，帮助您发现意料之中以及意料之外的情况。

详细介绍：[https://www.elastic.co/guide/cn/elasticsearch/guide/current/foreword_id.html](https://www.elastic.co/guide/cn/elasticsearch/guide/current/foreword_id.html)


## kibana


Kibana 是一款开源的数据分析和可视化平台，它是 Elastic Stack 成员之一，设计用于和 Elasticsearch 协作。您可以使用 Kibana 对 Elasticsearch 索引中的数据进行搜索、查看、交互操作。您可以很方便的利用图表、表格及地图对数据进行多元化的分析和呈现。


Kibana 可以使大数据通俗易懂。它很简单，基于浏览器的界面便于您快速创建和分享动态数据仪表板来追踪 Elasticsearch 的实时数据变化.




详细介绍：[https://www.elastic.co/guide/cn/kibana/current/introduction.html](https://www.elastic.co/guide/cn/kibana/current/introduction.html)


# 容器化EFK实现路径


[https://github.com/kayrus/elk-kubernetes](https://github.com/kayrus/elk-kubernetes)


直接拖代码下来，然后配置后 context, namespace , 即可安装；


```shell
cd elk-kubernetes

./deploy.sh --watch
```


下面是deploy.sh的脚本，可以简单看一下：


```shell
#!/bin/sh

CDIR=$(cd `dirname "$0"` && pwd)
cd "$CDIR"

print_red() {
  printf '%b' "\033[91m$1\033[0m\n"
}

print_green() {
  printf '%b' "\033[92m$1\033[0m\n"
}

render_template() {
  eval "echo \"$(cat "$1")\""
}


KUBECTL_PARAMS="--context=250091890580014312-cc3174dcd4fc14cf781b6fc422120ebd8"
NAMESPACE=${NAMESPACE:-sm}
KUBECTL="kubectl ${KUBECTL_PARAMS} --namespace=\"${NAMESPACE}\""

eval "kubectl ${KUBECTL_PARAMS} create namespace \"${NAMESPACE}\""

#NODES=$(eval "${KUBECTL} get nodes -l 'kubernetes.io/role!=master' -o go-template=\"{{range .items}}{{\\\$name := .metadata.name}}{{\\\$unschedulable := .spec.unschedulable}}{{range .status.conditions}}{{if eq .reason \\\"KubeletReady\\\"}}{{if eq .status \\\"True\\\"}}{{if not \\\$unschedulable}}{{\\\$name}}{{\\\"\\\\n\\\"}}{{end}}{{end}}{{end}}{{end}}{{end}}\"")
NODES=$(eval "${KUBECTL} get nodes -l 'sm.efk=data' -o go-template=\"{{range .items}}{{\\\$name := .metadata.name}}{{\\\$unschedulable := .spec.unschedulable}}{{range .status.conditions}}{{if eq .reason \\\"KubeletReady\\\"}}{{if eq .status \\\"True\\\"}}{{if not \\\$unschedulable}}{{\\\$name}}{{\\\"\\\\n\\\"}}{{end}}{{end}}{{end}}{{end}}{{end}}\"")
ES_DATA_REPLICAS=$(echo "$NODES" | wc -l)

if [ "$ES_DATA_REPLICAS" -lt 3 ]; then
  print_red "Minimum amount of Elasticsearch data nodes is 3 (in case when you have 1 replica shard), you have ${ES_DATA_REPLICAS} worker nodes"
  print_red "Won't deploy more than one Elasticsearch data pod per node exiting..."
  exit 1
fi

print_green "Labeling nodes which will serve Elasticsearch data pods"
for node in $NODES; do
  eval "${KUBECTL} label node ${node} elasticsearch.data=true --overwrite"
done

for yaml in *.yaml.tmpl; do
  render_template "${yaml}" | eval "${KUBECTL} create -f -"
done

for yaml in *.yaml; do
  eval "${KUBECTL} create -f \"${yaml}\""
done

eval "${KUBECTL} create configmap es-config --from-file=es-config --dry-run -o yaml" | eval "${KUBECTL} apply -f -"
eval "${KUBECTL} create configmap fluentd-config --from-file=docker/fluentd/td-agent.conf --dry-run -o yaml" | eval "${KUBECTL} apply -f -"
eval "${KUBECTL} create configmap kibana-config --from-file=kibana.yml --dry-run -o yaml" | eval "${KUBECTL} apply -f -"

eval "${KUBECTL} get pods $@"

```




简单分解一下部署的流程：


![k8s搭建EFK流程.png](https://cdn.nlark.com/yuque/0/2021/png/186661/1613661113280-49089837-119e-4674-bed5-f167db186d49.png#align=left&display=inline&height=1882&margin=%5Bobject%20Object%5D&name=k8s%E6%90%AD%E5%BB%BAEFK%E6%B5%81%E7%A8%8B.png&originHeight=1882&originWidth=2146&size=265502&status=done&style=none&width=2146)


我的k8s环境中没有搭建成功，后续搭建成功了再出详细的安装笔记。




# 小结


一句话概括本篇：EFK是一种通过日志代理客户端采集应用日志比较常用的实现方式。


![让容器日志无处可逃.png](https://cdn.nlark.com/yuque/0/2021/png/186661/1613659919440-db29d3f1-917e-4bbc-9a1c-079a6f8b8043.png#align=left&display=inline&height=2644&margin=%5Bobject%20Object%5D&name=%E8%AE%A9%E5%AE%B9%E5%99%A8%E6%97%A5%E5%BF%97%E6%97%A0%E5%A4%84%E5%8F%AF%E9%80%83.png&originHeight=2644&originWidth=3356&size=827548&status=done&style=none&width=3356)
