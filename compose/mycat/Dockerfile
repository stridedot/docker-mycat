# JRE(Java Runtime Enviroment) 是 Java 的运行环境。
# 面向 Java 程序的使用者，而不是开发者。
# 如果你仅下载并安装了 JRE，那么你的系统只能运行 Java 程序。
# JRE 是运行 Java 程序所必须环境的集合，包含 JVM 标准实现及 Java 核心类库。
# 它包括 Java 虚拟机、Java 平台核心类和支持文件。它不包含开发工具(编译器、调试器等)
FROM openjdk:8-jdk-stretch
MAINTAINER <tristin.zhao@outlook.com>
LABEL Description="使用 Mycat 做 MySQL 的读写分离"

# ARG MYCAT_VERSION=1.6.7.6

# Timezone and lib
# RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories \
#    && apk add -U tzdata \
#    && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
#    && apk del tzdata \
#    && apk add curl

# 在线安装 Mycat
#RUN set -ex \
#    && cd /tmp \
#    && curl -SL "http://dl.mycat.org.cn/${MYCAT_VERSION}/20211118155357/Mycat-server-${MYCAT_VERSION}-release-20211118155357-linux.tar.gz" -o Python.tar.gz \
#    && tar -zxf Python.tar.gz -C /usr/local/ \
#    && rm -f /tmp/Python.tar.gz \
#    && chmod u+x /usr/local/Python \
#    && chmod 755 /usr/local/Python/bin

ADD http://dl.mycat.org.cn/1.6.7.6/20211118155357/Mycat-server-1.6.7.6-release-20211118155357-linux.tar.gz /usr/local
RUN cd /usr/local && tar -zxvf Mycat-server-1.6.7.6-release-20211118155357-linux.tar.gz && ls -lna

ENV MYCAT_HOME=/usr/local/mycat
WORKDIR /usr/local/mycat

ENV TZ Asia/Shanghai

EXPOSE 8066 9066

CMD ["/usr/local/mycat/bin/mycat", "console","&"]