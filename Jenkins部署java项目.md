#### Jenkins部署java项目

参考文章：http://blog.51cto.com/zero01/2074341

##### 功能

- 持续的软件版本发布/测试项目；
- 监控外部调用执行的工作；

##### 环境依赖

- JDK1.8
- apache-tomcat-9.0.5
- apache-maven-3.0.5
- git

##### 准备工作

- 准备私有git仓库及测试项目、配置仓库密钥
- 准备tomcat安装包

##### 安装配置

JDK、tomcat、maven、git安装略……

- tomcat配置
  - 配置tomcat用户

      ```shell
      编辑 {TOMCAT_HOME}/conf/tomcat-users.xml
      # 在文件末尾加入以下内容
      <role rolename="admin"/>  # role配置角色
      <role rolename="admin-gui"/>
      <role rolename="admin-script"/>
      <role rolename="manager"/>
      <role rolename="manager-gui"/>
      <role rolename="manager-script"/>
      <role rolename="manager-jmx"/>
      <role rolename="manager-status"/> 
      <user name="admin" password="admin" roles="admin,manager,admin-gui,admin-script,manager-gui,manager-script,manager-jmx,manager-status" />
      ```

  - 配置tomcat的context.xml

      只需要配置白名单ip即可，默认只允许本地ip访问： 

      ```shell
      编辑 {TOMCAT_HOME}/webapps/manager/META-INF/context.xml
      # 修改如下
      <Context antiResourceLocking="false" privileged="true" >
        <Valve className="org.apache.catalina.valves.RemoteAddrValve"
      allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1|192.168.77.*" /># 这里可以根据你自己的机器ip进行配置
        <Manager sessionAttributeValueClassNameFilter="java\.lang\.(?:Boolean|Integer|Long|Number|String)|org\.apache\.catalina\.filters\.CsrfPreventionFilter\$LruCache(?:\$1)?|java\.util\.(?:Linked)?HashMap"/>
      </Context>
      ```

  		打开浏览器进入到Tomcat的web页面，然后点击 “manager webapp” 进入到管理页面： 
  		点击manager webapp即可进入Tomcat Web Application Manager

- Jenkins安装

  ```
  解压 jenkins-2.156.zip
  ```

  - 设置环境变量

    ```shell
    # Windows略
    # Linux在启动命令里添加指定位置
    export JENKINS_HOME =/usr/local/Jenkins
    ```

  - Jenkins主目录

    默认主目录位置为~/.jenkins 

    修改方法：

    - 启动 Servlet 容器之前，设置“JENKINS_HOME”环境变量设置为新的主目录 

    - 设置 “JENKINS_HOME” 系统属性 到 servlet 容器。 

    - 设置JNDI环境条目“JENKINS_HOME”到新目录。 

      

  - 启动

    通过内置的Jetty的servlet容器启动：

    进入Jenkins家目录执行

    ```shell
    $ java -jar jenkins.war --httpPort=8180	#这里避免和tomcat默认端口冲突
    ```

    常用命令参数：

    | 命令参数                            | 说明                                                 |
    | ----------------------------------- | ---------------------------------------------------- |
    | --httpPort = $ HTTP_PORT            | jenkins监听*HTTP端口*。默认端口号为8080。            |
    | --httpListenAddress = $ HTTP_HOST   | 监听IP地址， 默认值是0.0.0.0 -即侦听所有可用的接口。 |
    | --httpsPort = $ HTTP_PORT           | 使用HTTPS协议的端口$ HTTP_PORT                       |
    | --httpsListenAddress = $ HTTPS_HOST | 使用HTTPS协议的IP                                    |
    | --prefix = $ PREFIX | 访问jenkins的url后缀。如`http://localhost:8180/jenkins`                                    |
    |-XX：PermSize = 512M -XX：MaxPermSize = 2048M -Xmn128M -Xms1024m -Xmx2048M | 指定JVM启动参数 |

   访问：`http://localhost:8180/`，创建用户密码后进入主页面

  - 插件安装

    进入“管理插件”模块安装必需的插件 。本次部署，必要的插件：

    `Deploy to container Plugin`
    `Git client`
    `Git plugin`
    `Maven Integration`
    安装完成后重启jenkins

- Jenkins新建任务

  首页点击新建任务，选择`构建一个maven项目`，输入项目名。

  点击源码管理，选择`Git`

  - 添加Credentials 

    此处需要配置密钥，即git仓库的私钥。创建git ssh密钥简单步骤如下：

    ```shell
    # 进入本地仓库项目目录，或打开Git Bash
    ssh-keygen -t rsa -C "grezz07@gmail.com"   # "grezz07@gmail.com"为远程git仓库账号
    # 执行后会在C:\Users\admin\.ssh路径下会生成两个文件：id_rsa和id_rsa.pub
    若远程仓库为github，则进入个人设置页面，添加`SSH and GPG keys`,复制id_rsa.pub里面的内容，新建。
    ```

  - 配置远程git仓库

    `Repository URL `添加你的git项目地址；

    `Credentials `添加远程仓库密钥，点击add，选择使用私钥。`username`默认为git用户。将私钥内容粘贴至文本框，选择添加的密钥。

  - 构建触发器、构建环境、Pre Steps这几项这里保持默认 ；

  - 配置`Build`

    `Root POM `填入：pom.xml

    `Goals and options `填入：clean install -D maven.test.skip=true

  - `Post Steps `和`构建设置 `也保持默认

  - 邮件通知暂时没配置成功，提示`javax.mail.MessagingException: Could not connect to SMTP host: localhost, port: 25`

- Jenkins发布war包

  进入该任务的配置页面，选择构建后操作一栏；

  点击增加构建后操作步骤，选择 “Deploy war/ear to a container” ；

  `WAR/EAR files `输入`**/*.war`，表示匹配所有war(若部署为jar包则相应修改即可)；

  `Containers`选择Tomcat 8.x

  `Tomcat URL`填写要把war包发布到的那台机器的url,此处：`http://localhost:8080`

  `Credentials `添加tomcat的管理用户密码

  最后保存，即可立即构建任务