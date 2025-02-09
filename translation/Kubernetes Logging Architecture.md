记录架构

应用程序和系统日志可以帮助您了解在群集内部发生的情况。日志在调试问题和监视集群活动方面特别有用。大多数现代应用都拥有某种记录机制;无独有偶，大多数容器引擎的设计都类似地支持某种类型的日志记录。容器化应用程序最简单且最受欢迎的日志记录方法是写入stdout（标准输出流）和stderr（标准错误流）这两种。

然而，容器引擎或运行时提供的本机功能通常不足以构建完整的日志记录解决方案。例如，如果容器崩溃，pod被驱逐，或节点死亡，但是您仍然希望如往常一样存取应用程序的日志，因此，完整的日志记录解决方案的构建就迫在眉睫。因此，日志应具有独立于节点，pod或容器的单独存储和生命周期的新特性。这个概念经过更新之后的日志被称为集群级日志记录。群集级日志记录需要单独的后端来实现存储，分析和查询功能。

Kubernetes虽然不提供日志数据的本机存储解决方案，但您可以将许多现有的日志记录解决方案集成到Kubernetes集群中。

- Kubernetes的基本日志记录
- 记录节点级别
- 集群级日志记录体系结构

假设集群内部或外部存在日志记录后端，它将记录集群级日志记录体系结构。即使您对集群级日志记录不感兴趣，但您仍可能会在节点上找到有关如何存储和处理日志的说明，以便使用。





## Kubernetes的基本日志记录

在本节中，您可以看到Kubernetes中基本日志记录的示例，该示例会显示如何将数据输出到标准输出流。此演示使用了pod规范和容器，该容器每秒向标准输出写入一些文本。

```
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox
    args: [/bin/sh, -c,'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done']
```

要运行此pod，请使用以下命令：

```
kubectl apply -f https://k8s.io/examples/debug/counter-pod.yaml
pod/counter created
```

要获取日志，请使用以下kubectl logs命令：

```
kubectl logs counter
0: Mon Jan  1 00:00:00 UTC 2001
1: Mon Jan  1 00:00:01 UTC 2001
2: Mon Jan  1 00:00:02 UTC 2001
...
```

