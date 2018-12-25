# dockerbuild-gitlab-runner
gitlabRunner+maven+java+docker+npm镜像构建

构建脚本详情：[Docker-Gitlab-Runner](http://blog.iexxk.com/2018/07/31/Docker-Gitlab-Runner/)

### 环境支持

* maven:3.6.0
* java8:181.13-r0
* docker:18.09.0
* nodejs-npm
* shadow(权限修改)

### 自动构建镜像支持：java编译打包、maven编译打包、前端vue编译打包

### 使用

1. 注册方式和官方runner一样，命令执行模式选择shell就可以，其他默认就行
2. 测试，进入容器，然后执行docker命令，如果没有权限，执行`chown :root /var/run/docker.sock`

##### java/maven编译打包

`.gitlab-ci.yml`

```yaml
variables:
  #当前项目的.m2文件的maven私库配置文件setting.xml
  MAVEN_CLI_OPTS: "-s .m2/settings.xml --batch-mode"
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"
stages:
  - build
  - deploy

cache:
  paths:
    - .m2/repository/
    - target/

build:
  stage: build
  only:
    - manage-test-deploy
  script:
  # 登陆docker镜像仓库，单节点可以不要，这里用的明码，加密可以去配置变量
    - docker login 192.168.1.22:14005 -u admin -p xxx
  # 通过maven打包java项目  
    - mvn $MAVEN_CLI_OPTS clean install -DskipTests=true
  # 通过项目根目录下的dockerfile进行编译，镜像名字自定义
    - docker build -t 192.168.1.22:14005/manage/test/service:latest .
  # 有仓库推送到仓库，单节点可以不要  
    - docker push 192.168.1.22:14005/manage/test/service:latest
deploy:
  stage: deploy
  only:
    - manage-test-deploy
  script:
   #- docker stack deploy -c manager-test-ygl.yml manager-test-ygl
   # --force 强制更新
   # 先要启动集群部署服务，然后更新部署服务，也可以写个脚本判断第一次就创建部署，后面就更新
   - docker service update --image 192.168.1.22:14005/manage/test/service:latest --with-registry-auth --force manager-test-service
```

`dockerfile`举例：

```dockerfile
#基础镜像选择alpine 小巧安全流行方便
FROM www.3sreform.com:14005/tomcat:8-alpine-cst
#复制固定路径下打包好的jar包(target/*.jar)并重命名到容器跟目录(/app.jar)，或ADD
COPY target/app.war /usr/local/tomcat/webapps/
CMD ["catalina.sh", "run"]
```

`dockerstack.yml`举例

```yaml
# name: manager-test
version: '3.2'

services:
  service:
    restart: always
    image: 192.168.1.22:14005/manage/test/service:latest
    volumes:
      - /logs/service:/app/log
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure
      placement:
        constraints: [node.hostname == environment-test1]
```

##### nginx编译打包

`.gitlab-ci.yml`

```yaml
cache:
  untracked: true
  paths:
    - node_modules/
    
stages:
  - build

build:
  stage: build
  only:
  #限定分支为deploy才部署编译
    - deploy
  script:
  # 淘宝镜像
    - cnpm i
    - npm run build 
    - docker login 192.168.1.22:14005 -u admin -p XXX
    - docker build -t 192.168.1.22:14005/nginx:latest .
    - docker push 192.168.1.22:14005/nginx:latest

```

`dockerfile`文件举例

```dockerfile
FROM nginx:alpine
RUN rm /etc/nginx/conf.d/default.conf
ADD default.conf /etc/nginx/conf.d/ 
COPY dist/  /usr/share/nginx/html/
```











