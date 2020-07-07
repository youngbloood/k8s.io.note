## 关于SharedInformerFactory的一些介绍

1. 路径`k8s.io\client-go\informers\factory.go:189`
2. define
```
// SharedInformerFactory provides shared informers for resources in all known
// API group versions.
type SharedInformerFactory interface {
	internalinterfaces.SharedInformerFactory
	ForResource(resource schema.GroupVersionResource) (GenericInformer, error)
	WaitForCacheSync(stopCh <-chan struct{}) map[reflect.Type]bool

	Admissionregistration() admissionregistration.Interface
	Apps() apps.Interface
	Auditregistration() auditregistration.Interface
	Autoscaling() autoscaling.Interface
	Batch() batch.Interface
	Certificates() certificates.Interface
	Coordination() coordination.Interface
	Core() core.Interface
	Discovery() discovery.Interface
	Events() events.Interface
	Extensions() extensions.Interface
	Networking() networking.Interface
	Node() node.Interface
	Policy() policy.Interface
	Rbac() rbac.Interface
	Scheduling() scheduling.Interface
	Settings() settings.Interface
	Storage() storage.Interface
}
```
3. internalinterfaces.SharedInformerFactory定义
```
// SharedInformerFactory a small interface to allow for adding an informer without an import cycle
type SharedInformerFactory interface {
	Start(stopCh <-chan struct{})
	InformerFor(obj runtime.Object, newFunc NewInformerFunc) cache.SharedIndexInformer
}
```

4. 例如调用 informer.Core().V1().Pods().Informer()或者informer.Core().V1().Pods().Lister(),参考[informer_indexer.md](./informer_indexer.md)

5. SharedInformerFactory是一个Informer工厂，用于获取不同版本的不同资源。

6. 实现SharedInformerFactory接口的类型定义
```
type sharedInformerFactory struct {
	client           kubernetes.Interface
	namespace        string
	tweakListOptions internalinterfaces.TweakListOptionsFunc
	lock             sync.Mutex
	defaultResync    time.Duration
	customResync     map[reflect.Type]time.Duration

	// 存储cache.SharedIndexInformer的map
	informers map[reflect.Type]cache.SharedIndexInformer
	
	// 标识哪些resource的informer已启动
	startedInformers map[reflect.Type]bool
}
```


实现`internalinterfaces.SharedInformerFactory`接口的2个方法
```
// Start. 开启所有未启动的resource的informer
func (f *sharedInformerFactory) Start(stopCh <-chan struct{}) {
	f.lock.Lock()
	defer f.lock.Unlock()

	for informerType, informer := range f.informers {
		// 如果某resource的informer未启动，则启动
		if !f.startedInformers[informerType] {
			go informer.Run(stopCh)
			f.startedInformers[informerType] = true
		}
	}
}
func (f *sharedInformerFactory) InformerFor(obj runtime.Object, newFunc internalinterfaces.NewInformerFunc) cache.SharedIndexInformer {
	f.lock.Lock()
	defer f.lock.Unlock()

	informerType := reflect.TypeOf(obj)
	informer, exists := f.informers[informerType]
	// 该resource已注册，直接返回
	if exists {
		return informer
	}
	// 获取该resource设定的同步间隔时间
	resyncPeriod, exists := f.customResync[informerType]
	if !exists {
		resyncPeriod = f.defaultResync
	}
	// 未注册，调用newFunc生成一个该resource的informer，并写入map中
	informer = newFunc(f.client, resyncPeriod)
	f.informers[informerType] = informer

	return informer
}
```

其他接口的一些实现

```
...省略...
func (f *sharedInformerFactory) Core() core.Interface {
	return core.New(f, f.namespace, f.tweakListOptions)
}

func (f *sharedInformerFactory) Discovery() discovery.Interface {
	return discovery.New(f, f.namespace, f.tweakListOptions)
}

func (f *sharedInformerFactory) Events() events.Interface {
	return events.New(f, f.namespace, f.tweakListOptions)
}
...省略...
```

此处查看一下`core.New(f, f.namespace, f.tweakListOptions)`的实现

```
// Interface provides access to each of this group's versions.
type Interface interface {
	// V1 provides access to shared informers for resources in V1.
	V1() v1.Interface
}

type group struct {
	factory          internalinterfaces.SharedInformerFactory
	namespace        string
	tweakListOptions internalinterfaces.TweakListOptionsFunc
}

// New returns a new Interface.
func New(f internalinterfaces.SharedInformerFactory, namespace string, tweakListOptions internalinterfaces.TweakListOptionsFunc) Interface {
	return &group{factory: f, namespace: namespace, tweakListOptions: tweakListOptions}
}

// V1 returns a new v1.Interface.
func (g *group) V1() v1.Interface {
	return v1.New(g.factory, g.namespace, g.tweakListOptions)
}
```

