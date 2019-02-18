# SpringCloud On Kubernetes

Written By 

​       Hensh / xu.huang@100credit.com

[TOC]

## 一、背景

百融业务大部分部署在SpringCloud，SpringCloud天生支持负载均衡、熔断、服务发现等功能，但是服务的可扩展性、容灾性、资源隔离性上略显不足，特别是在业务上升、流量大幅提升的情况上越是呈现其不足。

Kubernetes天生的动态扩展、容灾处理、资源隔离等特性正是使得其成为SpringCloud的一大强力助手，而且Kubernetes是经过大量企业验证的可靠产品，一次迁移，多年受益。

基于以上论述，故SpringCloud On Kubernetes是明智之选。



## 二、SpringCloud迁移流程概述

#### 1 微服务改造（适配kubernetes）

运行在Kubernetes上的应用需要以容器方式制作、运行，所以涉及到服务镜像的制作（dockerfile）和编排（deployment.yml）。另外，由于容器和物理机的差异，也涉及到微服务适配Kubernetes的一些优化。

#### 2 Jenkins配置（流程化）

改造好的微服务怎么部署呢？

Jenkins提供代码的打包、镜像的制作等一序列过程，所以Jenkins是流程化的不二之选。

#### 3 Marmot部署（自动化）

制作好的镜像怎么运行？以后如何上线？

Marmot通过调用Jenkins执行流程完成镜像制作，并通过调用Kubernetes API执行创建容器应用等操作，所以Marmot还是流程自动化的发动机。



## 三、SpringCloud微服务改造

#### 1 依赖Jar打入主应用

##### 1.1 描述

- SpringCloud中主应用Jar包和依赖Jar包是分家的，依赖Jar包提前放入公共目录
- 本次迁移需要把依赖Jar包和主应用Jar包合并成一个，不再依赖公共目录

##### 1.2 原由

- 容器资源本身具有隔离性，为隔离性诞生的细粒度而努力，不建议依赖外部公共包
- 容器扩展性非常强，当节点上百上千时，公共目录可能变得冗余而庞大，粒度太粗不易管理
- 每个应用依赖的Jar包不会很大（几十M左右）

##### 1.3 改造

修改POM使用Shade打包（强制要求）

```xml
	<build>
		<resources>
			<resource>
				<directory>src/main/resources</directory>
				<excludes>
					<exclude>**/*.*</exclude>
				</excludes>
			</resource>
		</resources>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-shade-plugin</artifactId>
				<dependencies>
					<dependency>
						<groupId>org.springframework.boot</groupId>
						<artifactId>spring-boot-maven-plugin</artifactId>
						<version>1.2.7.RELEASE</version>
					</dependency>
				</dependencies>
				<configuration>
					<keepDependenciesWithProvidedScope>true</keepDependenciesWithProvidedScope>
					<createDependencyReducedPom>true</createDependencyReducedPom>
					<filters>
						<filter>
							<artifact>*:*</artifact>
							<excludes>
								<exclude>META-INF/*.SF</exclude>
								<exclude>META-INF/*.DSA</exclude>
								<exclude>META-INF/*.RSA</exclude>
							</excludes>
						</filter>
					</filters>
				</configuration>
				<executions>
					<execution>
						<phase>package</phase>
						<goals>
							<goal>shade</goal>
						</goals>
						<configuration>
							<transformers>
								<transformer
									implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
									<resource>META-INF/spring.handlers</resource>
								</transformer>
								<transformer
									implementation="org.springframework.boot.maven.PropertiesMergingResourceTransformer">
									<resource>META-INF/spring.factories</resource>
								</transformer>
								<transformer
									implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
									<resource>META-INF/spring.schemas</resource>
								</transformer>
							</transformers>
						</configuration>
					</execution>
				</executions>
			</plugin>
		</plugins>
		<defaultGoal>compile</defaultGoal>
	</build>
```

##### 1.4 注意

~~由于可运行Jar包使用java -cp命令，因为依赖外部配置，但要成功运行，必须在Jar包中生成"META-INF\MANIFEST.MF"文件，并且文件中必须有Main-Class，通过Main-Class找到运行的主类（如果Main-Class没有主类，运行时/POM指定，无效），使用Shade就可以达到这个目的。~~

