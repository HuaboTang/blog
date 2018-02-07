# 背景

最近使用SpringBoot和Docker，给公司的广告平台在阿里云上部署了弹性伸缩服务。这里记录下过程和配置。

阿里弹性扩容目前是基于ECS镜像来启动新的服务器。单纯使用镜像的问题是：当应用升级时，为了运行最新的服务，需要创建新的镜像，并修改弹性扩容的配置以调用新的镜像。费时费力。可替换的方案有两个：

- 在服务器上拉取代码，打包，再运行服务
- 使用Docker镜像，拉取最新的镜像并运行

第一种方案，不确定因素更多，如：pull代码的账号密码、打包依赖的版本等。选择第二种方案实施。

# 一些条件和限定
- 项目是Java写的，并使用了SpringBoot进行开发和部署
- 部署弹性伸缩的模块是处理http请求的
- Maven管理Jar依赖
- 以下内容将只描述在开发层面的处理，诸如阿里云弹性伸缩配置将不进行介绍

# 开始

## 镜像启动逻辑

1. 镜像安装Docker，设置开机启动
2. 使用systemctl，开机执行脚本，拉取最新镜像，并启动镜像容器


## SpringBoot增加Docker打包和发布

### pom.xml

打包和发布使用spotify提供的dockerfile插件，配置如下：

```xml
<plugin>
    <groupId>com.spotify</groupId>
    <artifactId>dockerfile-maven-plugin</artifactId>
    <version>1.3.6</version>
    <configuration>
        # 打包工作目录
        <contextDirectory>${project.build.directory}</contextDirectory>
        # Dockerfile中的参数
        <buildArgs>
            <JAR_FILE>${project.build.finalName}.jar</JAR_FILE>
        </buildArgs>
        # image版本
        <tag>latest</tag>
        # 使用maven setting来配置用户信息
        <useMavenSettingsForAuth>true</useMavenSettingsForAuth>
    </configuration>
</plugin>
```

### maven setting.xml

在maven的setting.xml中配置镜像仓库的相关信息：

```xml
<servers>
    ...
    <server>
        <id>阿里docker镜像仓库地址</id>
        <username>用户名</username>
        <password>密码</password>
        <configuration>
            <email>邮箱</email>
        </configuration>
    </server>
</servers>
```

### Dockerfile
```
FROM java:8
VOLUME ["/tmp", "/home/www/server/logs"]
ARG JAR_FILE
ADD ${JAR_FILE} @project.build.finalName@.jar

RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
  && echo 'Asia/Shanghai' >/etc/timezone \
ENTRYPOINT exec java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar /@project.build.finalName@.jar
```

### 打包命令

项目目录下执行：
```shell
mvn clean install -Dmaven.test.skip \
&& \ 
mvn -f xx/pom.xml install dockerfile:build dockerfile:push \
-Dmaven.test.skip
```

##  服务器初始化脚本

### 启动shell脚本
```shell
#!/bin/sh
##########################################
# $Name:        start_docker.sh
# $Version:     v1.0
# $Function:
# $Author:      vbobot
# $page:        http://blog.vbobot.com
# $Create Date: 2018-01-29
# $Desc:        启动server docker
##########################################

# Usage Function
usage() {
    echo "Usage: $0 do_handle"
}

do_handle() {
    # 使用正确的用户名和密码替换(username} 和 {password}    
    docker login --username={username} -p {password} registry.ap-southeast-1.aliyuncs.com
    docker pull {镜像地址}
    docker kill server
    docker rm server
    docker run -e "SPRING_PROFILES_ACTIVE=pro" \
        -e "USER=www" \
        -e "JAVA_OPTS=-Xms5g -Xmx5g -Xmn3g -XX:+UseConcMarkSweepGC -XX:+PrintGCDetails -Xloggc:/home/www/product/adt-server-data/gc.log" \
        -p 8080:8080 -t --name server \
        -v /home/www/server/logs:/home/www/server/logs \
        -v /tmp:/tmp \
        registry.ap-southeast-1.aliyuncs.com/codrim_cs/codrim-adt-server-data:latest
}

# Main Function
main() {
    case $1 in
        do_handle)
            do_handle
            ;;
        *)
            usage
            ;;
    esac
}

# Exec
if [ $# != 1 ] ; then
	usage
	exit 1
fi
main $1
```

### systemctl配置
使用systemctl配置开机启动任务，执行上面的shell。位置：/etc/systemd/system/xxxx.service
```
[Unit]
Description=Codrim adt server data docker
# 在docker启动之后执行
After=docker.service

[Service]
Type=forking
User=root
ExecStart=/bin/bash /root/start_docker.sh do_handle

[Install]
WantedBy=multi-user.target
```

# 参考资料


