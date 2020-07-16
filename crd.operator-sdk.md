## crd脚手架之operator-sdk


## 什么是operator-sdk
  [operator-sdk](https://github.com/operator-framework/operator-sdk)是一种k8s的crd脚手架，用户可用该工具快速构建一个属于自己的crd项目


## operator-project项目
  一个operator-project项目目录如下：
```
  <operator-name>
    ├─build             // 存放一些编译后的二进制文件
    │  └─bin
    ├─cmd
    │  └─manager        // 主函数入口
    ├─deploy            // 在k8s中发布crd需要的各种资源
    │  ├─crds
    │  └─olm-catalog    // operator-sdk生成由olm管理的一些必要文件
    │      └─memcached-operator
    │          └─manifests
    ├─pkg               // pkg包含自己需要定制的api和controller
    │  ├─apis           // 自定义api
    │  │  └─cache
    │  │      └─v1alpha1// 定义的api版本
    │  └─controller     // 自定义controller，用户处理自定义api或k8s已有的api
    │      └─memcached	// 定义的controller名称
    ├─test         
    │  └─e2e			// end-to-end测试
    └─version
````
      
## operator-project项目源码
 > 此处使用[operator-sdk-samples](https://github.com/operator-framework/operator-sdk-samples/tree/master/go/memcached-operator)为例展示源码

### api与controller

#### *_types.go
  pkg/apis/cache/v1alpha1/memcached_types.go中定义了`Memcached`；此部分由operator-sdk生成自定义crd的api。
  ```
  // +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// Memcached is the Schema for the memcacheds API
// +k8s:openapi-gen=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:path=memcacheds,scope=Namespaced
type Memcached struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   MemcachedSpec   `json:"spec,omitempty"`
	Status MemcachedStatus `json:"status,omitempty"`
}


// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.

// MemcachedSpec defines the desired state of Memcached
// +k8s:openapi-gen=true
type MemcachedSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "operator-sdk generate k8s" to regenerate code after modifying this file
	// Add custom validation using kubebuilder tags: https://book-v1.book.kubebuilder.io/beyond_basics/generating_crd.html

	// Size is the size of the memcached deployment
	Size int32 `json:"size"`
}

// MemcachedStatus defines the observed state of Memcached
// +k8s:openapi-gen=true
type MemcachedStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "operator-sdk generate k8s" to regenerate code after modifying this file
	// Add custom validation using kubebuilder tags: https://book-v1.book.kubebuilder.io/beyond_basics/generating_crd.html

	// Nodes are the names of the memcached pods
	// +listType=set
	Nodes []string `json:"nodes"`
}
  ```

  其中`MemcachedSpec`和`MemcachedStatus`由自己根据业务自定义

#### *_controller.go
  pkg/controller/memcached/memcached_controller.go中定义了`ReconcileMemcached`；此部分由operator-sdk生成自定义crd的controller，也可以生成一个k8s已有的controller，实现对已有resource的监听。
  ```
  
// ReconcileMemcached reconciles a Memcached object
type ReconcileMemcached struct {
	// TODO: Clarify the split client
	// This client, initialized using mgr.Client() above, is a split client
	// that reads objects from the cache and writes to the apiserver
	client client.Client
	scheme *runtime.Scheme
}

// Reconcile reads that state of the cluster for a Memcached object and makes changes based on the state read
// and what is in the Memcached.Spec
// TODO(user): Modify this Reconcile function to implement your Controller logic.  This example creates
// a Memcached Deployment for each Memcached CR
// Note:
// The Controller will requeue the Request to be processed again if the returned error is non-nil or
// Result.Requeue is true, otherwise upon completion it will remove the work from the queue.
func (r *ReconcileMemcached) Reconcile(request reconcile.Request) (reconcile.Result, error) {
    // 省略代码
	// 实现`Memcached`的监听；包含查询；更新；删除等操作
    // 省略代码
	return reconcile.Result{}, nil
}

  ```


### operator-project的整体流程

#### main.go
  ```
