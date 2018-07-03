---
title: Jenkins+Gitlab持续集成
date: 2018-05-11 16:42:28
tags: [Jekins, Gitlab]
---

## Install Jenkins

### Java Environment

下载jdk

```sh
wget http://download.oracle.com/otn-pub/java/jdk/8u171-b11/512cd62ec5174c3487ac17c61aaa89e8/jdk-8u171-linux-x64.tar.gz?AuthParam=1526090409_eeb74c2f68fe8539ce3cb2608695d850;

tar xvf jdk-8u171-linux-x64.tar.gz -C /opt/;
mv /opt/jdk1.8.0_171/ /opt/jdk/;
```

配置环境变量

编辑/etc/profile

```sh
JAVA_HOME=/opt/jdk
CLASSPATH=$JAVA_HOME/lib/
PATH=$PATH:$JAVA_HOME/bin
export PATH JAVA_HOME CLASSPATH
```

运行环境

```sh
source /etc/profile
```

验证环境

```sh
kdreader@kdreader-staging:~$ java -version
java version "1.8.0_171"
Java(TM) SE Runtime Environment (build 1.8.0_171-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.171-b11, mixed mode)
```

### Install jenkins

```sh
wget https://prodjenkinsreleases.blob.core.windows.net/debian/jenkins_2.121_all.deb; sudo  dpkg -i jenkins_2.121_all.deb;
```

更新`/etc/init.d/jenkins`PATH变量

```
PATH=/bin:/usr/bin:/sbin:/usr/sbin:/opt/jdk/bin/
```

启动 Jenkins

`sudo /etc/init.d/jenkins start`

配置nginx

添加`/etc/nginx/sites-enabled/jenkins.conf`

```nginx
server {
    listen 80;
    server_name  ci.kdreader.com;


    server_name_in_redirect on;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $remote_addr;
    proxy_set_header User-Agent $http_user_agent;
    proxy_set_header Referer $http_referer;
    proxy_connect_timeout      300;
    proxy_send_timeout         300;

    uwsgi_read_timeout 360;
    uwsgi_send_timeout 360;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

重启nginx

`sudo nginx -s reload`

## Config Jenkins

`http://ci.kdreader.com`访问Jenkins

初始化秘钥路径：`/var/lib/jenkins/secrets/initialAdminPassword`

### Install and Config Gitlab plugin

配置Credential

Gitlab生成token

![gitlab-generate-token](https://share-static.oss-cn-hangzhou.aliyuncs.com/mengyangyang.org/pictures/gitlab-generate-token.png)

配置Credentials

![jenkins-config-credential](https://share-static.oss-cn-hangzhou.aliyuncs.com/mengyangyang.org/pictures/jenkins-config-credential.png)

配置Gitlab

![jenkins-config-gitlab](https://share-static.oss-cn-hangzhou.aliyuncs.com/mengyangyang.org/pictures/jenkins-config-gitlab.png)

### Install and Config SSH Remote Hosts

配置Credentials

![jenkins-config-ssh-credential](https://share-static.oss-cn-hangzhou.aliyuncs.com/mengyangyang.org/pictures/jenkins-config-ssh-credential.png)

配置SSH remote hosts

![jenkins-config-ssh-remote-hosts](https://share-static.oss-cn-hangzhou.aliyuncs.com/mengyangyang.org/pictures/jenkins-config-ssh-remote-hosts.png)

### Config Email Notification

![jenkins-config-email-notification](https://share-static.oss-cn-hangzhou.aliyuncs.com/mengyangyang.org/pictures/jenkins-config-email-notification.png)

## Create Project

![jenkins-create-project](https://share-static.oss-cn-hangzhou.aliyuncs.com/mengyangyang.org/pictures/jenkins-create-project.png)

### Buil Trigger

![jenkins-project-trigger](https://share-static.oss-cn-hangzhou.aliyuncs.com/mengyangyang.org/pictures/jenkins-project-trigger.png)

配置Gitlab webhook

![gitlab-webhook](https://share-static.oss-cn-hangzhou.aliyuncs.com/mengyangyang.org/pictures/gitlab-webhook.png)

配置安全策略

![jenkins-security-config](https://share-static.oss-cn-hangzhou.aliyuncs.com/mengyangyang.org/pictures/jenkins-security-config.png)

### Build

![Jenkins-project-build](https://share-static.oss-cn-hangzhou.aliyuncs.com/mengyangyang.org/pictures/Jenkins-project-build.png)

### After Build

![jenkins-project-after-build](https://share-static.oss-cn-hangzhou.aliyuncs.com/mengyangyang.org/pictures/jenkins-project-after-build.png)

## Q&A

Question: Gitlab Webhook 403

```html
<html>
<head>
<meta http-equiv="Content-Type" content="text/html;charset=utf-8"/>
<title>Error 403 anonymous is missing the Job/Build permission</title>
</head>
<body><h2>HTTP ERROR 403</h2>
<p>Problem accessing /project/kdreader. Reason:
<pre>    anonymous is missing the Job/Build permission</pre></p><hr><a href="http://eclipse.org/jetty">Powered by Jetty:// 9.4.z-SNAPSHOT</a><hr/>

</body>
</html>
```

Solution:

配置Jenkins 匿名用户安全策略

![jenkins-security-config](https://share-static.oss-cn-hangzhou.aliyuncs.com/mengyangyang.org/pictures/jenkins-security-config.png)

## 参考

- [java jdk8](http://download.oracle.com/otn-pub/java/jdk/8u171-b11/512cd62ec5174c3487ac17c61aaa89e8/jdk-8u171-linux-x64.tar.gz?AuthParam=1526090409_eeb74c2f68fe8539ce3cb2608695d850)
- [jenkins debian package](https://pkg.jenkins.io/debian/)
- [配置Jenkins+Gitlab