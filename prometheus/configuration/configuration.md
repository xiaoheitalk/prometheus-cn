Prometheus通过命令行标志和配置文件进行配置。 虽然命令行标志配置了不可变的系统参数（例如存储位置，保留在磁盘和内存中的数据量等），但配置文件定义了与抓取作业及其实例相关的所有内容，以及哪些规则文件 载入。

要查看所有可用的命令行参数，执行`./prometheus -h`

Prometheus可以在运行时重新加载其配置。 如果新配置格式不正确，则不会应用更改。 通过向Prometheus进程发送SIGHUP或向`/-/reload`端点发送HTTP POST请求（启用`--web.enable-lifecycle`标志时）来触发配置重新加载。 这也将重新加载任何已配置的规则文件。

##### 一、配置文件
要指定要加载的配置文件，请使用`--config.file`标志。

该文件以YAML格式编写，由下面描述的方案定义。 括号表示参数是可选的。 对于非列表参数，该值设置为指定的默认值。

通用占位符定义如下：

- `<boolean>`：一个可以取值为`true`或`false`的布尔值
- `<duration>`：与正则表达式匹配的持续时间`[0-9] +（ms | [smhdwy]）`
- `<labelname>`：与正则表达式匹配的字符串`[a-zA-Z _] [a-zA-Z0-9 _] *`
- `<labelvalue>`：一串unicode字符
- `<filename>`：当前工作目录中的有效路径
- `<host>`：由主机名或IP后跟可选端口号组成的有效字符串
- `<path>`：有效的URL路径
- `<scheme>`：一个可以取值http或https的字符串
- `<string>`：常规字符串
- `<secret>`：一个秘密的常规字符串，例如密码
- `<tmpl_string>`：在使用前进行模板扩展的字符串

其他占位符是单独指定的。

