### 关于SharedInformerFactory的一些介绍

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

4. 例如调用 informer.Core().V1().Pods().Informer()/informer.Core().V1().Pods().Lister(),参考[informer_indexer.md](./informer_indexer.md)

5. SharedInformerFactory是一个Informer工厂，可以获取不同版本的不同资源。