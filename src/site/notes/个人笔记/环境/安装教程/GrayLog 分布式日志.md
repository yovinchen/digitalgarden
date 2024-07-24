---
{"dg-publish":true,"permalink":"/个人笔记/环境/安装教程/GrayLog 分布式日志/"}
---

`GrayLog`是一个轻量型的分布式日志管理平台，一个开源的日志聚合、分析、审计、展示和预警工具。在功能上来说，和 ELK类似，但又比 ELK要简单轻量许多。依靠着更加简洁，高效，部署使用简单的优势很快受到许多公司的青睐。

![](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704805912.png)

官网：https://www.graylog.org/

>- `GrayLog`方案的优势
>- 一体化方案，安装方便，不像ELK有3个独立系统间的集成问题。
>- 采集原始日志，并可以事后再添加字段，比如http_status_code，response_time等等。
>
>- 自己开发采集日志的脚本，并用curl/nc发送到`Graylog Server`，发送格式是自定义的GELF，`Flunted`和`Logstash`都有相应的输出GELF消息的插件。自己开发带来很大的自由度。实际上只需要用`inotifywait`监控日志的modify事件，并把日志的新增行用curl/netcat发送到`Graylog Server`就可。
>
>- 搜索结果高亮显示，就像`google`一样。
>
>- 搜索语法简单，比如： `source:mongo AND reponse_time_ms:>5000`，避免直接输入`elasticsearch`搜索json`语法`
>
>- 搜索条件可以导出为`elasticsearch`的搜索json`文本`，方便直接开发调用`elasticsearch rest api`的搜索脚本。
## Graylog工作流程

![img](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704762652.jpeg)

## 最小化安装基本框架

![img](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704762728.png)

流程如下：

- 微服务中的`GrayLog`客户端发送日志到`GrayLog`服务端
- `GrayLog`把日志信息格式化，存储到`Elasticsearch`
- 客户端通过浏览器访问`GrayLog`，`GrayLog`访问`Elasticsearch`

这里`MongoDB`是用来存储`GrayLog`的配置信息的，这样搭建集群时，`GrayLog`的各节点可以共享配置。

- `MongoDB` 用于存储配置元信息和配置数据（无需太多硬件资源配置）
- `Elasticsearch` 用于存储日志数据（内存大以及更高的磁盘IO）
- `Graylog` 用于读取日志、展示日志。（CPU密集）

缺点：

- 系统无冗余、容易出现单点故障，适合测试阶段

## 大型生产环境架构

![img](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704762891.png)

>- 客户访问和日志源面对的是前端的`LB`，`LB`通过`Graylog REST API `负载管理Graylog集群
>- `Elasticsearch` 使用集群方案，多节点存储数据，达到备份冗余、负载的效应
>- `MongoDB` 集群，具备自动的容错功能(auto-failover),自动恢复的(auto-recovery)的高可用
>- 各组件方便伸缩扩展
## 安装部署

使用 docker-compose  最小化安装


```
#部署Elasticsearch
docker run -d \
    --name elasticsearch \
    -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
    -e "discovery.type=single-node" \
    -v es-data:/usr/share/elasticsearch/data \
    -v es-plugins:/usr/share/elasticsearch/plugins \
    --privileged \
    -p 9200:9200 \
    -p 9300:9300 \
elasticsearch:7.17.5

#部署MongoDB（使用之前部署的服务即可）
docker run -d \
--name mongodb \
-p 27017:27017 \
--restart=always \
-v mongodb:/data/db \
-e MONGO_INITDB_ROOT_USERNAME=xlcs \
-e MONGO_INITDB_ROOT_PASSWORD=123321 \
mongo:4.4

#部署
docker run \
--name graylog \
-p 9000:9000 \
-p 12201:12201/udp \
-e GRAYLOG_HTTP_EXTERNAL_URI=http://<IP>:9000/ \
-e GRAYLOG_ELASTICSEARCH_HOSTS=http://<IP>:9200/ \
-e GRAYLOG_ROOT_TIMEZONE="Asia/Shanghai"  \
-e GRAYLOG_WEB_ENDPOINT_URI="http://<IP>:9000/:9000/api" \
-e GRAYLOG_PASSWORD_SECRET="somepasswordpepper" \
-e GRAYLOG_ROOT_PASSWORD_SHA2=8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918 \
-e GRAYLOG_MONGODB_URI=mongodb://xlcs:123321@<IP>:27017/admin \
-d \
graylog/graylog:4.3
```

