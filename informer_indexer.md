# client-go下的cache.SharedInformer的数据流向

## 一个cache.SharedIndexInformer的实现：
- 构造函数：`NewSharedIndexInformer(lw ListerWatcher, objType runtime.Object, defaultEventHandlerResyncPeriod time.Duration, indexers Indexers)`

  `lw：ListerWatcher`:是数据的来源处，通过http从数据源处获取数据
   
   `objType`:是资源类型

   `defaultEventHandlerResyncPeriod`:默认事件处理器重同步周期
   
   `indexers`:本地的indexer，用于构造一个 Indexer interface

- 主要配置：包含一个indexer（本地缓存）和一个`listerwatcher`(用于获取远程的资源)，`processor *sharedProcessor`类型

- 在`cache.SharedIndexInformer.Run(stop <-chan struct{})`时，创建一个`controller，controller`包含 一个`config`和一个`reflector`，`config`中包含如上2个主要配置，`Run()`驱动了informer从`ListerWatcher`中获取数据，并将数据改为event写入local cache中，最终交由注册如informer中的handler进行event的处理。

- 执行`controller.Run(stop <-chan struct{})`时，创建一个`Reflector`，该`Reflector`包含一个store（local cache，即一个indexer）和`listerwather`，均为上面主要配置

- 执行`Reflector.Run(stop <-chan struct{})`，会根据相应的机制从`listerwatcher`中获得数据，不断add到store/indexer（local cache）中

- `controller.processLoop()`方法会循环从queue(store)中pop出元素，交由`cache.SharedIndexInformer.HandleDeltas(obj interface{})error`处理该元素，将该元素最终交由`processor.distribute()`进行处理

- `processor.distribute()`会写入`sharedProcessor`里的`listeners([]*processorListener)`和`syncingListeners([]*processorListener)`中，最终进入`listener`的`addCh chan interface{}`中，由`processorListener.pop()`函数取出并包装放`入nextCh chan interface{}`中，由`processorListener.run()`函数从`nextCh`中取出，交由下一步的handler进行处理。

- `cache.SharedIndexInformer.AddEventHandler(handler ResourceEventHandler)`会注册handler在`SharedInformer`中，该handler就是上一步中的handler接收器，专门处理`nextCh`中收到的事件

## 类Informer的一些其他细节

- 以`replicaset`为例
1. 代码位置`k8s.io/client-go/informers/app/v1/replicaset.go:36`

```
// ReplicaSetInformer provides access to a shared informer and lister for
// ReplicaSets.
type ReplicaSetInformer interface {
	Informer() cache.SharedIndexInformer
	Lister() v1.ReplicaSetLister
}
// 其他informer类似，仅Lister()返回的类型不一样而已。
```
2. `ReplicaSetInformer`的实现
```
type replicaSetInformer struct {
	factory          internalinterfaces.SharedInformerFactory
	tweakListOptions internalinterfaces.TweakListOptionsFunc
	namespace        string
}

// NewReplicaSetInformer constructs a new informer for ReplicaSet type.
// Always prefer using an informer factory to get a shared informer instead of getting an independent
// one. This reduces memory footprint and number of connections to the server.
func NewReplicaSetInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers) cache.SharedIndexInformer {
	return NewFilteredReplicaSetInformer(client, namespace, resyncPeriod, indexers, nil)
}

// NewFilteredReplicaSetInformer constructs a new informer for ReplicaSet type.
// Always prefer using an informer factory to get a shared informer instead of getting an independent
// one. This reduces memory footprint and number of connections to the server.
func NewFilteredReplicaSetInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {
	return cache.NewSharedIndexInformer(
		&cache.ListWatch{
			ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.AppsV1().ReplicaSets(namespace).List(options)
			},
			WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.AppsV1().ReplicaSets(namespace).Watch(options)
			},
		},
		&appsv1.ReplicaSet{},
		resyncPeriod,
		indexers,
	)
}

func (f *replicaSetInformer) defaultInformer(client kubernetes.Interface, resyncPeriod time.Duration) cache.SharedIndexInformer {
	return NewFilteredReplicaSetInformer(client, f.namespace, resyncPeriod, cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc}, f.tweakListOptions)
}

func (f *replicaSetInformer) Informer() cache.SharedIndexInformer {
	return f.factory.InformerFor(&appsv1.ReplicaSet{}, f.defaultInformer)
}

func (f *replicaSetInformer) Lister() v1.ReplicaSetLister {
	return v1.NewReplicaSetLister(f.Informer().GetIndexer())
}

```
- 在调用`Informer()`时，会直接从f.factory的缓存中取出`cache.SharedIndexInformer`，如果f.factory中没有，则调用`f.defaultInformer`初始化一个`cache.SharedIndexInfomer`并返回。

- 在调用`Lister()`时，会调用`f.Informer().GetIndexer()`，该函数返回的是local cache的数据。所以`Lister()`是从local cache中获取到的数据