==使用"spring-boot-maven-plugin"打包无法使用java -cp运行（启动脚本默认使用java -cp，cp可以添加外部依赖配置）。==



#### 2 日志配置文件修改

##### 2.1 描述

- 日志目录为"/opt/SpringCloud/logs/服务名/服务所在容器名/"
- ”服务名“、”服务所在容器名“ 由环境变量引入（deployment.yml）
- 日志输出级别 由环境变量引入（deployment.yml）

##### 2.2 原由

一个物理节点可能同时运行多个相同的应用（因为Pod副本数量 可以大于 物理机节点数），而一个物理机上多个相同应用的日志不做区分的情况下可能写入同一个文件，就会造成日志混乱，传统的SpringCloud方式必然就不适用，那怎么区分一个物理机上多个相同应用的日志，方便快速定位问题呢？

Kubernetes中一个应用（Pod）的多个副本都会有不同的名字（PodName），可以通过PodName来区分，但是使用PodName有个缺点，每次重新部署，PodName都会变化，较多的部署会导致日志目录较多，目前的解决办法是运维定期执行清理脚本，删除七天前的日志目录。

在SpringCloud中，预发环境或线上环境经常面临修改日志级别适应不同问题的过程，而日志级别 由环境变量引入后，不再依赖具体日志配置文件，为一次打包、一切环境可用而努力。

##### 2.3 改造

参考如下logback.xml，主要引入三个变量：${NAME}、${POD_NAME}、${LOG_LEVEL}

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- 此处的debug指是否打印logback内部日志信息 -->
<configuration debug="false" scan="true" scanPeriod="10 seconds">
    <!-- 应用名称 -->
    <property name="APP_NAME" value="${NAME}" />
    <property name="POD_NAME" value="${POD_NAME}" />
    <property name="LOG_LEVEL" value="${LOG_LEVEL}" />
    <!--日志文件的保存路径,首先查找系统属性-Dlog.dir,如果存在就使用其；否则，在当前目录下创建名为logs目录做日志存放的目录 -->
    <property name="LOG_HOME" value="/opt/SpringCloud/logs/${APP_NAME}/${POD_NAME}" />
    <!-- 日志输出格式 -->
    <property name="ENCODER_PATTERN"
        value="%d{yyyy-MM-dd HH:mm:ss.SSS} [${POD_NAME}] [%thread] %-5level [%logger] -%msg%n" />
    <contextName>${POD_NAME}</contextName>

    <!-- 控制台日志：输出全部日志到控制台 -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <Pattern>${ENCODER_PATTERN}</Pattern>
        </encoder>
    </appender>

    <!-- 文件日志：输出全部日志到文件 -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_HOME}/${POD_NAME}.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_HOME}/${POD_NAME}.log.%d{yyyy-MM-dd}
            </fileNamePattern>
            <maxHistory>7</maxHistory>
        </rollingPolicy>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>${ENCODER_PATTERN}</pattern>
        </encoder>
    </appender>

    <!-- 错误日志：用于将错误日志输出到独立文件 -->
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_HOME}/${POD_NAME}_error.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_HOME}/${POD_NAME}_error.log.%d{yyyy-MM-dd}
            </fileNamePattern>
            <maxHistory>7</maxHistory>
        </rollingPolicy>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>${ENCODER_PATTERN}</pattern>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>
    </appender>

    <logger name="com.br" level="${LOG_LEVEL}" />
    <logger name="org.springframework.boot.context.embedded.tomcat" level="INFO" />

    <!--TRACE, DEBUG, INFO, WARN, ERROR, ALL and OFF -->
    <root>
        <level value="WARN" />
        <appender-ref ref="STDOUT" />
        <appender-ref ref="FILE" />
        <appender-ref ref="ERROR_FILE" />
    </root>
