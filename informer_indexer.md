# cache.SharedIndexInformer的深入了解

## SharedInformerFactory接口
位于k8s.io\client-go\tools\cache\shared_informer.go
```
// SharedInformerFactory provides shared informers for resources in all known
// API group versions.
type SharedInformerFactory interface {
	// 此处内嵌internalinterfaces.SharedInformerFactory
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
internalinterfaces.SharedInformerFactory定义
```
// SharedInformerFactory a small interface to allow for adding an informer without an import cycle
type SharedInformerFactory interface {
	Start(stopCh <-chan struct{})
	InformerFor(obj runtime.Object, newFunc NewInformerFunc) cache.SharedIndexInformer
}
```

实现SharedInformerFactory接口的结构体`sharedInformerFactory`
```
type sharedInformerFactory struct {
	
	// ...

	// 其中cache.SharedIndexInformer是本篇主要讲的内容，将会涉及到其数据流向
	// 此map的值是通过InformerFor()函数将各种资源的informer注入进来的
	informers map[reflect.Type]cache.SharedIndexInformer
	
	// 记录开启了的informer
	startedInformers map[reflect.Type]bool
}
```

## cache.SharedIndexInformer的定义
```
// SharedIndexInformer provides add and get Indexers ability based on SharedInformer.
type SharedIndexInformer interface {
	SharedInformer
	// AddIndexers add indexers to the informer before it starts.
	AddIndexers(indexers Indexers) error
	GetIndexer() Indexer
}
```

SharedInformer的定义
```
type SharedInformer interface {
	// AddEventHandler adds an event handler to the shared informer using the shared informer's resync
	// period.  Events to a single handler are delivered sequentially, but there is no coordination
	// between different handlers.
	AddEventHandler(handler ResourceEventHandler)
	// AddEventHandlerWithResyncPeriod adds an event handler to the
	// shared informer using the specified resync period.  The resync
	// operation consists of delivering to the handler a create
	// notification for every object in the informer's local cache; it
	// does not add any interactions with the authoritative storage.
	AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration)
	// GetStore returns the informer's local cache as a Store.
	GetStore() Store
	// GetController gives back a synthetic interface that "votes" to start the informer
	GetController() Controller
	// Run starts and runs the shared informer, returning after it stops.
	// The informer will be stopped when stopCh is closed.
	Run(stopCh <-chan struct{})
	// HasSynced returns true if the shared informer's store has been
	// informed by at least one full LIST of the authoritative state
	// of the informer's object collection.  This is unrelated to "resync".
	HasSynced() bool
	// LastSyncResourceVersion is the resource version observed when last synced with the underlying
	// store. The value returned is not synchronized with access to the underlying store and is not
	// thread-safe.
	LastSyncResourceVersion() string
}
```

实现SharedIndexInformer接口的结构体`sharedIndexInformer`
```
type sharedIndexInformer struct {
	// 本地缓存
	indexer    Indexer
	controller Controller
	processor             *sharedProcessor
	// 用于获取远程的资源
	listerWatcher ListerWatcher
	
	// 省略部分字段
}
```

cache包提供一个构造函数：`NewSharedIndexInformer(lw ListerWatcher, objType runtime.Object, defaultEventHandlerResyncPeriod time.Duration, indexers Indexers)`用来构造一个cache.SharedIndexInformer对象，其中：

  - `lw：ListerWatcher`:是数据的来源处，通过http从数据源处获取数据
   
  - `objType`:是资源类型

  - `defaultEventHandlerResyncPeriod`:默认事件处理器重同步周期
   
  - `indexers`:本地的indexer，用于构造一个 Indexer interface


## cache.SharedIndexInformer的数据来源
sharedIndexInformer.Run(stopCh <-chan struct{})函数
```
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	// new一个fifo队列
	fifo := NewDeltaFIFO(MetaNamespaceKeyFunc, s.indexer)

	cfg := &Config{
		Queue:            fifo,
		ListerWatcher:    s.listerWatcher,
		ObjectType:       s.objectType,
		FullResyncPeriod: s.resyncCheckPeriod,
		RetryOnError:     false,
		ShouldResync:     s.processor.shouldResync,
		// 将sharedIndexInformer.HandleDeltas注入Config中；此方法用来消耗掉queue中接收到的数据
		Process: s.HandleDeltas,
	}

	func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()
		// new一个controller
		s.controller = New(cfg)
		s.controller.(*controller).clock = s.clock
		s.started = true
	}()

	// Separate stop channel because Processor should be stopped strictly after controller
	processorStopCh := make(chan struct{})
	var wg wait.Group
	defer wg.Wait()              // Wait for Processor to stop
	defer close(processorStopCh) // Tell Processor to stop
	wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)
	wg.StartWithChannel(processorStopCh, s.processor.run)

	defer func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()
		s.stopped = true // Don't want any new listeners
	}()

	// 执行controller的Run()函数
	s.controller.Run(stopCh)
}
```
跳转到`New(cfg)`，位于`k8s.io\client-go\tools\cache\controller.go`中
```
// New makes a new Controller from the given Config.
func New(c *Config) Controller {
	ctlr := &controller{
		config: *c,
		clock:  &clock.RealClock{},
	}
	return ctlr
}
```
Controller接口
```
// Controller is a generic controller framework.
type Controller interface {
	Run(stopCh <-chan struct{})
	HasSynced() bool
	LastSyncResourceVersion() string
}
```

controller.Run()函数
```
func (c *controller) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	go func() {
		<-stopCh
		c.config.Queue.Close()
	}()
	// 创建一个reflector
	r := NewReflector(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		c.config.FullResyncPeriod,
	)
	r.ShouldResync = c.config.ShouldResync
	r.clock = c.clock

	c.reflectorMutex.Lock()
	c.reflector = r
	c.reflectorMutex.Unlock()

	var wg wait.Group
	defer wg.Wait()
	// reflector.Run()函数，此处就是数据的流入口
	wg.StartWithChannel(stopCh, r.Run)
	// 调用controller.processLoop()函数，此处是数据的流出口
	wait.Until(c.processLoop, time.Second, stopCh)
}
```

reflector.Run()函数
```
// Run starts a watch and handles watch events. Will restart the watch if it is closed.
// Run will exit when stopCh is closed.
func (r *Reflector) Run(stopCh <-chan struct{}) {
	klog.V(3).Infof("Starting reflector %v (%s) from %s", r.expectedTypeName, r.resyncPeriod, r.name)
	wait.Until(func() {
		// 此方法太长，不贴源码，功能：不断从`listerwatcher`中获取数据，存入store中，这个store实质为一个queue，再创建controller的时候创建的
		if err := r.ListAndWatch(stopCh); err != nil {
			utilruntime.HandleError(err)
		}
	}, r.period, stopCh)
}