import (
  "github.com/operator-framework/operator-sdk-samples/go/memcached-operator/pkg/apis"
	"github.com/operator-framework/operator-sdk-samples/go/memcached-operator/pkg/controller"
)
func main() {
  // 省略代码

	// 创建一个manager
	mgr, err := manager.New(cfg, options)
	if err != nil {
		log.Error(err, "")
		os.Exit(1)
	}

	log.Info("Registering Components.")

	// 将自定义的api资源注入scheme中
	if err := apis.AddToScheme(mgr.GetScheme()); err != nil {
		log.Error(err, "")
		os.Exit(1)
	}

	// Setup all Controllers
  // 见下部分解析
	if err := controller.AddToManager(mgr); err != nil {
		log.Error(err, "")
		os.Exit(1)
	}


  // ...
	// 启动manager
	if err := mgr.Start(signals.SetupSignalHandler()); err != nil {
		log.Error(err, "Manager exited non-zero")
		os.Exit(1)
	}
}
  ```

#### controller.AddToManager(mgr)
  ```
  // AddToManagerFuncs在init()函数中已经被append了 memcached.Add方法
var AddToManagerFuncs []func(manager.Manager) error

// AddToManager adds all Controllers to the Manager
func AddToManager(m manager.Manager) error {
  // 循环AddToManagerFuncs并把m作为参数传入其中
	for _, f := range AddToManagerFuncs {
		if err := f(m); err != nil {
			return err
		}
	}
	return nil

  ```

#### memcached.Add()
```
func Add(mgr manager.Manager) error {
	return add(mgr, newReconciler(mgr))
}

// newReconciler returns a new reconcile.Reconciler
func newReconciler(mgr manager.Manager) reconcile.Reconciler {
	return &ReconcileMemcached{client: mgr.GetClient(), scheme: mgr.GetScheme()}
}

// add adds a new Controller to mgr with r as the reconcile.Reconciler
func add(mgr manager.Manager, r reconcile.Reconciler) error {
	// Create a new controller
  // 见`controller.New()`一节
	c, err := controller.New("memcached-controller", mgr, controller.Options{Reconciler: r})
	if err != nil {
		return err
	}

	// Watch for changes to primary resource Memcached
  // reconcile.Reconciler里的数据来源
	err = c.Watch(&source.Kind{Type: &cachev1alpha1.Memcached{}}, &handler.EnqueueRequestForObject{})
	if err != nil {
		return err
	}

	// TODO(user): Modify this to be the types you create that are owned by the primary resource
	// Watch for changes to secondary resource Pods and requeue the owner Memcached
	err = c.Watch(&source.Kind{Type: &appsv1.Deployment{}}, &handler.EnqueueRequestForOwner{
		IsController: true,
		OwnerType:    &cachev1alpha1.Memcached{},
	})
	if err != nil {
		return err
	}

	err = c.Watch(&source.Kind{Type: &corev1.Service{}}, &handler.EnqueueRequestForOwner{
		IsController: true,
		OwnerType:    &cachev1alpha1.Memcached{},
	})
	if err != nil {
		return err
	}

	return nil
}
```

#### reconcile.Reconciler里的数据来源  handler.EnqueueRequestForObject
```
type EnqueueRequestForObject struct{}

// Create implements EventHandler
func (e *EnqueueRequestForObject) Create(evt event.CreateEvent, q workqueue.RateLimitingInterface) {
	if evt.Meta == nil {
		enqueueLog.Error(nil, "CreateEvent received with no metadata", "event", evt)
		return
	}

  // 封装成reconcile.Request
	q.Add(reconcile.Request{NamespacedName: types.NamespacedName{
		Name:      evt.Meta.GetName(),
		Namespace: evt.Meta.GetNamespace(),
	}})
}

// Update implements EventHandler
func (e *EnqueueRequestForObject) Update(evt event.UpdateEvent, q workqueue.RateLimitingInterface) {
	if evt.MetaOld != nil {
      // 封装成reconcile.Request
		q.Add(reconcile.Request{NamespacedName: types.NamespacedName{
			Name:      evt.MetaOld.GetName(),
			Namespace: evt.MetaOld.GetNamespace(),
		}})
	} else {
		enqueueLog.Error(nil, "UpdateEvent received with no old metadata", "event", evt)
	}

	if evt.MetaNew != nil {
    // 封装成reconcile.Request
		q.Add(reconcile.Request{NamespacedName: types.NamespacedName{
			Name:      evt.MetaNew.GetName(),
			Namespace: evt.MetaNew.GetNamespace(),
		}})
	} else {
		enqueueLog.Error(nil, "UpdateEvent received with no new metadata", "event", evt)
	}
}

// Delete implements EventHandler
func (e *EnqueueRequestForObject) Delete(evt event.DeleteEvent, q workqueue.RateLimitingInterface) {
	if evt.Meta == nil {
		enqueueLog.Error(nil, "DeleteEvent received with no metadata", "event", evt)
		return
	}
  // 封装成reconcile.Request
	q.Add(reconcile.Request{NamespacedName: types.NamespacedName{
		Name:      evt.Meta.GetName(),
		Namespace: evt.Meta.GetNamespace(),
	}})
}
```


#### controller.New()

```
// New returns a new Controller registered with the Manager.  The Manager will ensure that shared Caches have
// been synced before the Controller is Started.
func New(name string, mgr manager.Manager, options Options) (Controller, error) {
	if options.Reconciler == nil {
		return nil, fmt.Errorf("must specify Reconciler")
	}

	if len(name) == 0 {
		return nil, fmt.Errorf("must specify Name for Controller")
	}

	if options.MaxConcurrentReconciles <= 0 {
		options.MaxConcurrentReconciles = 1
	}

	// Inject dependencies into Reconciler
  // SetFields将mgr的一些配置注入到options.Reconciler中
	if err := mgr.SetFields(options.Reconciler); err != nil {
		return nil, err
	}

	
  // 初始化一个controller
	c := &controller.Controller{
		Do:                      options.Reconciler,
		Cache:                   mgr.GetCache(),
		Config:                  mgr.GetConfig(),
		Scheme:                  mgr.GetScheme(),
		Client:                  mgr.GetClient(),
		Recorder:                mgr.GetEventRecorderFor(name),
		Queue:                   workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), name),
		MaxConcurrentReconciles: options.MaxConcurrentReconciles,
		Name:                    name,
	}

	// 将controller组件添加到mgr里面，controller会在mgr的Add方法或Start方法中启动（执行controller的Start方法）
	return c, mgr.Add(c)
}
```

#### controller.Controller之reconcile.Request的消费

```