</configuration>
```

##### 2.4 注意

==容器应用日志目录非物理机日志目录==，因为容器日志目录由GFS挂载，挂载目录有变化，物理机访问目录为：/opt/gfsmnt/logs（对应容器/opt/SpringCloud/logs目录）。



#### 3 启动配置文件修改

##### 3.1 描述

- spring.application.name=${NAME}
- 服务名 由环境变量引入（deployment.yml）

##### 3.2 原由

- 保持服务名的一致性
- 为一处修改多处适用而努力

##### 3.3 改造

参考如下application.yml（部分配置），主要引入${NAME}变量

```yaml
spring:
  application:
    name: ${NAME}
  aop:  
    proxyTargetClass: true
server:
  port: 31003
  tomcat:
    uri-encoding: UTF-8
eureka:
  instance:
    appname: ${spring.application.name}
    virtualHostName: ${spring.application.name}
    secureVirtualHostName: ${spring.application.name}
    nonSecurePort: ${server.port}
```



#### 4 配置文件目录修改

##### 4.1 描述

- 在src/main/resources目录下新建pre（预发）、sim（准生产）、prod（生产）三个目录
- 把对应环境的配置文件放入对应目录
- 配置切换(CONF_ENV) 由环境变量引入（deployment.yml）

##### 4.2 原由

- 目前SpringCloud通过不同环境物理机放置不同配置文件做区分，虽然和应用本身解耦，但是和物理环境紧密相关
- 我们要为隔离性带来的 细粒度 而努力，不希望把 配置文件放到物理机
- 通过新建三个环境目录，在运行时通过环境变量切换，是不是很方便？

那么怎么做到的呢？

很简单：通过dockerfile在镜像过程中，把不同环境变量的配置文件放到不同目录，运行时指定要加载的环境目录即可（/opt/SpringCloud/app/服务名/config/pre、/opt/SpringCloud/app/服务名/config/sim、/opt/SpringCloud/app/服务名/config/prod）。不同的环境由外部环境变量控制（CONF_ENV）。

##### 4.3 改造

resources新建三个目录：

```html
src/main/resources
                  /pre

                  /sim

                  /prod
