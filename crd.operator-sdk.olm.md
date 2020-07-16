## operator-sdk olm管理operator项目

### 介绍
operator-sdk olm是集群中的一组资源，全称是`operator-sdk Operator Lifecycle Manager`，专门用于管理operator的生命周期，operator-sdk支持创建用户olm部署的清单以及在开启了olm的k8s集群上测试你的operator应用。
本文档简要地介绍了如何使操作员使用捆绑软件进行OLM就绪：
- 所有关于`operator-frameword manifest`的命令已由operator-sdk提供：[CLI overview](https://sdk.operatorframework.io/docs/olm-integration/cli-overview/)
- 生成`operator-frameword manifest`:[generation overview](https://sdk.operatorframework.io/docs/olm-integration/generating-a-csv)

NOTE:
- 本文使用的operator-sdk是`v0.18.2`
- 本文没有使用make进行manifest和bundle的生成

### Setup
olm是建立在一个operator项目之上的，所有目前你至少有一个operator-sdk项目开可以开始本文的一些内容；如果没有，可以跳转到operator-sdk的[开始](./crd.operator-sdk.start.md)看看详细，同时可以了解一下operator-sdk项目中`Reconcile`的[详细过程](./crd.operator-sdk.md)等。

### Enabling OLM
确保olm在当前的集群中是可用的。
NOTE： 可能当前集群已启用olm，只是不在默认的namespace（olm）之下，因此可以指定参数`--olm-namespace=<namespace>`进行查看命令或者打包packagemanifests等。

执行命令查看olm在当前集群中的状态
```
operator-sdk olm status
```
正常已启用olm的返回结果
```
xxx ~ % operator-sdk olm status 
INFO[0000] Fetching CRDs for version "0.15.1"           
INFO[0091] Fetching resources for version "0.15.1"      
INFO[0117] Successfully got OLM status for version "0.15.1" 

NAME                                            NAMESPACE    KIND                        STATUS
olm                                                          Namespace                   Installed
subscriptions.operators.coreos.com                           CustomResourceDefinition    Installed
operatorgroups.operators.coreos.com                          CustomResourceDefinition    Installed
installplans.operators.coreos.com                            CustomResourceDefinition    Installed
clusterserviceversions.operators.coreos.com                  CustomResourceDefinition    Installed
aggregate-olm-edit                                           ClusterRole                 Installed
catalog-operator                                olm          Deployment                  Installed
olm-operator                                    olm          Deployment                  Installed
operatorhubio-catalog                           olm          CatalogSource               Installed
olm-operators                                   olm          OperatorGroup               Installed
aggregate-olm-view                                           ClusterRole                 Installed
operators                                                    Namespace                   Installed
global-operators                                operators    OperatorGroup               Installed
olm-operator-serviceaccount                     olm          ServiceAccount              Installed
packageserver                                   olm          ClusterServiceVersion       Installed
system:controller:operator-lifecycle-manager                 ClusterRole                 Installed
catalogsources.operators.coreos.com                          CustomResourceDefinition    Installed
olm-operator-binding-olm                                     ClusterRoleBinding          Installed
```
若提示未安装，则使用命令`operator-sdk olm install`进行olm的安装，此处可能要使用科学上网法才能安装成功。
通过`operator-sdk olm uninstall`可进行卸载。


### Creating a manifests
1. manifest是operator-sdk 打包的csv(cluster service version)的统称。
2. 打包manifests主要是为了用operator-sdk将operator部署到集群中。

生成命令如下：
```
operator-sdk generate csv --csv-version=0.1.0
```
提供一个--csv-verion的版本号。

此时项目目录下会多出如下结构
```
<operator-name>
├── deploy
│   ├── olm-catalog
│   │   └── operator-name
│   │       └── manifests   // 打包成的manifests
│   │           ├── <group_name>_<service_name>_crd.yaml
│   │           └── <operator-name>.clusterserviceversion.yaml
```

此时就打包成了manifests


·打包需要发布的版本·
`operator-sdk generate csv --make-manifests=false --csv-version 0.1.0`
此时的项目目录下会多出如下结构
```
<operator-name>
├── deploy
│   ├── olm-catalog
│   │   └── operator-name
│   │       ├── 0.1.0   //  生成的待发布的manifests版本
│   │       │   └── <operator-name>.v0.1.0.clusterserviceversion.yaml
```
查看其中的yaml文件，可发现在`image`处写的是`REPLACE_IMAGE`，即此处地方image的值需要替换为正确的image地址，以发布operator应用时正确的拉取镜像。

### Creating a bundle
bundle是用于打包镜像的一些列文件
`operator-sdk generate bundle`打包bundle

此时的项目目录下会多出如下结构
```
<operator-name>
├── bundle.Dockerfile
├── deploy
│   ├── olm-catalog
│   │   └── operator-name
│   │       ├── metadata  // 一些元数据
│   │       │   └── annotations.yaml
│   │       └── operator-name.package.yaml
```


此时bundle就已经生成
使用docker打包镜像，并将其打上某个docker register的标签：
`docker build -f bundle.Dockerfile -t quay.io/<username>/memcached-operator:v0.1.0`
然后将其push到dock register中。
`docker push quay.io/<username>/memcached-operator:v0.1.0`

### Deploying an Operator with OLM
距离我们使用olm管理发布自己的operator仅差最后一步
将上述提到的`image: REPLACE_NAME`的值替换为上一步bundle所打的tag，然后我们就可以使用如下命令发布operator应用至集群中了
`operator-sdk run packagemanifests --operator-version 0.0.1`
```
INFO[0000] Running operator from directory packagemanifests
INFO[0000] Creating memcached-operator registry         
INFO[0000]   Creating ConfigMap "olm/memcached-operator-registry-manifests-package"
INFO[0000]   Creating ConfigMap "olm/memcached-operator-registry-manifests-0-0-1"
INFO[0000]   Creating Deployment "olm/memcached-operator-registry-server"
INFO[0000]   Creating Service "olm/memcached-operator-registry-server"
INFO[0000] Waiting for Deployment "olm/memcached-operator-registry-server" rollout to complete
INFO[0000]   Waiting for Deployment "olm/memcached-operator-registry-server" to rollout: 0 of 1 updated replicas are available
INFO[0066]   Deployment "olm/memcached-operator-registry-server" successfully rolled out
INFO[0066] Creating resources                           
INFO[0066]   Creating CatalogSource "default/memcached-operator-ocs"
INFO[0066]   Creating Subscription "default/memcached-operator-v0-0-1-sub"
INFO[0066]   Creating OperatorGroup "default/operator-sdk-og"
INFO[0066] Waiting for ClusterServiceVersion "default/memcached-operator.v0.0.1" to reach 'Succeeded' phase
INFO[0066]   Waiting for ClusterServiceVersion "default/memcached-operator.v0.0.1" to appear
INFO[0073]   Found ClusterServiceVersion "default/memcached-operator.v0.0.1" phase: Pending
INFO[0077]   Found ClusterServiceVersion "default/memcached-operator.v0.0.1" phase: InstallReady
INFO[0078]   Found ClusterServiceVersion "default/memcached-operator.v0.0.1" phase: Installing
INFO[0036]   Found ClusterServiceVersion "default/memcached-operator.v0.0.1" phase: Succeeded
INFO[0037] Successfully installed "memcached-operator.v0.0.1" on OLM version "0.15.1"
```


需要将operator应用从集群中卸载时：
`operator-sdk cleanup packagemanifests --operator-version 0.0.1`
```
INFO[0000] Deleting resources
INFO[0000]   Deleting CatalogSource "default/memcached-operator-ocs"
INFO[0000]   Deleting Subscription "default/memcached-operator-v0-0-1-sub"
INFO[0000]   Deleting OperatorGroup "default/operator-sdk-og"
INFO[0000]   Deleting CustomResourceDefinition "default/memcacheds.example.com"
INFO[0000]   Deleting ClusterServiceVersion "default/memcached-operator.v0.0.1"
INFO[0000]   Waiting for deleted resources to disappear
INFO[0001] Successfully uninstalled "memcached-operator.v0.0.1" on OLM version "0.15.1"
```