命令解读：

- 端口信息： 

- - `-p 9000:9000`：`GrayLog`的http服务端口，9000
  - `-p 12201:12201/udp`：`GrayLog`的GELF UDP协议端口，用于接收从微服务发来的日志信息

- 环境变量 

- - `-e GRAYLOG_HTTP_EXTERNAL_URI`：对外开放的ip和端口信息，这里用9000端口
  - `-e GRAYLOG_ELASTICSEARCH_HOSTS`：`GrayLog`依赖于`ES`，这里指定`ES`的地址
  - `-e GRAYLOG_WEB_ENDPOINT_URI`：对外开放的API地址
  - `-e GRAYLOG_PASSWORD_SECRET`：密码加密的秘钥
  - `-e GRAYLOG_ROOT_PASSWORD_SHA2`：密码加密后的密文。明文是`admin`，账户也是`admin`
  - `-e GRAYLOG_ROOT_TIMEZONE="Asia/Shanghai"`：`GrayLog`容器内时区
  - `-e GRAYLOG_MONGODB_URI`：指定`MongoDB`的链接信息

- `graylog/graylog:4.3`：使用的镜像名称，版本为4.3

查看运行状态
![image-20240109092047285](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704763247.png)

访问地址 http://<IP>:9000/ ， 如果可以看到如下界面说明启动成功。

![image-20240109092430238](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704763470.png)

通过 `admin/admin`登录，即可看到欢迎页面，目前还没有数据：

![image-20240109092519114](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704763519.png)

## 简单使用以及介绍

>- `Search` 搜索：该选项允许您按特定条件搜索和筛选日志数据，例如时间范围、源、消息和其他字段。还可以将搜索保存为仪表板小部件或警报。
>- `Streams` 流：流用于实时处理和过滤传入的日志数据。可以根据特定规则和条件创建流，Graylog 将自动对数据进行分类和索引以供进一步分析。
>- `Alerts` 警报：警报允许您为特定事件或条件设置通知。例如，可以设置警报以在出现突然错误峰值或在日志中出现特定消息时通知。 
>- `Dashboards` 仪表板：仪表板允许您以有意义的方式可视化日志数据。可以创建自定义仪表板，其中包含图表、图形和表格，显示想要监控的数据。
>- `Enterprise` 企业版：高级版功能，提供额外的功能，如基于角色的访问控制、LDAP/AD集成和集群，以实现高可用性和可扩展性。
>- `Security` 安全性：该选项提供加密、身份验证和授权等安全功能，以确保日志数据受到保护。
>- `System` 系统：该选项提供系统级别的设置和配置，例如管理用户、设置输入和输出插件以及监控系统性能。
### 1.配置 `Inputs`

部署完成`GrayLog`后，需要配置`Inputs`才能接收微服务发来的日志数据。

第一步，在`System`菜单中选择`Inputs`：

![image-20240109094050433](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704764450.png)

第二步，在页面的下拉选框中，选择`GELF UDP`：

![5037550cc85d85f5fe12861a64843466](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704764674.png)

然后点击`Launch new input`按钮：

![21110bb0182c00ebea37f4e235375fd3](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704764708.png)

点击`save`保存：

![image-20240109094333858](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704764614.png)

可以看到，`GELF UDP Inputs `保存成功。

### 2.整合微服务

现在，GrayLog的服务端日志收集器已经准备好，我们还需要在项目中添加GrayLog的客户端，将项目日志发送到GrayLog服务中，保存到ElasticSearch。

基本步骤如下：

- 引入GrayLog客户端依赖
- 配置Logback，集成GrayLog的Appender
- 启动并测试

导入依赖：