如果容器已崩溃，您可以使用kubectl logs从带有--previous（之前）标志的容器在先前实例化中检索日志。如果您的pod有多个容器，您应通过在命令中附加容器名称来指定要访问的容器的日志。相关详细信息，请参阅[kubectllogs](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#logs)文档。



### 记录节点级别

![](https://upload-images.jianshu.io/upload_images/18436006-515268beec1270fc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

任一容器应用程序都会会使用stdout（标准输出）写入，使用stderr（标准错误）处理，然后通过容器引擎向重定向到某处。例如，Docker容器引擎会将这两个流（stdout和stderr）重定向到日志驱动程序，该驱动程序在Kubernetes中，并以json格式配置和写入文件。

```
注意： Docker json日志记录驱动程序会将每行都视为一个独立的消息。即Docker日志记录驱动程序并不能直接支持将多行消息整合为一行。您需要在日志代理级别或更高级别的日志上实现处理多行消息。
```

在默认情况下，如果容器重新启动，则kubelet会根据其日志保留一个已终止的容器。如果从节点中删除出pod，则其相对应的容器，以及容器的日志也会被删除。

节点级日志记录中的一个重要考虑因素是实现日志轮换，这样日志就不会消耗掉节点上所有可用的存储。Kubernetes目前并不能实现轮换日志功能，但部署工具可以建立一个解决方案来实现这个功能。例如，在kube-up.sh脚本部署的Kubernetes集群中，有一个logrotate 工具其配置为每小时运行一次。您还可以将容器运行状态设置为自动循环，以此来记录应用程序日志，例如Docker log-opt即可实现此功能。在kube-up.sh脚本中，后一种方法用于GCP上的COS映像，而前一种方法可用于其他任何环境。在这两种情况的默认模式中，当其日志文件超过10MB时会启用循环功能。

例如，您可以kube-up.sh在相应的脚本中找到有关如何在GCP上设置COS映像日志记录的详细信息。

当您在基本日志记录示例中运行kubectl logs时，节点上的kubelet会处理请求并直接从日志文件中读取相关信息，并返回其所响应的相应内容。

```
注意：目前，如果某些外部系统已经执行了这次循环，那么只有最后一个日志文件的内容可以通过kubectl logs日志获得。如果有一个为10MB的文件，logrotate将执行循环，如果有两个文件，一个为10MB大小，另一个为空，kubectl logs将返回一个空响应。
```



### 系统组件日志

系统组件有两种类型：一为在容器中运行的系统组件，二为不在容器中运行的系统组件。

例如：

- Kubernetes调度程序和kube-proxy在容器中运行；
- Kubelet和容器运行时（比如Docker）不在容器中运行。

在有systemd的机器上，kubelet和容器会在运行时写入日志。如果systemd不存在，它们将会写入到.log目录中的/var/log文件中。容器内的系统组件总是直接写入到/var/log目录中，并不调用默认的日志记录机制。他们使用klog 日志库。您可以在日志记录的开发文档中找到用于记录这些组件约定的优先级。

与容器日志类似，/var/log中的系统组件日志也会被循环。在kube-up.sh脚本提供的Kubernetes集群中，这些日志被配置为每天由logrotate工具实现循环记录，或者当日志文件大小超过100MB时也进行循环记录。





## 集群级日志记录体系结构

虽然Kubernetes没有为集群级日志记录提供本机解决方案，但您可以考虑几种常见方法。

以下是一些选项：

- 使用在每个节点上运行的节点级日志记录代理；
- 包括用于登录应用程序窗格的专用sidecar容器；
- 将日志直接从应用程序中推送到后端。

### 使用节点日志记录代理

![](https://upload-images.jianshu.io/upload_images/18436006-8587b8a8d5cac6b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

您可以通过使用在每个节点上包含的节点级日志记录代理来实现群集级日志记录。日志记录代理是一种专用工具，用于公开日志或将日志推送到后端。通常，日志记录代理程序是一个容器，可以访问包含该节点中所有应用程序容器日志文件的目录。

由于日志记录代理必须在每个节点上运行，因此将其实现为节点上的DaemonSet副本，清单窗格或专用本机进程是很常见的。然而，后两种方法均被弃用且非常不建议使用。

对于Kubernetes集群，使用节点级日志记录代理是最常见和最建议的，因为它每个节点只创建一个代理，并且不需要对节点上运行的应用程序进行任何更改。但是，节点级日志记录仅适用于应用程序的标准输出和标准错误。

Kubernetes没有指定日志代理，但是有两个可选的日志代理与Kubernetes版本一起被打包：Stackdriver Logging用于Google Cloud Platform和Elasticsearch。您可以在专用文档中找到更多信息和说明。两者均使用流畅的自定义配置来作为节点上的代理。



## 使用带有日志代理的sidecar容器

您可以通过以下方式之一使用sidecar容器：

- sidecar容器将应用程序日志流式的传输到自己的日志并实现标准输出;
- sidecar容器运行日志记录代理程序，该代理程序配置为从应用程序容器中获取日志;

### 流式sidecar集装箱

通过让您的sidecar容器传输到到他们自己的日志上并实现stdout（标准输出）和stderr（标准错误）功能，您可以使用已经在每个节点上运行的kubelet和日志记录代理功能。sidecar容器从文件、套接字或日志中读取日志信息。每个单独的sidecar容器将日志打印到其自己stdout（标准输出）和stderr（标准错误）流。

这种方法允许您从应用程序的不同部分分离多个日志流，其中一些可能缺乏对写入stdout（标准输出）和写入stderr（标准错误）的支持。重定向日志背后的逻辑是最小的，因此它几乎不会花费多少开销。另外，因为 stdout（标准输出）和stderr（标准错误）是由kubelet处理，所以你可以使用内置工具kubectl logs来实现相应功能。

请考虑以下示例:pod运行单个容器，容器使用两种不同的格式写入两个不同的日志文件。这是Pod的配置文件：

```
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
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  volumes:
  - name: varlog
    emptyDir: {}
```

即使您设法将两个组件的容器流重定向为stdout（标准输出），在同一个日志流中包含不同格式的日志也会是一团糟。相反，你可以引入两个边车容器。每个sidecar容器都可以从共享卷中拖出特定的日志文件，然后将日志重定向到自己的stdout（标准输出）流。

这是一个包含两个sidecar容器的pod的配置文件：

```
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
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-log-1
    image: busybox
    args: [/bin/sh, -c, 'tail -n+1 -f /var/log/1.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-log-2
    image: busybox
    args: [/bin/sh, -c, 'tail -n+1 -f /var/log/2.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  volumes:
  - name: varlog
emptyDir: {}
```

现在，当您运行此pod时，可以通过运行以下命令单独访问每个日志流：

```
kubectl logs counter count-log-1
```

```
0: Mon Jan  1 00:00:00 UTC 2001
1: Mon Jan  1 00:00:01 UTC 2001
2: Mon Jan  1 00:00:02 UTC 2001
...
```

```
kubectl logs counter count-log-2
```

```
Mon Jan  1 00:00:00 UTC 2001 INFO 0
Mon Jan  1 00:00:01 UTC 2001 INFO 1
Mon Jan  1 00:00:02 UTC 2001 INFO 2
...
```

群集中安装的节点级代理会自动获取这些日志流，并不需要进行进一步配置。如果您愿意，您可以将代理配置为根据源容器解析日志行。

请注意，尽管CPU和内存使用率较低（cpu为几毫微秒，内存为几兆字节），但将日志写入文件然后将其流式传输stdout（标准输出）会使磁盘使用量增加一倍。如果您有一个写入单个文件的应用程序，通常最好将其设置为 /dev/stdout目标而不是实现流式sidecar容器方法。

sidecar容器还可用于循环应用程序本身无法循环的日志文件。这种方法的一个例子就是定期运行logrotate的小容器。但是，建议直接使用stdout（标准输出）和stderr（标准错误）轮换并且将保留策略留给kubelet。





## 带有日志记录代理的Sidecar容器

![](https://upload-images.jianshu.io/upload_images/18436006-80f0d5604597f5f9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果节点级日志记录代理程序不够灵活，您可以创建一个带有单独日志记录代理程序的sidecar容器，该代理程序可以专门配置为与您的应用程序一起运行。

```
注意：在sidecar容器中使用日志记录代理会消耗大量资源。此外，因为它们不受kubelet控制，您将无法使用kubectl logs命令访问这些日志。
```

举个例子，你可以使用的Stackdriver，它使用fluentd作为记录剂。以下是两个可用于实现此方法的配置文件。第一个文件包含配置流利的ConfigMap。

```
CODES:
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluentd.conf: |
    <source>
      type tail
      format none
      path /var/log/1.log
      pos_file /var/log/1.log.pos
      tag count.format1
    </source>

    <source>
      type tail
      format none
      path /var/log/2.log
      pos_file /var/log/2.log.pos
      tag count.format2
    </source>

    <match **>
      type google_cloud
    </match>
```

注意： fluentd的配置超出了本文的范围。有关配置流利的信息，请参阅[官方流利文档](https://docs.fluentd.org/)

第二个文件描述了一个具有流畅的边车容器的pod。该pod可以安装一个容量，其中熟悉它的人可以获取其配置数据。

```
CODES:
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
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
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
    volumeMounts:
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

一段时间后，您可以在Stackdriver界面中找到相关日志消息。

请记住，这只是一个示例，您实际上可以用流畅的使用任何日志代理，从应用程序容器内的任何源读取。



### 直接从应用程序中公开日志

![](https://upload-images.jianshu.io/upload_images/18436006-8cf7c48086ac3873.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

您可以直接通过从每个应用程序公开或推送日志来实现集群级的日志记录;但是，这种日志记录机制的实现超出了Kubernetes的控制范围。





