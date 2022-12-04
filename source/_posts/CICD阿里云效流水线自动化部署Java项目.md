---
title: CI/CD 阿里云效流水线自动化部署Java项目
date: 2022/11/6 10:25
updated: 2022/11/6 10:25
tags: CI/CD
categories: 运维
sticky: 1
cover: https://zangzang.oss-cn-beijing.aliyuncs.com/img/002838-1667579318a9fc.jpg
---
### CI/CD 阿里云效流水线自动化部署Java项目

## 前提

>1. 需要拥有自己的云服务器(不一定是阿里的)
>2. 项目代码已经提交到代码仓库，如gitee,github等

## 上传代码仓库到云效

### 1、开通云效

[云效官网](https://www.aliyun.com/product/yunxiao)

![image-20221106094639040](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221106094639040.png)

点击免费使用

###  2、上传代码

点击代码管理

![image-20221106094746224](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221106094746224.png)

然后右上角有一个添加库点击之后选择导入代码库。之后选择你要导入的方式，导入之后会看到这个界面

![image-20221106095020342](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221106095020342.png)

点击这样就完成了代码的上传

## 创建(配置)流水线

![image-20221106095122017](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221106095122017.png)

他会让你选择模板，一般情况下会先扫描然后测试

![image-20221106095240332](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221106095240332.png)

### 1、构建上传编辑

![image-20221106095855665](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221106095855665.png)

### 2、主机部署

![image-20221106100005070](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221106100005070.png)

一开始是这样的，我们要选择主机组

点击新建然后看到主机类型，因为我的服务器不是在阿里所以选择自建的，如果是阿里的自行探索，这里官方引导的不错

![image-20221106100048510](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221106100048510.png)

![image-20221106100218723](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221106100218723.png)

看官方已经说的很清楚了，我就不做过多的解释了，然后接下来部署配置和我写一样就可以了

![image-20221106100315025](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221106100315025.png)

### 3、部署脚本

这个东西其实每个人都不是很一样，因为他自己创建的我们之前写那个deploy.sh那个文件，文件换行符是dos格式的"\r\n"可能后边会报错，此时我们先自己去服务器下载一个dos2unix，下载命令如下

```linux
sudo yum install dos2unix
```

然后我们配置部署脚本之前都去生成deploy文件的目录进行一下格式转换

部署脚本如下

![image-20221106100757128](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221106100757128.png)

```shell
mkdir -p /home/admin/zverify-blog //创建文件夹
tar zxvf /home/admin/app/package.tgz -C /home/admin/zverify-blog/
cd /home/admin/zverify-blog 跳转到目录对文件字符进行格式转义
dos2unix deploy.sh
sh /home/admin/zverify-blog/deploy.sh restart 重新启动脚本

```

>注意上边脚本的目录除了zverify-blog与我不同，其余与我相同即可，需注意这个东西是文件打成jar包之后的名称，请`自行更换`

此时流水线就配置好啦，我们先保存。然后去代码仓库

## 项目配置

### 1、编辑脚本文件deploy.sh

**文件位置**：项目最外层,与pom文件同一层级

**文件内容**：

```sh
#!/bin/bash

#修改为jar包名称
APP_NAME=zverify-blog


PROG_NAME=$0
ACTION=$1
APP_START_TIMEOUT=20    # 等待应用启动的时间
APP_PORT=8080         # 应用端口
HEALTH_CHECK_URL=http://xxxxx:${APP_PORT}/ # 应用健康检查URL
HEALTH_CHECK_FILE_DIR=/home/admin/status   # 脚本会在这个目录下生成nginx-status文件
APP_HOME=/home/admin/${APP_NAME} # 从package.tgz中解压出来的jar包放到这个目录下
JAR_NAME=${APP_HOME}/target/${APP_NAME}.jar # jar包的名字
JAVA_OUT=${APP_HOME}/logs/start.log  #应用的启动日志

#创建出相关目录
mkdir -p ${HEALTH_CHECK_FILE_DIR}
mkdir -p ${APP_HOME}/logs
usage() {
    echo "Usage: $PROG_NAME {start|stop|restart}"
    exit 2
}

health_check() {
    exptime=0
    echo "checking ${HEALTH_CHECK_URL}"
    while true
        do
            status_code=`/usr/bin/curl -L -o /dev/null --connect-timeout 5 -s -w %{http_code}  ${HEALTH_CHECK_URL}`
            if [ "$?" != "0" ]; then
               echo -n -e "\rapplication not started"
            else
                echo "code is $status_code"
                if [ "$status_code" == "200" ];then
                    break
                fi
            fi
            sleep 1
            ((exptime++))

            echo -e "\rWait app to pass health check: $exptime..."

            if [ $exptime -gt ${APP_START_TIMEOUT} ]; then
                echo 'app start failed'
               exit 1
            fi
        done
    echo "check ${HEALTH_CHECK_URL} success"
}
start_application() {
    echo "starting java process"
    nohup java -jar ${JAR_NAME} > ${JAVA_OUT} 2>&1 &
    echo "started java process"
}

stop_application() {
   checkjavapid=`ps -ef | grep java | grep ${APP_NAME} | grep -v grep |grep -v 'deploy.sh'| awk '{print$2}'`

   if [[ ! $checkjavapid ]];then
      echo -e "\rno java process"
      return
   fi

   echo "stop java process"
   times=60
   for e in $(seq 60)
   do
        sleep 1
        COSTTIME=$(($times - $e ))
        checkjavapid=`ps -ef | grep java | grep ${APP_NAME} | grep -v grep |grep -v 'deploy.sh'| awk '{print$2}'`
        if [[ $checkjavapid ]];then
            kill -9 $checkjavapid
            echo -e  "\r        -- stopping java lasts `expr $COSTTIME` seconds."
        else
            echo -e "\rjava process has exited"
            break;
        fi
   done
   echo ""
}
start() {
    start_application
    health_check
}
stop() {
    stop_application
}
case "$ACTION" in
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
        usage
    ;;
esac
```

需要更改的地方有一下几个

![image-20221106101521862](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221106101521862.png)

这里说一下第三个更改的地方，健康检查url，他在部署结束的时候回先去看一下这个端口是否可以访问，可以访问说明部署成功了，如果访问不到说明部署失败了，建议放一个不需要权限就可以访问的接口

### 2、pom文件

![image-20221106101727132](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221106101727132.png)

圈起来的地方改成jar包名称

```xml
<build>
            <finalName>zverify-blog</finalName>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <configuration>
                        <excludes>
                            <exclude>
                                <groupId>org.projectlombok</groupId>
                                <artifactId>lombok</artifactId>
                            </exclude>
                        </excludes>
                    </configuration>
                </plugin>
            </plugins>
        </build>
```

下面配置好了之后进行保存，然后到流水线界面运行

![image-20221106101958566](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221106101958566.png)

看到一下界面说明运行成功了

## 自动化部署配置

千万一定要允许成功，并且自己测试可以访问接口在来进行自动化部署哦

我们来流水线编辑

![image-20221106102124586](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221106102124586.png)

配置触发

复制代码

![image-20221106102221309](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221106102221309.png)

来这里进行新建就好啦。随便改一下代码进行提交测试

![image-20221106102358963](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221106102358963.png)

这样就被触发了。


> 我的博客即将同步至腾讯云开发者社区，邀请大家一同入驻：https://cloud.tencent.com/developer/support-plan?invite_code=ocj4bhqfct36
