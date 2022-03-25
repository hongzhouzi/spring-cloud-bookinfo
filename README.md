# spring-cloud-bookinfo
Spring Cloud microservices demo coppied from Istio Bookinfo



整个微服务应用中包含了 5 个组件

1. productpage 是一个由 react 开发的前端组件
2. gateway 是一个由 spring-cloud-gateway 提供的 API 网关服务
3. details 是一个 spring-cloud 微服务，提供了书籍详情 API
4. reviews-v1 是一个 spring-cloud 微服务，提供了基础的书籍评论信息，review-v2 在 review-v1 的基础之上额外的提供了评分数据，依赖 ratings 服务
5. ratings 是一个 golang 开发的微服务组件

在 springcloud demo 中我们用到以上的组件有：productpage、details、reviews，另外网关使用定制化后可配置网关路由的网关镜像。



微服务上云，主要分为以下几个步骤：

1. 打包应用：将应用打包成可运行的应用包
2. 制作镜像：docker 根据 Dockerfile 打包成制定镜像
3. 推送镜像：将镜像推送到仓库，部署时到仓库中取镜像
4. 部署应用：将应用部署在 K8s 集群上

打包应用的过程也可写在制作镜像的 Dockerfile 中，以下示例就是采取的这种方式，下面就分为准备镜像、部署应用这两个大步骤操作。



## 准备镜像

