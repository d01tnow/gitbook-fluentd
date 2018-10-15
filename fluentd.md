# Fluentd 安装配置简介

开源数据收集服务

## 安装

推荐使用 Docker 方式安装部署

### docker-compose

下面的例子中使用 @type tail 插件作为 source

``` yml
version: '3.3'
services:
  fluentd-nginx:
    image: harbor.iibu.com/fluentd:v2.0

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      ## 只读方式挂载配置文件.
      ## 源目录是部署环境相关的, 可修改.
      ## 目标目录不可修改.
      - $PWD/fluentd/config/fluentd.conf:/etc/fluentd/fluentd.conf:ro

      ## 挂载日志目录, fluentd 在该目录下监控数据文件变更.
      ## 源目录是部署环境相关的, 可修改.
      ## 目标目录需要和 fluentd.conf 中 <source>.path 中指定的一致.
      ## 如果要监测多个目录下的数据文件,有两种方案
      ## 1. 只映射一个目录, 该目录作为数据文件的根目录.
      ## 2. 映射多个目录
      - /var/log/nginx/:/fluentd/log/

      ## 挂载 pos 文件存放目录.
      ## 源目录是部署环境相关的, 可修改.
      ## 目标目录需要和 fluentd.conf 中 <source>.pos_file 指定的一致.
      - $PWD/fluentd/pos/:/fluentd/pos/

    environment:
      ## 部署环境相关. 如果有多个 fluentd 实例, 最好每个实例使用唯一的名称.
      ## 用于传递真实主机名给 fluentd 的 record_transformer 插件, 环境变量名称需要同 <filter nginx>
      ## @type record_transformer <record> hostname "#{ENV['HOST_NAME']" </record></filter> 中, 'HOST_NAME' 一致
      - HOST_NAME=nginx-node1

    logging:
      options:
        ## 控制 fluentd 自身日志量, 最多3个文件, 每个文件最大 100 MB
        max-size: '100M'
        max-file: '3'

```

## 配置

部署人员可以重点关注[部署环境相关配置信息](#部署环境相关配置信息)

配置文件格式:

``` conf
<source>
</source>
<filter>
</filter>
<match **>
</match>
```

### 指令

* system: 系统范围的配置信息
* source: 该指令指明输入资源
* filter: 事件处理管道
* match: 输出, 处理经过管道过滤后的数据
* label: 将内部路由的 filter 和 match 分组, 方便管理复杂任务
* @include: 重用已有的配置文件

### 配置信息

### 部署环境相关配置信息

简单地, 全文搜索 **部署环境**

#### docker-compose.yml

参见 [docker-compose](#docker-compose). 主要是 volumes 和 environment.

#### fluentd.conf

``` conf
<system>
  ## 指定进程名称
  process_name fluentd-nginx
</system>
<source>
  ## 使用内置插件 tail 处理输入源. [官方文档](https://docs.fluentd.org/v1.0/articles/in_tail)
  @type tail

  ## 部署环境相关. 多个路径以 ',' 分隔
  ## 支持通配符 '*' 和 ruby 的 strftime 格式: %Y - 年, e.g.: 2018; %m - 月, e.g.: 04; %d - 日, e.g.: 09
  ## 使用通配符时需要注意命名规则是否包含了转储的文件, 如果包含那么会造成重复上传数据.
  ## 使用下面配置时, fluentd 监控目录: /fluend/log/ 下的 *.log 文件, 以及(假设当前系统时间为: 2018-09-05) /fluentd/log/20180905/ 目录下的 *.log 文件
  path /fluentd/log/*.log, /fluentd/log/%Y%m%d/*.log

  ## 从监控列表中排除某些文件, 格式为数组.
  # exclude_path ["", ""]

  ## 部署环境相关. 位置文件存放位置. 很重要. fluentd 用它记录监控文件的处理位置.
  ## 如果使用 Docker 方式部署, 则仅需要修改映射卷的源即可. /source/pos/:/fluentd/pos/
  ## 一个 <source> 一个 pos_file. 不要在多个 <source> 配置中共享 pos_file.
  ## 如果目录下有多个 pos_file, 注意修改下面配置中的文件名. 避免同名.
  pos_file /fluentd/pos/log.pos

  ## fluentd 会上传文件的路径, 这个参数指定路径的键值.
  path_key log_path

  ## 用于后续处理的标签, 命名规则: 多个单词, 以'.' 分隔
  tag log.nginx

  ## 刷新监控列表的时间间隔. 单位: 秒. 默认: 60 秒
  refresh_interval 15

  ## 从 pos_file 中记录位置开始读. 默认: false, 即从文件尾巴开始读.
  ## 避免文件未被监控, 或者 fluentd 服务重启过程造成部分数据丢失, 建议启用该选项.
  read_from_head true

  ## 解析器
  ## 如果数据文件每一行都是 json 序列化的字符串, 简单的使用 @type json 插件即可
  <parse>
    @type regexp
    expression ^(?<remote>[^ ]*) (?<host>[^ ]*) (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*) +\S*)?" (?<code>[^ ]*) (?<size>[^ ]*) ("(?<referrer>[^\"]*)" "(?<agent>[^\"]*)")? "(?<xforwarded>[^\"]*)" "--elapsed=(?<elapsed>[\d\.]*)s" "(?<up_addr>[^\"]*)"?$
    time_format %d/%b/%Y:%H:%M:%S %z
    types elapsed:float, code:integer, size:integer
  </parse>
</source>

## filter 指令. 匹配上面 <source> 中的 tag 标签. 支持通配符. 比如: log.*
<filter log.nginx>
  ## record_transformer - 用于处理记录的插件
  @type record_transformer

  ## 启用 ruby 语法
  enable_ruby true

  ## 自动类型转换
  auto_typecast true

  ## 增加的记录
  <record>
    ## 增加键为: elapsed, 值为 <source> 处理后的记录中 "elapsed" 的值 * 1000.
    ## 因为 nginx 默认输出的 'elapsed' 单位是秒. * 1000 后单位是毫秒
    elapsed ${record["elapsed"] * 1000}
    hostname "#{ENV['HOST_NAME']}"
  </record>
</filter>

## output 指令. 匹配上面的 tag 标签. 支持通配符. 比如: log.*
<match log.nginx>
  ## 输出到 elasticsearch
  @type elasticsearch

  ## 部署环境相关. 指明 ip:port. ',' 分隔.
  hosts 192.168.43.106:9200,192.168.43.107

  ## 索引名称使用 logstash 格式. 即 prefix-%Y.%m.%d. 默认的 prefix 为 logstash.
  logstash_format true
  ## 指定 索引名称的前缀
  logstash_prefix candy

  ## 是否包含 tag 键, 默认: false
  include_tag_key true

  ## 存储的 _type 键名, 默认是 fluentd, ES7.0 只能是 _doc
  type_name _doc
</match>

## 上面的 macth 都不匹配进入到这个指令
<match **>
  ## 输出到 stdout
  @type stdout
</match>
```