[可以在此处找到有效的示例文件。](https://github.com/prometheus/prometheus/blob/release-2.8/config/testdata/conf.good.yml)

全局配置指定在所有其他配置上下文中有效的参数。 它们还可用作其他配置节的默认值。
```
global:
  # 抓取目标实例的频率时间值，默认1分钟
  [ scrape_interval: <duration> | default = 1m ]

  # 一次抓取请求超时时间值，默认10s
  [ scrape_timeout: <duration> | default = 10s ]

  #  执行配置文件规则的频率时间值, 默认1m
  [ evaluation_interval: <duration> | default = 1m ]

  # 当和外部系统通信时(federation, remote storage, Alertmanager), 这些标签会增加到度量指标数据中
  #（联合，远程存储，Alertma# nager）.
  external_labels:
    [ <labelname>: <labelvalue> ... ]

# 规则文件指定了一个globs列表. 
# 规则文件指定规则文件路径列表。规则和警报是从所有匹配的文件中读取的
rule_files:
  [ - <filepath_glob> ... ]

# 抓取配置列表.
scrape_configs:
  [ - <scrape_config> ... ]

# 警报指定与Alertmanager相关的设置.
alerting:
  alert_relabel_configs:
    [ - <relabel_config> ... ]
  alertmanagers:
    [ - <alertmanager_config> ... ]

# 与远程写入功能相关的设置.
remote_write:
  [ - <remote_write> ... ]

# 与远程读取功能相关的设置.
remote_read:
  [ - <remote_read> ... ]
```
###### 1.1 `<scrape_config>`
`<scrape_config>`部分指定一组描述如何刮除它们的目标和参数。 在一般情况下，一个scrape配置指定单个作业。 在高级配置中，这可能会改变。

目标可以通过`<static_configs>`参数静态配置，也可以使用其中一种支持的服务发现机制动态发现。

此外，`<relabel_configs>`允许在抓取之前对任何目标及其标签进行高级修改。

其中`<job_name>`在所有scrape配置中必须是唯一的。
```
# 默认分配给已抓取指标的job名称。
job_name: <job_name>

# 从job中抓取目标的频率.
[ scrape_interval: <duration> | default = <global_config.scrape_interval> ]

# 抓取此job时，每次抓取超时时间.
[ scrape_timeout: <duration> | default = <global_config.scrape_timeout> ]

# 从目标列表中抓取度量指标的http资源路径, 默认为/metrics
[ metrics_path: <path> | default = /metrics ]

# honor_labels控制Prometheus如何处理已经存在于已抓取数据中的标签与Prometheus将附加服务器端的标签之间的冲突（"job"和"instance"标签，手动配置的目标标签以及服务发现实现生成的标签）。
# 
# 如果honor_labels设置为"true"，则通过保留已抓取数据的标签值并忽略冲突的服务器端标签来解决标签冲突。
#
# 如果honor_labels设置为"false"，则通过将已抓取数据中的冲突标签重命名为"exported_ <original-label>"（例如"exported_instance"，"exported_job"）然后附加服务器端标签来解决标签冲突。 这对于联合等用例很有用，其中应保留目标中指定的所有标签。
# 
# 请注意，任何全局配置的"external_labels"都不受此设置的影响。 在与外部系统通信时，它们始终仅在时间序列尚未具有给定标签时应用，否则将被忽略。
# 
[ honor_labels: <boolean> | default = false ]

# 配置用于请求的协议方案, 默认为http请求
[ scheme: <scheme> | default = http ]

# 可选的HTTP URL参数.
params:
  [ <string>: [<string>, ...] ]

# 使用配置的用户名和密码在每个scrape请求上设置`Authorization`标头。 password和password_file是互斥的。
basic_auth:
  [ username: <string> ]
  [ password: <secret> ]
  [ password_file: <string> ]

# 使用配置的承载令牌在每个scrape请求上设置`Authorization`标头。 它`bearer_token_file`和是互斥的。
[ bearer_token: <secret> ]

# 使用配置的承载令牌在每个scrape请求上设置`Authorization`标头。 它`bearer_token`和是互斥的。
[ bearer_token_file: /path/to/bearer/token/file ]

# 配置scrape请求的TLS设置.
tls_config:
  [ <tls_config> ]

# 可选的代理URL.
[ proxy_url: <string> ]

# Azure服务发现配置列表.
azure_sd_configs:
  [ - <azure_sd_config> ... ]

# Consul服务发现配置列表.
consul_sd_configs:
  [ - <consul_sd_config> ... ]

# DNS服务发现配置列表。
dns_sd_configs:
  [ - <dns_sd_config> ... ]

# EC2服务发现配置列表。
ec2_sd_configs:
  [ - <ec2_sd_config> ... ]

# OpenStack服务发现配置列表。
openstack_sd_configs:
  [ - <openstack_sd_config> ... ]

# 文件服务发现配置列表。
file_sd_configs:
  [ - <file_sd_config> ... ]

# GCE服务发现配置列表。
gce_sd_configs:
  [ - <gce_sd_config> ... ]

# Kubernetes服务发现配置列表。
kubernetes_sd_configs:
  [ - <kubernetes_sd_config> ... ]

# Marathon服务发现配置列表。
marathon_sd_configs:
  [ - <marathon_sd_config> ... ]

# AirBnB的神经服务发现配置列表。
nerve_sd_configs:
  [ - <nerve_sd_config> ... ]

# Zookeeper Serverset服务发现配置列表。
serverset_sd_configs:
  [ - <serverset_sd_config> ... ]

# Triton服务发现配置列表。
triton_sd_configs:
  [ - <triton_sd_config> ... ]

# 此job的标记静态配置目标列表。
static_configs:
  [ - <static_config> ... ]

# 目标重新标记配置列表。
relabel_configs:
  [ - <relabel_config> ... ]

# 度量标准重新配置列表。
metric_relabel_configs:
  [ - <relabel_config> ... ]

# 对每个将被接受的样本数量的每次抓取限制。
# 如果在度量重新标记后存在超过此数量的样本，则整个抓取将被视为失败。 0表示没有限制。
[ sample_limit: <int> | default = 0 ]
```

###### 1.2 `<tls_config>`
`tls_config`允许配置TLS连接。

```
# 用于验证API服务器证书的CA证书。
[ ca_file: <filename> ]

# 用于服务器的客户端证书身份验证的证书和密钥文件。
[ cert_file: <filename> ]
[ key_file: <filename> ]

# ServerName扩展名，用于指示服务器的名称。
# https://tools.ietf.org/html/rfc4366#section-3.1
[ server_name: <string> ]

# 禁用服务器证书的验证。
[ insecure_skip_verify: <boolean> ]
```

###### 1.3 <azure_sd_config>

**Azure SD正处于测试阶段：在未来的版本中，仍然可能对配置进行实质性修改**

Azure SD配置允许从Azure虚拟机中检索和获取目标。

下面的测试标签在relabeling期间在目标上仍然是可用的：

- `__meta_azure_machine_id`: 机器ID
- `__meta_azure_machine_location`: 机器运行的位置
- `__meta_azure_machine_name`: 机器名称
- `__meta_azure_machine_private_ip`: 机器的内网IP
- `__meta_azure_machine_resource_group`: 机器的资源组
- `__meta_azure_tag_<tagname>`: 机器的每个tag值

对于Azure发现，看看下面的配置选项：

```
# The information to access the Azure API.
# The subscription ID.
subscription_id: <string>
# The tenant ID.
tenant_id: <string>
# The client ID.
client_id: <string>
# The client secret.
client_secret: <string>

# Refresh interval to re-read the instance list.
[ refresh_interval: <duration> | default = 300s ]

# The port to scrape metrics from. If using the public IP address, this must
# instead be specified in the relabeling rule.
[ port: <int> | default = 80 ]

```

###### 1.4 <consul_sd_config>

Consul服务发现配置允许从Consul's Catalog API中检索和获取目标。

下面的meta标签在relabeling期间在目标上仍然是可用的：

- `__meta_consul_address`: 目标地址
- `__meta_consul_dc`: 目标的数据中心名称
- `__meta_consul_node`: 目标的节点名称
- `__meta_consul_service_address`: 目标的服务地址
- `__meta_consul_service_id`: 目标的服务ID
- `__meta_consul_service_port`: 目标的服务端口
- `__meta_consul_service`: 这个目标属于哪个服务名称
- `__meta_consul_tags`: 由标签分隔符链接的目标的标签列表

```
# 下面配置是访问Consul API所需要的信息
server: <host>
[ token: <string> ]
[ datacenter: <string> ]
[ scheme: <string> ]
[ username: <string> ]
[ password: <string> ]

# 指定对于某个目标的服务列表被检测， 如果省略，所有服务被抓取
services:
  [ - <string> ]

# The string by which Consul tags are joined into the tag label.
[ tag_separator: <string> | default = , ]

```

注意：用于获取目标的IP和PORT，被组装到`<__meta_consul_address>:<__meta_consul_service_port>`。然而，在一些Consul创建过程中，这个相关地址在`__meta_consul_service_address`。在这些例子中，你能使用[relabel](https://prometheus.io/docs/operating/configuration/#relabel_config)特性去替换指定的`__address__`标签。

###### 1.5 `<dns_sd_config>`

基于DNS的服务发现配置允许指定一组DNS域名，这些域名会定期查询以发现目标列表。 要联系的DNS服务器从`/etc/resolv.conf`中读取。

此服务发现方法仅支持基本的DNS A，AAAA和SRV记录查询，但不支持RFC6763中指定的高级DNS-SD方法。

在重新标记阶段，元标签`__meta_dns_name`在每个目标上可用，并设置为生成已发现目标的记录名称。
```
# 要查询的DNS域名列表。
names:
  [ - <domain_name> ]

# 要执行的DNS查询的类型。
[ type: <query_type> | default = 'SRV' ]

# 查询类型不是SRV时使用的端口号。
[ port: <number>]

# 提供名称后刷新的时间。
[ refresh_interval: <duration> | default = 30s ]
```
其中`<domain_name>`是有效的DNS域名。 其中`<query_type>`是SRV，A或AAAA。

###### 1.6 <ec2_sd_config>

EC2 SD配置允许从AWS EC2实例中检索目标。默认情况下用内网IP地址, 但是在relabeling期间可以改变成公网ID地址。

下面meta标签在relabeling期间在目标上是可用的：

- `__meta_ec2_availability_zone`: 正在运行的实例的可用域。
- `__meta_ec2_instance_id`: EC2的实例ID
- `__meta_ec2_instance_state`: EC2的实例状态
- `__meta_ec2_instance_type`: EC2的实例类型
- `__meta_ec2_private_ip`: 如果存在，表示内网IP的地址
- `__meta_ec2_public_dns_name`: 如果可用，表示实例的公网DNS名称
- `__meta_ec2_public_ip`: 如果可用，表示实例的公网IP地址
- `__meta_ec2_subnet_id`: 如果可用，表示子网IDs的列表。
- `__meta_ec2_tag_<tagkey>`: 这个实例的tag值
- `__meta_ec2_vpc_id`: 如果可用，表示正在运行的实例的VPC的ID

对于EC2 discovery，看看下面的配置选项：

```
# 访问EC2 API的信息

# AWS域
region: <string>

# AWS API keys. 如果空白，环境变量`AWS_ACCESS_KEY_ID`和`AWS_SECRET_ACCESS_KEY`可以被使用
[ access_key: <string> ]
[ secret_key: <string> ]
# Named AWS profile used to connect to the API.
[ profile: <string> ]

# Refresh interval to re-read the instance list.
[ refresh_interval: <duration> | default = 60s ]

# The port to scrape metrics from. If using the public IP address, this must
# instead be specified in the relabeling rule.
[ port: <int> | default = 80 ]
```

###### 1.7 <file_sd_config>

基于文件的服务发现提供了一些通用方法去配置静态目标，以及作为插件自定义服务发现机制的接口。

它读取包含零个或者多个`<static_config>s`的一些文件。通过磁盘监视器检测对所有定义文件的更改，并立即应用。文件可能以YAML或JSON格式提供。只应用于形成良好目标群体的变化。

这个JSON文件必须包含静态配置的列表，使用这个格式：

```
[
  {
    "targets": [ "<host>", ... ],
    "labels": {
      "<labelname>": "<labelvalue>", ...
    }
  },
  ...
]
```

文件内容也可以通过周期性刷新时间重新加载。

在标签重构阶段，每个目标有一个meta标签`__meta_filepath`。它的值被设置成从目标中提取的文件路径。

```
# Patterns for files from which target groups are extracted.
files:
  [ - <filename_pattern> ... ]

# Refresh interval to re-read the files.
[ refresh_interval: <duration> | default = 5m ]
```

`filename_pattern`可以是以`.json, .yml, .yaml`结尾。最后路径段可以包含单个`*`，它匹配任何字符顺序，例如: `my/path/tg_*.json`。

在`v0.20`, `names`: 用`files:`代替。

###### 1.8 <gce_sd_config>

**GCE SD在测试中：在将来版本中，配置可能会有实质性变化。**

从GCP GCE实例中，GCE SD配置允许检索和获取目标。这个内网IP地址被默认使用，但是在relabeling期间，这个公网IP地址可能会发生变化。

在relabeling期间，下面的meta标签在目标上是可用的：

- `__meta_gce_instance_name`: 实例名称
- `__meta_gce_metadata_<name>`: 实例每一个metadata项
- `__meta_gce_network`: 实例的网络
- `__meta_gce_private_ip`: 实例的内网IP
- `__meta_gce_project`: 正在运行的GCP项目
- `__meta_gce_public_ip`: 如果存在，表示GCP的公网IP地址
- `__meta_gce_subnetwork`: 实例的子网
- `__meta_gce_tags`: 实例的tag列表
- `__meta_gce_zone`: 正在运行的实例的GCE区域

对于GCE discovery，看看下面的配置选项：

```
# The information to access the GCE API.

# The GCP Project
project: <string>

# The zone of the scrape targets. If you need multiple zones use multiple
# gce_sd_configs.
zone: <string>

# Filter can be used optionally to filter the instance list by other criteria
[ filter: <string> ]

# Refresh interval to re-read the instance list
[ refresh_interval: <duration> | default = 60s ]

# The port to scrape metrics from. If using the public IP address, this must
# instead be specified in the relabeling rule.
[ port: <int> | default = 80 ]

# The tag separator is used to separate the tags on concatenation
[ tag_separator: <string> | default = , ]
```

Google Cloud SDK默认客户端通过查找一下位置发现凭据，优先选择找到的第一个位置：

1. 由GOOGLE_APPLICATION_CREENTIALS环境变量指定的JSON文件
2. 一个JSON文件在大家都熟悉的路径下：$HOME/.config/gclooud/application_default_credentials.json
3. 从GCE元数据服务器获取

如果Prometheus运行在GCE上，关联这个正在运行的实例的服务账号，应该至少可以从计算资源上有读取数据的权限。如果运行在GCE外面，需要确保创建一个合适的服务账号，并把证书文件放置在指定的某个地方。





###### 1.9 `<kubernetes_sd_config>`

Kubernetes SD配置允许从[Kubernetes](https://kubernetes.io/)的RESTAPI中检索scrape目标，并始终与群集状态保持同步。

可以配置以下`role`类型之一来发现目标：
1. `node`
  `node`角色发现每个群集节点有一个目标，其地址默认为Kubelet的HTTP端口。 目标地址默认为`NodeInternalIP`，`NodeExternalIP`，`NodeLegacyHostIP`和`NodeHostName`的地址类型顺序中Kubernetes节点对象的第一个现有地址。

  可用元标签：

- `__meta_kubernetes_node_name`：节点对象的名称。
- `__meta_kubernetes_node_label_ <labelname>`：节点对象中的每个标签。
` `__meta_kubernetes_node_annotation_<annotationname>`：节点对象中的每个注释。
- `__meta_kubernetes_node_address_<address_type>`：每个节点地址类型的第一个地址（如果存在）。
- 
此外，节点的`instance`标签将设置为从API服务器检索的节点名称。

2. `service`
  `service`角色为每个服务发现每个服务端口的目标。 这对于服务的黑盒监控通常很有用。 该地址将设置为服务的Kubernetes DNS名称和相应的服务端口。

  可用元标签：

- `__meta_kubernetes_namespace`：服务对象的命名空间。
- `__meta_kubernetes_service_annotation_<annotationname>`：服务对象的注释。
- `__meta_kubernetes_service_cluster_ip`：服务的群集IP地址。 （不适用于ExternalName类型的服务）
- `__meta_kubernetes_service_external_name`：服务的DNS名称。 （适用于ExternalName类型的服务）
- `__meta_kubernetes_service_label_ <labelname>`：服务对象的标签。
- `__meta_kubernetes_service_name`：服务对象的名称。
- `__meta_kubernetes_service_port_name`：目标服务端口的名称。
- `__meta_kubernetes_service_port_number`：目标的服务端口号。
- `__meta_kubernetes_service_port_protocol`：目标服务端口的协议。

3. `pod`
  `pod`角色发现所有`pod`并将其容器暴露为目标。 对于容器的每个声明端口，将生成单个目标。 如果容器没有指定端口，则会创建每个容器的无端口目标，以通过重新标记手动添加端口。

  可用元标签：

- `__meta_kubernetes_namespace`：`pod`对象的命名空间。
- `__meta_kubernetes_pod_name`：`pod`对象的名称。
- `__meta_kubernetes_pod_ip`：`pod`对象的`pod IP`。
- `__meta_kubernetes_pod_label_ <labelname>`：`pod`对象的标签。
- `__meta_kubernetes_pod_annotation_ <annotationname>`：`pod`对象的注释。
- `__meta_kubernetes_pod_container_name`：目标地址指向的容器的名称。
- `__meta_kubernetes_pod_container_port_name`：容器端口的名称。
- `__meta_kubernetes_pod_container_port_number`：容器端口号。
- `__meta_kubernetes_pod_container_port_protocol`：容器端口的协议。
- `__meta_kubernetes_pod_ready`：对于`pod`的就绪状态，设置为`true`或`false`。
- `__meta_kubernetes_pod_phase`：在生命周期中设置为`Pending`，`Running`，`Succeeded`，`Failed`或`Unknown`。
- `__meta_kubernetes_pod_node_name`：将`pod`安排到的节点的名称。
- `__meta_kubernetes_pod_host_ip`：`pod`对象的当前主机`IP`。
- `__meta_kubernetes_pod_uid`：`pod`对象的`UID`。
- `__meta_kubernetes_pod_controller_kind`：对象类型的`pod`控制器。
- `__meta_kubernetes_pod_controller_name`：`pod`控制器的名称。

4. `endpoints`
  `endpoints`角色从列出的服务端点发现目标。 对于每个端点地址，每个端口发现一个目标。 如果端点由`pod`支持，则`pod`的所有其他容器端口（未绑定到端点端口）也会被发现为目标。

  可用元标签：

- `__meta_kubernetes_namespace`：端点对象的命名空间。
- `__meta_kubernetes_endpoints_name`：端点对象的名称。对于直接从端点列表中发现的所有目标（不是从底层`pod`中另外推断的那些），附加以下标签：
- `__meta_kubernetes_endpoint_ready`：对端点的就绪状态设置为`true`或`false`。
- `__meta_kubernetes_endpoint_port_name`：端点端口的名称。
- `__meta_kubernetes_endpoint_port_protocol`：端点端口的协议。
- `__meta_kubernetes_endpoint_address_target_kind`：端点地址目标的种类。
- `__meta_kubernetes_endpoint_address_target_name`：端点地址目标的名称。
如果端点属于某个服务，则会附加角色：服务发现的所有标签。
对于由`pod`支持的所有目标，将附加角色的所有标签：`pod`发现。

5. `ingress`
  `ingress`角色发现每个入口的每个路径的目标。 这通常用于黑盒监控入口。 地址将设置为入口规范中指定的主机。

  可用元标签：

- `__meta_kubernetes_namespace`：入口对象的名称空间。
- `__meta_kubernetes_ingress_name`：入口对象的名称。
- `__meta_kubernetes_ingress_label_ <labelname>`：入口对象的标签。
- `__meta_kubernetes_ingress_annotation_<annotationname>`：入口对象的注释。
- `__meta_kubernetes_ingress_scheme`：入口的协议方案，如果设置了TLS配置，则为https。 默认为http。
- `__meta_kubernetes_ingress_path`：来自入口规范的路径。 默认为/。

有关Kubernetes发现的配置选项，请参见下文：
```
# 访问Kubernetes API的信息。

# API服务器地址。 如果保留为空，则假定Prometheus在集群内部运行并自动发现API服务器，并在/var/run/secrets/kubernetes.io/serviceaccount/上使用pod的CA证书和不记名令牌文件。
[ api_server: <host> ]

# 应该被发现的实体的Kubernetes角色。
role: <role>

# 用于向API服务器进行身份验证的可选身份验证信息。请注意，`basic_auth`，`bearer_token`和`bearer_token_file`选项是互斥的.password和password_file是互斥的。

# 可选的HTTP基本认证信息。
basic_auth:
  [ username: <string> ]
  [ password: <secret> ]
  [ password_file: <string> ]

# 可选的承载令牌认证信息。
[ bearer_token: <secret> ]

# 可选的承载令牌文件认证信息。
[ bearer_token_file: <filename> ]

# 可选的代理URL。
[ proxy_url: <string> ]

# TLS配置。
tls_config:
  [ <tls_config> ]

# 可选命名空间发现 如果省略，则使用所有名称空间。
namespaces:
  names:
    [ - <string> ]
```
其中`<role>`必须是`endpoints`，`service`，`pod`，`node`或`ingress`。

有关为Kubernetes配置Prometheus的详细[示例](https://github.com/prometheus/prometheus/blob/release-2.8/documentation/examples/prometheus-kubernetes.yml)，请参阅此示例Prometheus配置文件。

您可能希望查看第三方Prometheus[操作](https://github.com/coreos/prometheus-operator)，它可以在Kubernetes上自动执行Prometheus设置。

###### 1.10 <marathon_sd_config>

**Marathon SD正在测试中：在将来的版本中配置可能会有实质性的变化**

Marathon SD配置使用[Marathon](https://mesosphere.github.io/marathon/)REST API允许检索和获取目标。Prometheus将会定期地检查当前运行的任务REST端点，以及对每个app创建一个目标组，这个app至少有一个健康的任务。

在relabeling期间，下面的meta标签在目标机上是可用的：

- `__meta_marathon_app`: app的名称
- `__meta_marathon_image`: 正在使用的Docker镜像名称
- `__meta_marathon_task`: Mesos任务ID
- `__meta_marathon_app_label_<labelname>`: 附加在app上的Marathon标签

对于Marathon发现，详见下面的配置选项：

```
# List of URLs to be used to contact Marathon servers.
# You need to provide at least one server URL, but should provide URLs for
# all masters you have running.
servers:
  - <string>

# Polling interval
[ refresh_interval: <duration> | default = 30s ]
```

默认情况下，在Markdown的每个列出的app会被Prometheus抓取。如果不是所有提供Prometheus度量指标，你能使用一个Marathon标签和Prometheus relabeling去控制实际过程中被获取的实例。默认情况下所有的app也会以Prometheus系统中的一个任务的形式显示出来，这可以通过使用relabeling改变这些。

###### 1.11 <nerve_sd_config>

从存储在Zookeeper中的AirBnB's Nerve上，Nerve SD配置允许检索和获取目标。

在relabeling期间，下面的meta标签在目标上是可用的：

- `__meta_nerve_path`: 在Zookeeper集群中的端节点全路径
- `__meta_nerve_endpoint_host`: 端点的IP
- `__meta_nerve_endpoint_port`: 端点的端口
- `__meta_nerve_endpoint_name`: 端点的名称

```
# The Zookeeper servers.
servers:
  - <host>
# Paths can point to a single service, or the root of a tree of services.
paths:
  - <string>
[ timeout: <duration> | default = 10s ]
```

###### 1.12 <serverset_sd_config>

Serverset SD配置允许检索和获取从存储在Zookeeper中的Serversetsd的目标。Servesets由[Finagle](https://twitter.github.io/finagle/)和[Aurora](http://aurora.apache.org/)经常使用。

在relabeling期间，下面的meta标签在目标上是可用的：

- `__meta_serverset_path`: 在zookeeper里的serverset成员的全路径
- `__meta_serverset_endpoint_host`: 默认端点的host
- `__meta_serverset_endpoint_port`: 默认端点的端口
- `__meta_serverset_endpoint_host_<endpoint>`: 给定端点的host
- `__meta_serverset_endpoint_port_<endpoint>`: 给定端点的port
- `__meta_serverset_shard`: 成员的分片数
- `__meta_serverset_status`: 成员的状态

```
# The Zookeeper servers.
servers:
  - <host>
# Paths can point to a single serverset, or the root of a tree of serversets.
paths:
  - <string>
[ timeout: <duration> | default = 10s ]
```

Serverset数据必须是JSON格式，Thrift格式当前不被支持

###### 1.13 <triton_sd_config>

** Triton SD正在测试中：在将来的版本中配置可能会有实质性的变化**

[Triton](https://github.com/joyent/triton) SD配置允许从容器监控发现端点的目标中检索和获取。

在relabeling期间，下面的meta标签在目标上是可用的：

- `__meta_triton_machine_id`: 目标容器的UUID
- `__meta_triton_machine_alias`: 目标容器的别名
- `__meta_triton_machine_image`: 目标容器的镜像类型
- `__meta_triton_machine_server_id`: 目标容器的服务UUID

```
# The information to access the Triton discovery API.

# The account to use for discovering new target containers.
account: <string>

# The DNS suffix which should be applied to target containers.
dns_suffix: <string>

# The Triton discovery endpoint (e.g. 'cmon.us-east-3b.triton.zone'). This is
# often the same value as dns_suffix.
endpoint: <string>

# The port to use for discovery and metric scraping.
[ port: <int> | default = 9163 ]

# The interval which should should be used for refreshing target containers.
[ refresh_interval: <duration> | default = 60s ]

# The Triton discovery API version.
[ version: <int> | default = 1 ]

# TLS configuration.
tls_config:
  [ <tls_config> ]
```

###### 1.14 `<static_config>`

static_config允许指定目标列表和它们的公共标签集。 这是在scrape配置中指定静态目标的规范方法。
```
# 静态配置指定的目标。
targets:
  [ - '<host>' ]

# 分配给从目标中已抓取的所有指标的标签。
labels:
  [ <labelname>: <labelvalue> ... ]
```
###### 1.15 `<relabel_config>`
重新标记是一种强大的工具，可以在抓取目标之前动态重写目标的标签集。 每个抓取配置可以配置多个重新标记步骤。 它们按照它们在配置文件中的出现顺序应用于每个目标的标签集。

最初，除了配置的每目标标签之外，目标的作业标签设置为相应的scrape配置的`job_name`值。 `__address__`标签设置为目标的`<host>：<port>`地址。 重新标记后，如果在重新标记期间未设置实例标签，则实例标签默认设置为`__address__`的值。 `__scheme__`和`__metrics_path__`标签分别设置为目标的方案和度量标准路径。 `__param_ <name>`标签设置为名为`<name>`的第一个传递的URL参数的值。

在重新标记阶段，可以使用带有`__meta_`前缀的附加标签。 它们由提供目标的服务发现机制设置，并在不同机制之间变化。

在目标重新标记完成后，将从标签集中删除以`__`开头的标签。

如果重新标记步骤仅需临时存储标签值（作为后续重新标记步骤的输入），请使用`__tmp`标签名称前缀。 保证Prometheus本身不会使用此前缀。
```
# 源标签从现有标签中选择值。 它们的内容使用已配置的分隔符进行连接，并与已配置的正则表达式进行匹配，以进行替换，保留和删除操作。
[ source_labels: '[' <labelname> [, ...] ']' ]

# 分隔符放置在连接的源标签值之间。
[ separator: <string> | default = ; ]

# 在替换操作中将结果值写入的标签。
# 替换操作是强制性的。 正则表达式捕获组可用。
[ target_label: <labelname> ]

# 与提取的值匹配的正则表达式。
[ regex: <regex> | default = (.*) ]

# 采用源标签值的散列的模数。
[ modulus: <uint64> ]

# 如果正则表达式匹配，则执行正则表达式替换的替换值。 正则表达式捕获组可用。
[ replacement: <string> | default = $1 ]

# 基于正则表达式匹配执行的操作。
[ action: <relabel_action> | default = replace ]
```
`<regex>`是任何有效的RE2正则表达式。 它是`replace`，`keep`，`drop`，`labelmap`，`labeldrop`和`labelkeep`操作所必需的。 正则表达式固定在两端。 要取消锚定正则表达式，请使用。`* <regex>.*`。

`<relabel_action>`确定要采取的重新签名行动：
- `replace`：将`regex`与连接的`source_labels`匹配。 然后，将`target_label`设置为`replacement`，将匹配组引用（`${1}`，`${2}`，...）替换为其值。 如果正则表达式不匹配，则不进行替换。
- `keep`：删除`regex`与连接的`source_labels`不匹配的目标。
- `drop`：删除`regex`与连接的`source_labels`匹配的目标。
- `hashmod`：将`target_label`设置为连接的`source_labels`的哈希模数。
- `labelmap`：将`regex`与所有标签名称匹配。 然后将匹配标签的值复制到替换时给出的标签名称，替换为匹配组引用（`${1}`，`{2}`，...）替换为其值。
- `labeldrop`：将`regex`与所有标签名称匹配。匹配的任何标签都将从标签集中删除。
- `labelkeep`：将`regex`与所有标签名称匹配。任何不匹配的标签都将从标签集中删除。

必须小心使用`labeldrop`和`labelkeep`，以确保在删除标签后仍然对指标进行唯一标记。

###### 1.16 `<metric_relabel_configs>`
度量重新标记应用于样本，作为摄取前的最后一步。 它具有与目标重新标记相同的配置格式和操作。 度量标准重新标记不适用于自动生成的时间序列，例如`up`。

一个用途是将黑名单时间序列列入黑名单，这些时间序列太昂贵而无法摄取。

###### 1.17 `<alert_relabel_configs>`
警报重新标记在发送到Alertmanager之前应用于警报。 它具有与目标重新标记相同的配置格式和操作。 外部标签后应用警报重新标记。

这样做的一个用途是确保具有不同外部标签的HA对Prometheus服务器发送相同的警报。

###### 1.18 `<alertmanager_config>`
`alertmanager_config`部分指定Prometheus服务器向其发送警报的Alertmanager实例。 它还提供参数以配置如何与这些Alertmanagers进行通信。

Alertmanagers可以通过`static_configs`参数静态配置，也可以使用其中一种支持的服务发现机制动态发现。

此外，`relabel_configs`允许从发现的实体中选择Alertmanagers，并对使用的API路径提供高级修改，该路径通过`__alerts_path__`标签公开。

```
# 推送警报时按目标Alertmanager超时。
[ timeout: <duration> | default = 10s ]

# 将推送HTTP路径警报的前缀。
[ path_prefix: <path> | default = / ]

# 配置用于请求的协议方案。
[ scheme: <scheme> | default = http ]

# 使用配置的用户名和密码在每个请求上设置`Authorization`标头。 password和password_file是互斥的。
basic_auth:
  [ username: <string> ]
  [ password: <string> ]
  [ password_file: <string> ]

# 使用配置的承载令牌在每个请求上设置“Authorization”标头。 它与`bearer_token_file`互斥。
[ bearer_token: <string> ]

# 使用配置的承载令牌在每个请求上设置“Authorization”标头。 它与`bearer_token`互斥。
[ bearer_token_file: /path/to/bearer/token/file ]

# 配置scrape请求的TLS设置。
tls_config:
  [ <tls_config> ]

# 可选的代理URL。
[ proxy_url: <string> ]

# Azure服务发现配置列表。
azure_sd_configs:
  [ - <azure_sd_config> ... ]

# Consul服务发现配置列表。
consul_sd_configs:
  [ - <consul_sd_config> ... ]

# DNS服务发现配置列表。
dns_sd_configs:
  [ - <dns_sd_config> ... ]

# ECS服务发现配置列表。
ec2_sd_configs:
  [ - <ec2_sd_config> ... ]

# 文件服务发现配置列表。
file_sd_configs:
  [ - <file_sd_config> ... ]

# GCE服务发现配置列表。
gce_sd_configs:
  [ - <gce_sd_config> ... ]

# K8S服务发现配置列表。
kubernetes_sd_configs:
  [ - <kubernetes_sd_config> ... ]

# Marathon服务发现配置列表。
marathon_sd_configs:
  [ - <marathon_sd_config> ... ]

# AirBnB's Nerve 服务发现配置列表。
nerve_sd_configs:
  [ - <nerve_sd_config> ... ]

# Zookepper服务发现配置列表。
serverset_sd_configs:
  [ - <serverset_sd_config> ... ]

# Triton服务发现配置列表。
triton_sd_configs:
  [ - <triton_sd_config> ... ]

# 标记为静态配置的Alertmanagers列表。
static_configs:
  [ - <static_config> ... ]

# Alertmanager重新配置列表。
relabel_configs:
  [ - <relabel_config> ... ]
```

###### 1.19 `<remote_write>`
`write_relabel_configs`是在将样本发送到远程端点之前应用于样本的重新标记。 在外部标签之后应用写入重新标记。 这可用于限制发送的样本。

有一个如何使用此功能的小型[演示](https://github.com/prometheus/prometheus/tree/release-2.8/documentation/examples/remote_storage)。
```
# 要发送样本的端点的URL.
url: <string>

# 对远程写端点的请求超时。
[ remote_timeout: <duration> | default = 30s ]

# 远程写入重新标记配置列表。
write_relabel_configs:
  [ - <relabel_config> ... ]

# 使用配置的用户名和密码在每个远程写请求上设置`Authorization`标头.password和password_file是互斥的。
basic_auth:
  [ username: <string> ]
  [ password: <string> ]
  [ password_file: <string> ]

# 使用配置的承载令牌在每个远程写请求上设置`Authorization`头。 它与`bearer_token_file`互斥。
[ bearer_token: <string> ]

# 使用配置的承载令牌在每个远程写请求上设置`Authorization`头。 它与`bearer_token`互斥。
[ bearer_token_file: /path/to/bearer/token/file ]

# 配置远程写入请求的TLS设置。
tls_config:
  [ <tls_config> ]

# 可选的代理URL。
[ proxy_url: <string> ]

# 配置用于写入远程存储的队列。
queue_config:
  # 在我们开始删除之前每个分片缓冲的样本数。
  [ capacity: <int> | default = 10000 ]
  # 最大分片数，即并发数。
  [ max_shards: <int> | default = 1000 ]
  # 最小分片数，即并发数。
  [ min_shards: <int> | default = 1 ]
  # 每次发送的最大样本数。
  [ max_samples_per_send: <int> | default = 100]
  # 样本在缓冲区中等待的最长时间。
  [ batch_send_deadline: <duration> | default = 5s ]
  # 在可恢复错误上重试批处理的最大次数。
  [ max_retries: <int> | default = 3 ]
  # 初始重试延迟。 每次重试都会加倍。
  [ min_backoff: <duration> | default = 30ms ]
  # 最大重试延迟。
  [ max_backoff: <duration> | default = 100ms ]
```
有一个与此功能[集成](https://prometheus.io/docs/operating/integrations/#remote-endpoints-and-storage)的列表。
###### 1.20 `<remote_read`
```
# 要发送样本的端点的URL.
url: <string>

# 可选的匹配器列表，必须存在于选择器中以查询远程读取端点。
required_matchers:
  [ <labelname>: <labelvalue> ... ]

# 对远程读取端点的请求超时。
[ remote_timeout: <duration> | default = 1m ]

# 本地存储应该有完整的数据。
[ read_recent: <boolean> | default = false ]

# 使用配置的用户名和密码在每个远程写请求上设置`Authorization`标头.password和password_file是互斥的。
basic_auth:
  [ username: <string> ]
  [ password: <string> ]
  [ password_file: <string> ]

# 使用配置的承载令牌在每个远程写请求上设置`Authorization`头。 它与`bearer_toke_filen`互斥。
[ bearer_token: <string> ]

# 使用配置的承载令牌在每个远程写请求上设置`Authorization`头。 它与`bearer_token`互斥。
[ bearer_token_file: /path/to/bearer/token/file ]

# 配置远程写入请求的TLS设置。
tls_config:
  [ <tls_config> ]

# 可选的代理URL。
[ proxy_url: <string> ]
```
有一个与此功能[集成](https://prometheus.io/docs/operating/integrations/#remote-endpoints-and-storage)的列表。