前提：在本机上安装 docker ，并建好镜像仓库，以下示例用的镜像仓库使用的 [Docker Hub](https://hub.docker.com/) 镜像仓库。

### productpage

到项目的 productpage 目录下。

```sh
# 构建镜像
docker build -t hongzhouwei/productpage:v1 .
# 将镜像推送到镜像仓库
docker push hongzhouwei/productpage:v1
```



### details

到项目目录下，将项目目录下 Dockerfile 让里面 `ARG TARGET=deatils`。

```sh
# 构建镜像
docker build -t hongzhouwei/deatils:v1 .
# 将镜像推送到镜像仓库
docker push hongzhouwei/deatils:v1
```



### reviews

到项目目录下，将项目目录下 Dockerfile 让里面 `ARG TARGET=reviews`。

```sh
# 构建镜像
docker build -t hongzhouwei/reviews:v1 .
# 将镜像推送到镜像仓库
docker push hongzhouwei/reviews:v1
```



若没有准备镜像，可用以下镜像demo

```sh
hongzhouwei/productpage:latest
hongzhouwei/deatils:latest
hongzhouwei/reviews:latest
```



## 部署应用

### 在 Kubesphere 上启用 springcloud

**页面方式启用：**

使用 admin 账号登录后 -> 平台管理 -> 集群管理 -> CRD菜单栏 -> 编辑 ClusterConfiguration 中 ks-installer 将 spec.springcloud.enabled 设置为 true，也可在里面对部署的 nacos 参数自由配置，[参数说明](https://github.com/nacos-group/nacos-k8s/blob/master/helm/README.md#configuration)。



**命令行方式启用：**

```sh
kubectl -n kubesphere-system edit cc ks-installer
# 然后将 spec.springcloud.enabled 设置为 true，也可在里面对部署的 nacos 参数自由配置
```



### 部署配置 bookinfo

前提：在 ks 上安装后[创建企业空间、项目、用户和平台角色](https://kubesphere.com.cn/docs/quick-start/create-workspace-and-project/)再进行下面操作。

此示例中创建的企业空间为：xxx、项目名为：springcloud-demo、操作账户为：admin

#### 一、productpage

是本示例中的前端访问入口，此前端项目不用在 Spring Cloud 微服务中部署，在【应用负载中部署成普通的工作负载】。

1. ##### 创建工作负载并通过环境变量指定网关地址

   填写镜像：前面我们构建的镜像是 `hongzhouwei/productpage:v1`，若没有构建镜像可以用`hongzhouwei/productpage:latest`demo， 那么这儿我们也就填写对应的镜像地址。

   设置环境变量：（API_SERVER : http://gateway.springcloud-demo.svc:8080）

   因为在前端项目中通过反向代理 访问 API 网关，通过环境变量配置 API 网关地址。网关我们下一步再部署，网关名字我们就设为 gateway、端口设为8080，那么这儿访问网关的地址就是  http://gateway.springcloud-demo.svc:8080 。

   ![image-20220227233627671](docs/images/image-20220227233627671.png)

   ![image-20220325105400821](docs/images/image-20220325105400821.png)

   

2. ##### 创建服务并指定工作负载

   因为这个服务是需要对外访问的前端应用，所以需要创建服务并在后面需要通过应用路由暴露出来。

   创建服务时需要关联到对应的工作负载，并填写对应端口。（productpage 用的 3000 端口，通过应用路由访问配置80端口）

   ![image-20220323110817295](docs/images/image-20220323110817295.png)

3. ##### 开启项目网关，创建应用路由对外提供访问

   开启网关，可以就用 NodePort 方式，直接点确定。

   ![image-20220323110919468](docs/images/image-20220323110919468.png)

   创建应用路由：路由规则中设置成路由到前面创建的服务。

   创建后若访问不到，可能的问题有：

   1. 如果默认生成的内网的地址，那么需要在该路由中【更多操作】->【编辑YAML】把内网地址改成外网地址。
   2. 如果该端口访问不到，还需要在防火墙或安全组中将端口出来。

   ![image-20220323111048000](docs/images/image-20220323111048000.png)

   ![image-20220323112138111](docs/images/image-20220323112138111.png)

4. ##### 打开前端访问页面

   可以看到能访问到前端页面了，只是目前没有配置其他微服务，查看详情那些接口还请求不到数据。

![image-20220323112217042](docs/images/image-20220323112217042.png)

#### 二、gateway

gateway 是一个 spring-cloud-gateway 应用，作为后端 API 的入口。这儿使用的网关是基于 springcloud 定制化的网关，在里面支持网关路由管理。

使用镜像：`hongzhouwei/spring-cloud-gateway:latest` 端口设置：容器端口和服务端口都为8080

![image-20220324182749996](docs/images/image-20220324182749996.png)

创建网关后会 apply 相应的 service 和 deployment。



#### 三、details

details 提供了具体的书籍详情 API，我们可以通过 product id 获取书籍的详细信息。这个是springcloud微服务，需要在【 Spring Cloud 微服务中部署】。

1. ##### 创建实例

   命名为：details。（**注意：在创建实例命名微服务时，名字中不要出现下划线，否则会出现找不该服务的问题。**）

   填写镜像：前面我们构建的镜像是 `hongzhouwei/details:v1`，若没有构建镜像可以用`hongzhouwei/datails:latest`demo， 那么这儿我们也就填写对应的镜像地址。

   端口设置：设置8080端口

   **创建实例后，一般需要等会儿才可以看到服务注册上来。创建过程可在应用负载->工作负载中看到。**

   

2. ##### 在微服务网关中配置服务路由

   配置规则可参照 springcloud-gateway ，这儿主要配置以下几项：

   ```yaml
           - id: details
             uri: lb://details
             predicates:
               - Path=/api/v1/products/*
   ```

   注意 uri 中 lb://xxx ，这儿 xxx 是微服务的名字，**在创建实例命名微服务时，名字中不要出现下划线**，否则会出现找不该服务的问题。

   ![image-20220325111905287](docs/images/image-20220325111905287.png)

   

3. ##### 检查 productpage 中书籍详情是否正常显示

   可看到下面书籍详情页显示出来了。

   ![image-20220325112345534](docs/images/image-20220325112345534.png)



#### 四、reviews-v1

reviews 应用提供书籍评论相关的 API，可以通过配置开启是否展示评分。这个是springcloud微服务，需要在【 Spring Cloud 微服务中部署】

我们这儿部署 v1 版本的 reviews 后端服务，默认不展示书籍评分。

1. ##### 创建实例

   命名为：reviews-v1

   填写镜像：前面我们构建的镜像是 `hongzhouwei/reviews:v1`，若没有构建镜像可以用`hongzhouwei/reviews:latest`demo， 那么这儿我们也就填写对应的镜像地址。

   端口设置：设置8080端口

   

4. ##### 在微服务网关中配置服务路由

   配置规则可参照 springcloud-gateway ，这儿主要配置以下几项：

```
        - id: reviews-v1
          uri: lb://reviews-v1
          predicates:
            - Path=/api/v1/products/*/reviews
```



3. ##### 检查 productpage 中书籍评论是否正常显示

可看到下面书籍评论页显示出来了。

![image-20220325114454233](docs/images/image-20220325114454233.png)



## 服务管理



## 配置管理



## 微服务网关管理





## 问题记录





## 总结