```xml
<dependency>
    <groupId>biz.paluch.logging</groupId>
    <artifactId>logstash-gelf</artifactId>
    <version>1.15.0</version>
</dependency>
```

配置`Logback`，在配置文件中增加 `GELF`的`appender`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- 说明：
    1、日志级别及文件 日志记录采用分级记录，级别与日志文件名相对应，不同级别的日志信息记录到不同的日志文件中。
    2、日志级别可以根据开发环境进行配置，为方便统一管理查看日志，日志文件路径统一由LOG_PATH:-.配置在/home/项目名称/logs
-->
<configuration>
    <!-- 引入默认设置 -->
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <!-- 编码格式设置 -->
    <property name="ENCODING" value="UTF-8"/>
    <!-- 日志文件的存储地址，由application.yml中的logging.path配置，根路径默认同项目路径 -->
<!--    <property name="LOG_HOME" value="${LOG_PATH:-./logs}"/>-->
    <property name="LOG_HOME" value="/Users/yovinchen/Desktop/project/xlcs/xlcs-parent/data/logs"/>
    <property name="LOG_FILE_MAX_SIZE" value="100MB"/>
    <property name="LOG_FILE_MAX_HISTORY" value="180"/>
    <property name="LOG_TOTAL_SIZE_CAP" value="100GB"/>
    <!--应用名称-->
    <springProperty scope="context" name="APP_NAME" source="spring.application.name" defaultValue="springBoot"/>
    <!-- 常规输出格式：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符 -->
    <property name="NORMAL_LOG_PATTERN"
              value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50}.%method@%line - %msg%n"/>
    <!-- 彩色输出格式：magenta：洋红，boldMagenta：粗红，yan：青色，·#══> -->
    <property name="CONSOLE_LOG_PATTERN"
              value="%boldMagenta([%d{yyyy-MM-dd HH:mm:ss.SSS}]) %red([%thread]) %boldMagenta(%-5level) %blue(%logger{20}.%method@%line) %magenta(·#═>) %cyan(%msg%n)"/>

    <!-- ==========================控制台输出设置========================== -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
            <charset>${ENCODING}</charset>
        </encoder>
    </appender>

    <!-- ==========================按天输出DEBUG日志设置========================== -->
    <appender name="DEBUG_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <!--设置文件命名格式-->
            <FileNamePattern>${LOG_HOME}/${APP_NAME}/debug/${POD_NAME}-%d{yyyy-MM-dd}-%i.log</FileNamePattern>
            <!--设置日志文件大小，超过就重新生成文件，默认10M-->
            <maxFileSize>${LOG_FILE_MAX_SIZE}</maxFileSize>
            <!--日志文件保留天数，默认30天-->
            <maxHistory>${LOG_FILE_MAX_HISTORY}</maxHistory>

            <totalSizeCap>${LOG_TOTAL_SIZE_CAP}</totalSizeCap>
        </rollingPolicy>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>DEBUG</level>             <!-- 设置拦截的对象为INFO级别日志 -->
            <onMatch>ACCEPT</onMatch>       <!-- 当遇到了DEBUG级别时，启用该段配置 -->
            <onMismatch>DENY</onMismatch>   <!-- 没有遇到DEBUG级别日志时，屏蔽该段配置 -->
        </filter>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>${NORMAL_LOG_PATTERN}</pattern>
            <charset>${ENCODING}</charset>
        </encoder>
    </appender>

    <!-- ==========================按天输出日志设置========================== -->
    <appender name="INFO_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <!--设置文件命名格式-->
            <FileNamePattern>${LOG_HOME}/${APP_NAME}/info/${POD_NAME}-%d{yyyy-MM-dd}-%i.log</FileNamePattern>
            <!--设置日志文件大小，超过就重新生成文件，默认10M-->
            <maxFileSize>${LOG_FILE_MAX_SIZE}</maxFileSize>
            <!--日志文件保留天数，默认30天-->
            <maxHistory>${LOG_FILE_MAX_HISTORY}</maxHistory>

            <totalSizeCap>${LOG_TOTAL_SIZE_CAP}</totalSizeCap>

        </rollingPolicy>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>             <!-- 设置拦截的对象为INFO级别日志 -->
            <onMatch>ACCEPT</onMatch>       <!-- 当遇到了INFO级别时，启用该段配置 -->
            <onMismatch>DENY</onMismatch>   <!-- 没有遇到INFO级别日志时，屏蔽该段配置 -->
        </filter>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>${NORMAL_LOG_PATTERN}</pattern>
            <charset>${ENCODING}</charset>
        </encoder>
    </appender>

    <!-- ==========================按天输出ERROR级别日志设置========================== -->
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <!--设置文件命名格式-->
            <FileNamePattern>${LOG_HOME}/${APP_NAME}/error/${POD_NAME}-%d{yyyy-MM-dd}-%i.log</FileNamePattern>
            <!--设置日志文件大小，超过就重新生成文件，默认10M-->
            <maxFileSize>${LOG_FILE_MAX_SIZE}</maxFileSize>
            <!--日志文件保留天数，默认30天-->
            <maxHistory>${LOG_FILE_MAX_HISTORY}</maxHistory>

            <totalSizeCap>${LOG_TOTAL_SIZE_CAP}</totalSizeCap>
        </rollingPolicy>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>            <!-- 设置拦截的对象为ERROR级别日志 -->
            <onMatch>ACCEPT</onMatch>       <!-- 当遇到了ERROR级别时，启用该段配置 -->
            <onMismatch>DENY</onMismatch>   <!-- 没有遇到ERROR级别日志时，屏蔽该段配置 -->
        </filter>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>${NORMAL_LOG_PATTERN}</pattern>
            <charset>${ENCODING}</charset>
        </encoder>
    </appender>
    <!--druid慢查询日志输出，没有使用druid监控的去掉这部分以及下面的一个相关logger-->
    <appender name="DRUID_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <!--设置文件命名格式-->
            <FileNamePattern>${LOG_HOME}/${APP_NAME}/druid/SlowSql_${POD_NAME}-%d{yyyy-MM-dd}-%i.log</FileNamePattern>
            <!--设置日志文件大小，超过就重新生成文件，默认10M-->
            <maxFileSize>${LOG_FILE_MAX_SIZE}</maxFileSize>
            <!--日志文件保留天数，默认30天-->
            <maxHistory>${LOG_FILE_MAX_HISTORY}</maxHistory>

            <totalSizeCap>${LOG_TOTAL_SIZE_CAP}</totalSizeCap>
        </rollingPolicy>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>            <!-- 设置拦截的对象为ERROR级别日志 -->
            <onMatch>ACCEPT</onMatch>       <!-- 当遇到了ERROR级别时，启用该段配置 -->
            <onMismatch>DENY</onMismatch>   <!-- 没有遇到ERROR级别日志时，屏蔽该段配置 -->
        </filter>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>${NORMAL_LOG_PATTERN}</pattern>
            <charset>${ENCODING}</charset>
        </encoder>
    </appender>
    <!-- ===日志输出级别，OFF level > FATAL > ERROR > WARN > INFO > DEBUG > ALL level=== -->
    <logger name="com.asiainfo" level="INFO"/>
    <logger name="org.springframework.boot.web.embedded.tomcat.TomcatWebServer" level="INFO"/>
    <logger name="org.springframework" level="WARN"/>
    <logger name="com.baomidou.mybatisplus" level="WARN"/>
    <logger name="org.apache.kafka.clients.NetworkClient" level="INFO"/>
    <logger name="org.apache.kafka.clients.consumer.ConsumerConfig" level="INFO"/>
    <!--druid相关logger，-->
    <logger name="com.alibaba.druid.filter.stat.StatFilter" level="ERROR">
        <appender-ref ref="DRUID_FILE"/>
        <appender-ref ref="CONSOLE"/>
    </logger>
    <appender name="GELF" class="biz.paluch.logging.gelf.logback.GelfLogbackAppender">
        <!--GrayLog服务地址-->
        <host>udp:<IP></host>
        <!--GrayLog服务端口-->
        <port>12201</port>
        <version>1.1</version>
        <!--当前服务名称-->
        <facility>${appName}</facility>
        <extractStackTrace>true</extractStackTrace>
        <filterStackTrace>true</filterStackTrace>
        <mdcProfiling>true</mdcProfiling>
        <timestampPattern>yyyy-MM-dd HH:mm:ss,SSS</timestampPattern>
        <maximumMessageSize>8192</maximumMessageSize>
    </appender>

    <!-- ======开发环境：打印控制台和输出到文件====== -->
    <springProfile name="dev"><!-- 由application.yml中的spring.profiles.active配置 -->
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="INFO_FILE"/>
            <appender-ref ref="ERROR_FILE"/>
            <appender-ref ref="GELF"/>
        </root>
    </springProfile>

    <!-- ======测试环境：打印控制台和输出到文件====== -->
    <springProfile name="test"><!-- 由application.yml中的spring.profiles.active配置 -->
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="DEBUG_FILE"/>
            <appender-ref ref="INFO_FILE"/>
            <appender-ref ref="ERROR_FILE"/>
            <appender-ref ref="GELF"/>
        </root>
    </springProfile>

    <!-- ======生产环境：打印控制台和输出到文件====== -->
    <springProfile name="prod"><!-- 由application.yml中的spring.profiles.active配置 -->
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="DEBUG_FILE"/>
            <appender-ref ref="INFO_FILE"/>
            <appender-ref ref="ERROR_FILE"/>
            <appender-ref ref="GELF"/>
        </root>
    </springProfile>