```

## cache.SharedIndexInformer的数据去向
controller.processLoop()函数
```
func (c *controller) processLoop() {
	for {
		// 将c.config.Process转化为PopProcessFunc方法，不断消耗掉queue中Pop出的数据
		obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
		// ...
	}
}
```

有上面的代码可知，c.config.Process本质就是`sharedIndexInformer.HandleDeltas`方法

sharedIndexInformer.HandleDeltas()函数

```

func (s *sharedIndexInformer) HandleDeltas(obj interface{}) error {
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	// from oldest to newest
	for _, d := range obj.(Deltas) {
		switch d.Type {
		case Sync, Added, Updated:
			isSync := d.Type == Sync
			s.cacheMutationDetector.AddObject(d.Object)
			if old, exists, err := s.indexer.Get(d.Object); err == nil && exists {
				if err := s.indexer.Update(d.Object); err != nil {
					return err
				}
				// 交由distribute处理
				s.processor.distribute(updateNotification{oldObj: old, newObj: d.Object}, isSync)
			} else {
				if err := s.indexer.Add(d.Object); err != nil {
					return err
				}
				// 交由distribute处理
				s.processor.distribute(addNotification{newObj: d.Object}, isSync)
			}
		case Deleted:
			if err := s.indexer.Delete(d.Object); err != nil {
				return err
			}
			// 交由distribute处理
			s.processor.distribute(deleteNotification{oldObj: d.Object}, false)
		}
	}
	return nil
}
```

processor.distribute()函数

```
func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()

	if sync {
		// 分发到其下的listener中
		for _, listener := range p.syncingListeners {
			listener.add(obj)
		}
	} else {
		// 分发到其下的listener中
		for _, listener := range p.listeners {
			listener.add(obj)
		}
	}
}
```

由processor.pop()函数将上步的obj取出来放入nextCh中
```
func (p *processorListener) pop() {
	defer utilruntime.HandleCrash()
	defer close(p.nextCh) // Tell .run() to stop

	var nextCh chan<- interface{}
	var notification interface{}
	for {
		select {
			// 写入nextCh
		case nextCh <- notification:
			// Notification dispatched
			var ok bool
			notification, ok = p.pendingNotifications.ReadOne()
			if !ok { // Nothing to pop
				nextCh = nil // Disable this select case
			}
		case notificationToAdd, ok := <-p.addCh:
			if !ok {
				return
			}
			if notification == nil { // No notification to pop (and pendingNotifications is empty)
				// 跳过放入pending中，直接放入nextCh中
				notification = notificationToAdd
				nextCh = p.nextCh
			} else { 
				// 放入pending中，缓冲
				p.pendingNotifications.WriteOne(notificationToAdd)
			}
		}
	}
}
```

最后，由processor.run()函数从nextCh或pending中取出来
```
func (p *processorListener) run() {
	// this call blocks until the channel is closed.  When a panic happens during the notification
	// we will catch it, **the offending item will be skipped!**, and after a short delay (one second)
	// the next notification will be attempted.  This is usually better than the alternative of never
	// delivering again.
	stopCh := make(chan struct{})
	wait.Until(func() {
		// this gives us a few quick retries before a long pause and then a few more quick retries
		err := wait.ExponentialBackoff(retry.DefaultRetry, func() (bool, error) {
			// 从nextCh中取出，由handler进行下一步处理
			for next := range p.nextCh {
				switch notification := next.(type) {
				case updateNotification:
					p.handler.OnUpdate(notification.oldObj, notification.newObj)
				case addNotification:
					p.handler.OnAdd(notification.newObj)
				case deleteNotification:
					p.handler.OnDelete(notification.oldObj)
				default:
					utilruntime.HandleError(fmt.Errorf("unrecognized notification: %T", next))
				}
			}
			// the only way to get here is if the p.nextCh is empty and closed
			return true, nil
		})

		// the only way to get here is if the p.nextCh is empty and closed
		if err == nil {
			close(stopCh)
		}
	}, 1*time.Minute, stopCh)
}
```

`cache.SharedIndexInformer.AddEventHandler(handler ResourceEventHandler)`会注册handler在`SharedInformer`中，该handler就是上一步中的handler接收器，专门处理`nextCh`中收到的事件

最后，obj被注册的ResourceEventHandler的方法消耗掉。


# 总结
- 调用NewSharedIndexInformer(lw ListerWatcher,...)方法new一个`SharedIndexInformer`时，lw就是该SharedIndexInformer监听数据的来源。
- SharedIndexInformer.AddEventHandler(handler ResourceEventHandler)方法会注册一个`ResourceEventHandler`，这个handler最终会处理掉从lw中缓存下来的所有数据。




## 附：k8s.io/client-go/informers/<resource>/<version>下的Informer的一些其他细节

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

- 此类resource下的informer是围绕`cache.SharedIndexInfomer`和`internalinterfaces.SharedInformerFactory`两个接口进行设计的。
- 在调用`Informer()`时，会直接从f.factory的缓存中取出`cache.SharedIndexInformer`，如果f.factory中没有，则调用`f.defaultInformer`初始化一个`cache.SharedIndexInfomer`并返回。

- 在调用`Lister()`时，会调用`f.Informer().GetIndexer()`，该函数返回的是local cache的数据。所以`Lister()`是从local cache中获取到的数据。