```

##### 4.4 注意

==由于resources下面的内容不需要打入Jar包，故请移除==

```xml
<resources>
	<resource>
		<directory>src/main/resources</directory>
			<excludes>
				<exclude>**/*.*</exclude>
			</excludes>
	</resource>
</resources>
```



#### 5 Dockerfile编写

##### 5.1 描述

Dockerfile用来把应用制作成能在Kubernetes上运行的内容（镜像），而这也是Dockerfile的精华，一次制作好镜像，任何容器环境通用（一个镜像就是一个精简的小型系统，包括该应用运行所需要的所有依赖，所以不管放到哪个容器环境，都能完美运行）。

##### 5.2 Dockerfile内容

Dockerfile参考如下

```dockerfile
#指定基础镜像
FROM registry.100credit.cn/centos6/jdk_7u80:v1.0
#Dockerfile使用变量（可以在Dockerfile中进行引用）
ARG APPNAME=api-gateway
ARG APP_HOME=/opt/SpringCloud
#工作目录
WORKDIR $APP_HOME
#指定appname标签
LABEL appname=$APPNAME \
#指定version标签
version=1.0.0
#把在宿主机上编译好的jar包复制到容器内的指定地址，格式为”COPY [宿主机目录] [容器内目录]“，注意宿主机目录一定是Dockerfile所在目录的相对目录
COPY target/api-gateway*.jar $APP_HOME/app/$APPNAME/lib/
#拷贝不同环境配置文件到config目录
COPY src/main/resources/ $APP_HOME/app/$APPNAME/config/
#拷贝启动文件到 工作目录
COPY cloudserver.sh $APP_HOME
#环境变量配置
#0为注册中心，1为非注册中心应用
ENV APP_TYPE=1 \
#HOME目录
APP_HOME=$APP_HOME \
#服务目录
SERVICE_HOME=$APP_HOME/app/$APPNAME \
#启动类
MAIN_CLASS=com.br.api.gateway.BrAPIApplication \
#application文件名
APP_CONFIG=application.yml \
#是否依赖公共jar, 1 依赖 2 不依赖 
DEPEND_COMMON=2 \
#启动参数
APP_PARAM="-Xmx2g -Xms2g -XX:NewRatio=4 -XX:PermSize=128m -Xss256k -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled"
#指定启动容器时执行的命令
CMD ["sh", "cloudserver.sh", "start"]
```

##### 5.3 Dockerfile位置

由于Jenkins打镜像默认找文件根目录，所以需要把写好的Dockerfile放入pom同级的目录中。

##### 5.4 注意

- ==Dockerfile每执行一行，代表一层，建议尽量减少层数。==
- ==LABEL和ENV可以一层定义多个（一行），所以编写时，使用"\\"作为一行的连接标识。==



#### 6 容器内微服务启动脚本拷贝

##### 6.1 描述

拷贝cloudserver.sh到应用根目录（pom）同级。

##### 6.2 原由

和SpringCloud一样，容器内的微服务应用也需要启动，只不过启动脚本经过物理机运行脚本改写，以适用Kubernetes。

##### 6.3 启动脚本内容

cloudserver.sh

```shell
#!/bin/bash
echo $*

#修复centos dns解析慢问题（ipV6超时）
echo 'options single-request-reopen' >> /etc/resolv.conf

APP_TYPE=${APP_TYPE}
[ ! -z "$APP_TYPE" ] || { echo "APP_TYPE 不能为空，请配置APP_TYPE，0为注册中心，1为非注册中心应用。";
        if [ "$1" = "stop" ]; then exit 0;
        else exit 1; fi; }

CLOUDSERVER_HOME=${APP_HOME}
[ -d "$CLOUDSERVER_HOME" ] || { echo "CLOUDSERVER_HOME 路径不存在: $CLOUDSERVER_HOME";
        if [ "$1" = "stop" ]; then exit 0;
        else exit 1; fi; }
        
CONF_ENV=${CONF_ENV}
[ ! -z "$CONF_ENV" ] || { echo "环境变量 CONF_ENV 为空或者配置错误！";
                if [ "$1" = "stop" ]; then exit 0;
        else exit 1; fi; }
        
LOG_LEVEL=${LOG_LEVEL}
[ ! -z "$LOG_LEVEL" ] || { echo "环境变量 LOG_LEVEL 为空或者配置错误！";
               if [ "$1" = "stop" ]; then exit 0;
        else exit 1; fi; }

POD_NAME=${POD_NAME}
[ ! -z "$POD_NAME" ] || { echo "环境变量 POD_NAME 为空或者配置错误！";
               if [ "$1" = "stop" ]; then exit 0;
        else exit 1; fi; }
        
APP_CONFIG=${APP_CONFIG}
[ ! -z "$APP_CONFIG" ] || { echo " APP_CONFIG 为空或者配置错误！${APP_CONFIG}";
               if [ "$1" = "stop" ]; then exit 0;
        else exit 1; fi; }

SERVICE_HOME=${SERVICE_HOME}
[ ! -z "$$SERVICE_HOME" ] || { echo "SERVICE_HOME 该文件不存在或者没有权限: $SERVICE_HOME";
        if [ "$1" = "stop" ]; then exit 0;
        else exit 5; fi; }

NAME=${APPNAME}
export NAME
[ ! -z "$NAME" ] || { echo "环境变量 APPNAME 为空或者配置错误！";
        if [ "$1" = "stop" ]; then exit 0;
       else exit 5; fi; }

PORT=${SERVER_PORT}

[ ! -z "$PORT" ] || { echo "环境变量SPRING_APPLICATION_PORT为空或者配置错误！";
       if [ "$1" = "stop" ]; then exit 0;
       else exit 5; fi; }

URL=`echo ${EUREKA_CLIENT_SERVICE_URL_defaultZone} | awk -F'[ ,]+' {'print $1'}`
[ ! -z "$URL" ] || { echo "环境变量EUREKA_CLIENT_SERVICE_URL_defaultZone为空或者配置错误！";
                if [ "$1" = "stop" ]; then exit 0;
        else exit 5; fi; }

NAME_UPPER="$(echo $NAME| tr '[:lower:]' '[:upper:]')" 

HOSTNAME=`hostname`

[ ! -z "$HOSTNAME" ] || { echo "hostname 配置错误，请检查/etc/hosts 文件配置。";
        if [ "$1" = "stop" ]; then exit 0;
        else exit 5; fi; }

APPNAME="${HOSTNAME}:${NAME}:${PORT}" 

CLOUDSERVER_PID_FILE="$CLOUDSERVER_HOME/pid"

CLOUDSERVER_LOG_FILE="$CLOUDSERVER_HOME/logs/$NAME/$POD_NAME/cloudserver.log"

CLOUDSERVER_JAVA_CMD="$JAVA_HOME/bin/java" 

JAVA_OPTIONS="${APP_PARAM}" 

MAIN_CLASS=${MAIN_CLASS}

DEPEND_COMMON=${DEPEND_COMMON}

echo $MAIN_CLASS

JAVA_CMD="$CLOUDSERVER_JAVA_CMD $JAVA_OPTIONS  -cp $SERVICE_HOME/lib/*:$SERVICE_HOME/config/$CONF_ENV $MAIN_CLASS "

if [[ $DEPEND_COMMON == '2' ]] ; then
    echo "依赖标示：$DEPEND_COMMON" 
    JAVA_CMD="$CLOUDSERVER_JAVA_CMD $JAVA_OPTIONS  -cp $SERVICE_HOME/lib/*:$SERVICE_HOME/config/$CONF_ENV  $MAIN_CLASS " 
else
    echo "依赖标示：$DEPEND_COMMON"
fi

if [[ $SERVICE_HOME == '/opt/SpringCloud/app/RULE-SERVICE' ]] ; then
    echo "规则服务" 
    JAVA_CMD="$CLOUDSERVER_JAVA_CMD $JAVA_OPTIONS  -cp $SERVICE_HOME/lib/*:$SERVICE_HOME/config/$CONF_ENV  $MAIN_CLASS $SERVICE_HOME/config/$CONF_ENV/parseJs.js " 
fi

CONFIG_PATH=${APP_CONFIG}
PARAMS="--spring.config.location=${SERVICE_HOME}/config/${CONF_ENV}/ --server.tomcat.max-threads=1000" 

RETVAL=0

#检查端口是否被监听方法
function checkport(){
    PID=$1
    num=`netstat -ntpl | grep $PID/ | wc -l`
    print_comm="netstat -ntpl 2>&1 | grep $PID/ " 
    echo "检查端口监听状态，请稍等!" 
    while [[ $num -le 0 ]];
    do
        echo -ne "." 
        sleep 1
        num=`netstat -ntpl | grep $PID/ | wc -l`
        if [ $num -gt 0 ];then
            echo "" 
            echo "端口监听信息如下：" 
            printline= eval "$print_comm" 
            echo $printline
        fi        
    done 

}

#查询注册中心状态
function getstatus() {
  APPSTATUS=`curl -XGET ${URL}apps -s | egrep '<instanceId>|<status>' |sed '1,$s/\s\+<instanceId>\|\s\+<status>\|<\/status>//g' |sed 'N;s/<\/instanceId>\n/ /g' | grep ${APPNAME} |awk {'print $2'}`
  echo ${APPSTATUS}
}

#注册中心解除注册
function stopapp() {
  STATUS=`getstatus`
  if [[ $STATUS == 'UP' ]] ; then
    echo -e "应用[${NAME}]状态为[${STATUS}],现在通知注册中心停止该服务！\n" >> "$CLOUDSERVER_LOG_FILE"
    curl -XPUT -s  "${URL}apps/${NAME_UPPER}/${APPNAME}/status?value=OUT_OF_SERVICE" 
    while [[ `getstatus` != 'OUT_OF_SERVICE' ]];
    do
      echo -e  $(date) "停止服务循环获取该服务注册状态\n" >> "$CLOUDSERVER_LOG_FILE"
      echo -ne "." 
      sleep 1
    done
    echo -e "应用[${NAME}]通知注册中心停止服务成功！\n" >> "$CLOUDSERVER_LOG_FILE"
    echo "success"
  elif [[ $STATUS == 'OUT_OF_SERVICE' ]]; then
    echo -e "应用[${NAME}]状态为[${STATUS}],该应用已解除注册，请勿重复操作！\n"  >> "$CLOUDSERVER_LOG_FILE"
  else
    echo -e "应用[${NAME}]状态为[${STATUS}], 状态码无效!\n"  >> "$CLOUDSERVER_LOG_FILE"
    echo "failure" 
  fi
}

#注册中心注册
function startapp() {
  STATUS=`getstatus`
  if [[ $STATUS == 'OUT_OF_SERVICE' || $STATUS == '' ]] ; then
    echo -e "应用[${NAME}]状态为[${STATUS}],现在通知注册中心启动该服务！\n"  >> "$CLOUDSERVER_LOG_FILE"
    curl -XPUT -s "${URL}apps/${NAME_UPPER}/${APPNAME}/status?value=UP" 
    while [[ `getstatus` != 'UP' ]];
    do
      echo -e  $(date) "启动服务循环获取服务注册状态\n" >> "$CLOUDSERVER_LOG_FILE"
      echo -ne "."
      sleep 2
    done
    echo -e "应用[${NAME}]通知注册中心启动服务成功！\n"  >> "$CLOUDSERVER_LOG_FILE"
    echo "success"
  elif [[ $STATUS == 'UP' ]]; then
    echo -e "应用[${NAME}]状态为[${STATUS}],该应用已启动完毕！\n" >> "$CLOUDSERVER_LOG_FILE"
  else 
    echo -e "应用[${NAME}]状态为[${STATUS}], 状态码无效!\n" >> "$CLOUDSERVER_LOG_FILE"
    echo "failure" 
  fi
}

function register(){
    sleep 5
    checkport $PID
    echo -e $(date) "服务注册开始\n" >> "$CLOUDSERVER_LOG_FILE"
    if [ $APP_TYPE != 0 ]; then 
        sleep 5
        startapp
    fi
    echo -e $(date) "服务注册成功\n" >> "$CLOUDSERVER_LOG_FILE"
    echo "success" 
}

#启动服务方法
function start() {
    START_COMM="$JAVA_CMD $PARAMS &" 
    echo "执行启动命令:[$START_COMM]"
    echo -e $(date) "服务启动开始\n" >> "$CLOUDSERVER_LOG_FILE"
    eval "$START_COMM" 
    RETVAL=$?
    if [ $RETVAL = 0 ]; then
        PID=$!
        echo $PID > "$CLOUDSERVER_PID_FILE"
        echo "执行启动命令成功！" 
        register &
        wait $PID
    else
        echo "failure" 
    fi
}

#停止服务方法
function stop() {
    echo -e $(date) "服务停止开始\n" >> "$CLOUDSERVER_LOG_FILE"
    if [ $APP_TYPE != 0 ]; then
        stopapp
        echo -e $(date) "服务停止成功\n" >> "$CLOUDSERVER_LOG_FILE"
        sleep 10
        echo -e  "进程睡眠10秒\n" >> "$CLOUDSERVER_LOG_FILE"
        cd /opt/SpringCloud/logs/${NAME}/  &&  mv ${POD_NAME} ${POD_NAME}_$(date +%Y%m%d)
    fi
    kill -9 `cat "$CLOUDSERVER_PID_FILE"`

}

case $1 in
    start)
        start
        ;;
    stop)
        stop
        ;;
    restart)
        stop
        start
        ;;        
    *)
        echo "Usage: $0 {start|stop|status|try-restart|restart|force-reload|reload|probe}" 
        exit 1
        ;;

esac

exit $RETVAL
```



#### 7 deployment.yaml编写

##### 7.1 描述

编写应用deployment.yaml，放入marmot，由marmot使用。

##### 7.2 原由

Deployment描述应用（容器）如何在Kubernetes上进行调度、执行、容灾等一序列问题（编排），可以说Deployment是容器的命脉，也是Kubernetes的优势体现。

Marmot通过调用Kubernetes API，传给Kubernetes deployment远程启动应用。

##### 7.3 内容

deployment.yaml参考

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: api-gateway  #Deployment name
  namespace: default #Deployment所在namespace
  labels:            #此Deployment选择拥有api-gateway标签的containers
    app: api-gateway   
    ver: 190115-1        #应用版本
spec:
  replicas: 2         #应用副本数（该应用具体实例数量）
  revisionHistoryLimit: 10        #保留的历史版本，用作回滚（默认值，可根据实际情况修改）
  minReadySeconds: 5        #多少秒之后启动容器（默认值，可根据实际情况修改）
  strategy:        #升级策略（默认为滚动升级，不需要修改）
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        #滚动升级中，容器副本的最大数量（默认值，可根据实际情况修改）
      maxUnavailable: 25%        #滚动升级中，容器副本停止的最大数量（默认值，可根据实际情况修改）
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
        affinityLabel: 190115-1
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - api-gateway
                - key: affinityLabel
                  operator: In
                  values:
                  - 190115-1
              topologyKey: kubernetes.io/hostname
#      hostAliases:        #配置容器的Host（可选参数）
#      - ip: "192.168.23.78" 
#        hostnames:
#        - "speed-api.100credit.cn" 
      containers:
      - name: api-gateway
        image: registry.100credit.cn/spring_cloud/api-gateway:1.0.0 #镜像仓库地址
        imagePullPolicy: Always
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "/opt/SpringCloud/cloudserver.sh", "stop"]
        ports:
        - containerPort: 80
        env:  #环境变量
        - name: EUREKA_CLIENT_SERVICE_URL_defaultZone   #eureka地址
          value: "http://eureka-0/eureka/,http://eureka-1/eureka/,http://eureka-2/eureka/" 
        - name: SERVER_PORT    #微服务服务端口
          value: "80" 
        - name: CONF_ENV       #微服务配置目录环境变量
          value: "pre" 
        - name: LOG_LEVEL      #日志级别
          value: "DEBUG" 
        - name: APPNAME        #微服务名
          value: "api-gateway" 
        - name: POD_NAME       #PODNAME，用来区分日志目录
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        volumeMounts:
        - mountPath: "/opt/SpringCloud/logs" 
          name: api-gateway
        livenessProbe:        #容器健康检查（可选参数）
          httpGet:
            path: /health    #微服务默认有health接口，使用此接口即可
            port: 80        #与容器端口保持一致
            scheme: HTTP
          initialDelaySeconds: 30  #健康检查初始化延时
          timeoutSeconds: 2        #每次检查超时时间
          periodSeconds: 10        #检查周期
          failureThreshold: 3        #最少连续探测失败多少次才被认定为失败
        readinessProbe:        #容器启动检查（可选参数）
          httpGet:
            path: /health
            port: 80        #与容器端口保持一致
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 2
          periodSeconds: 10
          failureThreshold: 3        #最少连续探测失败多少次才被认定为失败
        resources:
          requests: 
            memory: 512Mi        #内存最小使用量（默认值，可根据实际情况修改）
            cpu: "0.2"        #cpu最小使用率（默认值，可根据实际情况修改）
          limits:
            memory: 4Gi        #内存最大使用量（默认值，可根据实际情况修改）
            cpu: "2"        #cpu最大使用率（默认值，可根据实际情况修改）
      volumes:
        - name: api-gateway
          persistentVolumeClaim:
            claimName: glusterfs-volume-logs

```

##### 7.4 注意

EUREKA_CLIENT_SERVICE_URL_defaultZone变量的==eureka地址必须以"/"结尾==：http://eureka-0/eureka/

eureka需要配置host访问（通过ingress映射）

```tex
#预发
192.168.23.88 eureka-0.100credit.cn eureka-1.100credit.cn eureka-2.100credit.cn openapi2.100credit.cn
#线上
192.168.22.135 eureka-0.100credit.cn eureka-1.100credit.cn eureka-2.100credit.cn openapi2.100credit.cn
```



## 四、Jenkins和marmot流程请参考redmine中其他wiki页面