</configuration>

```

点击`search`按钮即可看到日志数据：

![image-20240109095535652](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704765336.png)

### 3.日志回收策略

到此graylog的基础配置就算完成了，已经可以收到日志数据。

但是在实际工作中，服务日志会非常多，这么多的日志，如果不进行存储限制，那么不久就会占满磁盘，查询变慢等等，而且过久的历史日志对于实际工作中的有效性也会很低。

Graylog则自身集成了日志数据限制的配置，可以通过如下进行设置：

![image-20240109145002566](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704783002.png)

选择`Default index set`的`Edit`按钮：

![38fba6a5b77842928d69838c6e366c62](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704785689.png)

GrayLog有3种日志回收限制，触发以后就会开始回收空间，删除索引：

![2c6218061870d65547522aebf22335e3](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704785718.png)

分别是：

- `Index Message Count`：按照日志数量统计，默认超过`20000000`条日志开始清理
- `Index Size`：按照日志大小统计，默认超过`1GB`开始清理
- `Index Message Count`：按照日志日期清理，默认日志存储1天

### 4.搜索语法

在search页面，可以完成基本的日志搜索功能：

![c645ec766d0f2eb6109fab1344d04cd1](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704785655.png)

```shell
#不指定字段，默认从message字段查询
输入：undo

#输入两个关键字，关系为or
undo 统计

#加引号是需要完整匹配
"undo 统计"

#指定字段查询，level表示日志级别，ERROR（3）、WARNING（4）、NOTICE（5）、INFO（6）、DEBUG（7）
level: 6

#或条件
level:(6 OR 7)
```

更多查询官网文档：https://docs.graylog.org/docs/query-language

### 5.自定义展示字段

![bd999948509ec96203230c1a1139ce78](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704785625.png)

效果如下：

![image-20240109153728442](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704785848.png)

### 6.日志统计仪表盘

GrayLog支持把日志按照自己需要的方式形成统计报表，并把许多报表组合一起，形成DashBoard（仪表盘），方便对日志统计分析。

创建仪表盘

![image-20240109153845240](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704785925.png)

![9585c83b65fc665c3a9db3055c997c8f](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704788275.png)

![image-20240109153955165](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704785995.png)

在这里创建小组件
![f29f62d315bee87d182a570f408da69c](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704788310.png)





![image-20240109161518566](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704788119.png)

![image-20240109161531180](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704788131.png)

![image-20240109161550617](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704788151.png)

![image-20240109161648891](https://lsky.hhdxw.top/imghub/2024/01/image-202401091704788209.png)