// 实现Runnable接口
func (c *Controller) Start(stop <-chan struct{}) error {
  // ...
  // 开启MaxConcurrentReconciles个goroutine消费
	for i := 0; i < c.MaxConcurrentReconciles; i++ {
		// woker()进行消费
		go wait.Until(c.worker, c.JitterPeriod, stop)
	}
}

// worker runs a worker thread that just dequeues items, processes them, and marks them done.
// It enforces that the reconcileHandler is never invoked concurrently with the same object.
func (c *Controller) worker() {
  // 为true的状态下一直循环调用
	for c.processNextWorkItem() {
	}
}

// processNextWorkItem will read a single work item off the workqueue and
// attempt to process it, by calling the reconcileHandler.
func (c *Controller) processNextWorkItem() bool {
  // 从queue中获取request
	obj, shutdown := c.Queue.Get()
	if shutdown {
		// Stop working
		return false
	}
	defer c.Queue.Done(obj)
  // 处理queue中获取到的obj
	return c.reconcileHandler(obj)
}

func (c *Controller) reconcileHandler(obj interface{}) bool {
	// Update metrics after processing each item
	reconcileStartTS := time.Now()
	defer func() {
		c.updateMetrics(time.Now().Sub(reconcileStartTS))
	}()

	var req reconcile.Request
	var ok bool
	if req, ok = obj.(reconcile.Request); !ok {
    // 断言不是reconcile.Request类型，直接丢弃该obj
		c.Queue.Forget(obj)
		return true
	}
  
  // 调用Reconcile，既自定义的controller的Reconcile方法
  // 返回err时，进行限速重入队列，等待下次消费
	if result, err := c.Do.Reconcile(req); err != nil {
		c.Queue.AddRateLimited(req)
		return false
	} else if result.RequeueAfter > 0 {
    // 延时重入队列
		c.Queue.Forget(obj)
		c.Queue.AddAfter(req, result.RequeueAfter)
		return true
	} else if result.Requeue {
    // 立即重入队列
		c.Queue.AddRateLimited(req)
		ctrlmetrics.ReconcileTotal.WithLabelValues(c.Name, "requeue").Inc()
		return true
	}
  // 不在重入队列
	c.Queue.Forget(obj)
	return true
}

```


### 总结
- 知道了sigs.k8s.io\controller-runtime\pkg\internal\controller\controller.go中的数据的来源（watch中添加）和去向（reconcileHandler中消费）
- 知道了在合适会调用自定义的controller的Reconciler函数
