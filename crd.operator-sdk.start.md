## crd脚手架之operator-sdk的快速上手

### 前言
  本篇使用的[operator-sdk](https://github.com/operator-framework/operator-sdk)工具是0.18.2版本，是截至本篇文章时最新的operator-sdk版本

### 安装
  - 在operator-sdk的[release](https://github.com/operator-framework/operator-sdk/releases)页面下载不同系统的不同版本
  - [operator-sdk](https://github.com/operator-framework/operator-sdk)下载源码自行编译
```
  git clone https://github.com/operator-framework/operator-sdk
  $ cd operator-sdk
  $ git checkout master
  $ make tidy
  $ make install
```
### 创建一个go项目
执行命令`operator-sdk new <operator-name>`
生成的项目结构如下：
<operator-name>
├── build
│   ├── Dockerfile
│   └── bin
│       ├── entrypoint
│       └── user_setup
├── cmd
│   └── manager
│       └── main.go
├── deploy
│   ├── operator.yaml
│   ├── role.yaml
│   ├── role_binding.yaml
│   └── service_account.yaml
├── go.mod
├── go.sum
├── pkg
│   ├── apis
│   │   └── apis.go
│   └── controller
│       └── controller.go
├── tools.go
└── version
    └── version.go


### 生成自定义resource
例
```
cd <operator-name>
命令模版`operator-sdk add api --api-version $GROUP_NAME/$VERSION --kind $SERVICE_NAME`
如执行命令`operator-sdk add api --api-version app.example.com/v1alpha1 --kind AppService`
```


生成之后的目录较之前，多了如下结构：
<operator-name>
：
├── deploy
│   ├── crds                    // 该目录为新增
│   │   ├── app.example.com_appservices_crd.yaml
│   │   └── app.example.com_v1alpha1_appservice_cr.yaml
：
├── pkg
│   ├── apis
│   │   ├── addtoscheme_app_v1alpha1.go   // 该文件为新增
│   │   ├── apis.go
│   │   └── app                 // 该目录为新增
│   │       ├── group.go
│   │       └── v1alpha1
│   │           ├── appservice_types.go
│   │           ├── doc.go
│   │           ├── register.go
│   │           └── zz_generated.deepcopy.go

在deploy下多了crds目录，pkg/apis下多了app/和addtoscheme_app_v1alpha1.go等目录或文件

- deploy下多了crds目录：该文件下的*.yaml文件用于发布到k8s中的模版，可以自定义其内容然后使用kubectl apply -f *.yaml 发布至k8s中。
- pkg/apis/addtoscheme_app_v1alpha1.go：将自定义resource注册入AddToSchemes，使得gvk(GroupVersionKinds)知道该类型
- pkg/apis/app/：v1alpha1表示生成resource的版本号，其下的`*_types.go`生成了自定义的resource的结构体，只需要在`Spec`和`Status`中添加自定义字段就行

### 为自定义resource添加controller
例
```
cd <operator-name>
命令模版`operator-sdk add controller --api-version $GROUP_NAME/$VERSION --kind $SERVICE_NAME`
如执行命令`operator-sdk add controller --api-version app.example.com/v1alpha1 --kind AppService`
```

生成之后的目录较之前，多了如下结构：
<operator-name>
：
├── pkg
：
│   └── controller
│       ├── add_appservice.go       // 该文件为新增
│       ├── appservice              // 该目录为新增
│       │   └── appservice_controller.go

- controller/add_appservice.go：将自定义生成的controller注入到manager中
- controller/appservice/：主要实现Reconcile()方法，监听自定义resource的添加，修改，删除等事件，并作出相应的操作


### 为已有resource添加controller
例
```
cd <operator-name>
// 创建一个关于Deployment监听的controller
operator-sdk add controller  --api-version=k8s.io.api/v1 --kind=Deployment  --custom-api-import=k8s.io/api/apps/v1
```
生成之后的目录较之前，多了如下结构：
<operator-name>
：
├── pkg
：  ：
│   └── controller
│       ├── add_deployment.go             // 新增文件
│       └── deployment                    // 新增
│           └── deployment_controller.go


- 在deployment_controller.go中的Reconcile()实现自己的监听逻辑

### 本地运行crds以调试
```
export OPERATOR_NAME=<operator-name>
operator-sdk run local --watch-namespace=default
```

- 这样就能本地运行该crds，方便调试。前提是已有一个k8s的集群下，如果没有，建议用[k3d](https://github.com/rancher/k3d)快递搭建一个k8s集群

### 集群内运行crds
```
operator-sdk build quay.io/example/<operator-name>:v0.0.1
sed -i 's|REPLACE_IMAGE|quay.io/example/<operator-name>:v0.0.1|g' deploy/operator.yaml
docker push quay.io/example/<operator-name>:v0.0.1
```

这里需要先将crd打包成一个镜像，并用镜像地址替换掉deploy/operator.yaml中的"REPLACE_IMAGE"，即将`image`部分改为打包的有效地址
执行如下命令即可将crd部署到集群中
```
kubectl create -f deploy
```

### 使用operator-olm管理crd
见[crd.operator-sdk.olm.md](./crd.operator-sdk.olm.md)

### OpenAPI validation
对于自定义resource，在某些场景下，需要对某些字段进行校验，可以使用添加注释的方法添加校验：
```
// +kubebuilder:validation:Enum=Lion;Wolf;Dragon
type Alias string
```
上述注释限定Alias只能为Lion;Wolf;Dragon其中之一，否则校验通不过。
关于更多的[kubebuiler的校验方法](https://book.kubebuilder.io/reference/markers/crd-validation.html)可自行查阅文档。

> 注：如果此时你已经用`operator-sdk olm`管理operator的生命周期，在重新添加校验之后，直接执行`kubectl create -f deploy/crds/<gourp>_<version>_<kind>_cr.yaml`进行验证校验字段是否生效，会发现即使是错误的字段，kubectl也告诉你<Kind>创建成功

正确操作：需要重新生成crd，manifests和csv；然后使用`operator-sdk cleanup packagemanifests --operator-version <version>`清楚掉当前的的operator的olm，重新执行命令
`operator-sdk run packagemanifests --operator-version <version>`，此后新添加的validation才能生效。

若没用olm管理operator，而是直接使用kubectl创建资源，在重新生成之后创建则校验立即生效。


### Unit Testing