由于core只有一个V1版本的设定，所以只实现了`V1() v1.Interface`接口

接下来看看`v1.New(g.factory, g.namespace, g.tweakListOptions)`和`v1.Interface`接口的实现和定义

v1.Interface
```
// Interface provides access to all the informers in this group version.
type Interface interface {
	// ComponentStatuses returns a ComponentStatusInformer.
	ComponentStatuses() ComponentStatusInformer
	// ConfigMaps returns a ConfigMapInformer.
	ConfigMaps() ConfigMapInformer
	// Endpoints returns a EndpointsInformer.
	Endpoints() EndpointsInformer
	// Events returns a EventInformer.
	Events() EventInformer
	// LimitRanges returns a LimitRangeInformer.
	LimitRanges() LimitRangeInformer
	// Namespaces returns a NamespaceInformer.
	Namespaces() NamespaceInformer
	// Nodes returns a NodeInformer.
	Nodes() NodeInformer
	// PersistentVolumes returns a PersistentVolumeInformer.
	PersistentVolumes() PersistentVolumeInformer
	// PersistentVolumeClaims returns a PersistentVolumeClaimInformer.
	PersistentVolumeClaims() PersistentVolumeClaimInformer
	// Pods returns a PodInformer.
	Pods() PodInformer
	// PodTemplates returns a PodTemplateInformer.
	PodTemplates() PodTemplateInformer
	// ReplicationControllers returns a ReplicationControllerInformer.
	ReplicationControllers() ReplicationControllerInformer
	// ResourceQuotas returns a ResourceQuotaInformer.
	ResourceQuotas() ResourceQuotaInformer
	// Secrets returns a SecretInformer.
	Secrets() SecretInformer
	// Services returns a ServiceInformer.
	Services() ServiceInformer
	// ServiceAccounts returns a ServiceAccountInformer.
	ServiceAccounts() ServiceAccountInformer
}
```

```
type version struct {
	factory          internalinterfaces.SharedInformerFactory
	namespace        string
	tweakListOptions internalinterfaces.TweakListOptionsFunc
}

// New returns a new Interface.
func New(f internalinterfaces.SharedInformerFactory, namespace string, tweakListOptions internalinterfaces.TweakListOptionsFunc) Interface {
	return &version{factory: f, namespace: namespace, tweakListOptions: tweakListOptions}
}

...省略...

// Pods returns a PodInformer.
// 实现v1.Interface里的Pods() PodInformer方法
func (v *version) Pods() PodInformer {
	return &podInformer{factory: v.factory, namespace: v.namespace, tweakListOptions: v.tweakListOptions}
}

...省略...
```

`PodInformer`和`podInformer`
```
type PodInformer interface {
	Informer() cache.SharedIndexInformer
	Lister() v1.PodLister
}

// 实现PodInformer接口
type podInformer struct {
	factory          internalinterfaces.SharedInformerFactory
	tweakListOptions internalinterfaces.TweakListOptionsFunc
	namespace        string
}

func (f *podInformer) Lister() v1.PodLister {
	return v1.NewPodLister(f.Informer().GetIndexer())
}

func (f *podInformer) Informer() cache.SharedIndexInformer {
	return f.factory.InformerFor(&corev1.Pod{}, f.defaultInformer)
}

func NewFilteredPodInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {
	return cache.NewSharedIndexInformer(
		&cache.ListWatch{
			ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.CoreV1().Pods(namespace).List(options)
			},
			WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.CoreV1().Pods(namespace).Watch(options)
			},
		},
		&corev1.Pod{},
		resyncPeriod,
		indexers,
	)
}

func (f *podInformer) defaultInformer(client kubernetes.Interface, resyncPeriod time.Duration) cache.SharedIndexInformer {
	return NewFilteredPodInformer(client, f.namespace, resyncPeriod, cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc}, f.tweakListOptions)
}

其中`cache.SharedIndexInformer`的实现细节可以跳转[这里](./informer_indexer.md)
```
## 总结
- 这样一层一层的接口封装与实现，就实现了`SharedInformerFactory`接口。
- 其中最重要的是`internalinterfaces.SharedInformerFactory`接口的实现，多理解该接口下两个方法的实现
