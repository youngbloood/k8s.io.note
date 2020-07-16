# Event

## 前言
- event的详细定义见k8s.io\api\core\v1\types.go
- event表示集群中对一些事件的报告

## 关于event的一些实现细节，此处以kube-controller-manager中的replicaset为例

> 1. 本章看完会event在EventBroadcaster中的数据流向
> 2. 了解kube-controller-manager中对replicaset的同步处理时发生了什么

### 1. 创建ReplicaSet实例的方法
[k8s.io/kubernetes/cmd/kube-controller-manager/app/apps.go:69]($GOPATH/k8s.io/kubernetes/cmd/kube-controller-manager/app/apps.go:69)
```
func startReplicaSetController(ctx ControllerContext) (http.Handler, bool, error) {
	if !ctx.AvailableResources[schema.GroupVersionResource{Group: "apps", Version: "v1", Resource: "replicasets"}] {
		return nil, false, nil
	}
	go replicaset.NewReplicaSetController(
		ctx.InformerFactory.Apps().V1().ReplicaSets(),
		ctx.InformerFactory.Core().V1().Pods(),
		ctx.ClientBuilder.ClientOrDie("replicaset-controller"),
		replicaset.BurstReplicas,
	).Run(int(ctx.ComponentConfig.ReplicaSetController.ConcurrentRSSyncs), ctx.Stop)
	return nil, true, nil
}
```
[k8s.io/kubernetes/pkg/controller/replicaset/replica_set.go:109]($GOPATH/k8s.io/kubernetes/pkg/controller/replicaset/replica_set.go:109)
```
// NewReplicaSetController configures a replica set controller with the specified event recorder
func NewReplicaSetController(rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int) *ReplicaSetController {
    // LABEL: 主干1（创建一个Broadcaster）
	eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartLogging(klog.Infof)
    // LABEL: 主干3（event的流出口）
	eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: kubeClient.CoreV1().Events("")})
	return NewBaseController(rsInformer, podInformer, kubeClient, burstReplicas,
		apps.SchemeGroupVersion.WithKind("ReplicaSet"),
		"replicaset_controller",
		"replicaset",
        // controller.RealPodControl将被赋值于ReplicaSetController.podControl
		controller.RealPodControl{
			KubeClient: kubeClient,
            // LABEL：主干2（event的流入口）
			Recorder:   eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "replicaset-controller"}),
		},
	)
}
```

### 2. `ReplicaSet.Run`
ReplicaSet.Run将运行Run()函数，处理ReplicaSet，追踪代码发现最后由ReplicaSet.syncHandler(key string)处理从local cache中拿到的数据（key是从local cache中来的数据，这里不了解的可以看看[informer_factory.md](./informer_factory.md)和[informer_indexer.md](./informer_indexer.md)的介绍）
该方法最终指向[k8s.io\kubernetes\pkg\controller\replicaset\replica_set.go:562]($GOPATH/k8s.io\kubernetes\pkg\controller\replicaset\replica_set.go:562)
```
// syncReplicaSet will sync the ReplicaSet with the given key if it has had its expectations fulfilled,
// meaning it did not expect to see any more of its pods created or deleted. This function is not meant to be
// invoked concurrently with the same key.
func (rsc *ReplicaSetController) syncReplicaSet(key string) error {
    // ...
    // 该replicaset资源是否需要同步
	rsNeedsSync := rsc.expectations.SatisfiedExpectations(key)
	
    // ...
	var manageReplicasErr error
	if rsNeedsSync && rs.DeletionTimestamp == nil {
        // 进行同步
		manageReplicasErr = rsc.manageReplicas(filteredPods, rs)
	}
	if manageReplicasErr == nil && updatedRS.Spec.MinReadySeconds > 0 &&
		updatedRS.Status.ReadyReplicas == *(updatedRS.Spec.Replicas) &&
		updatedRS.Status.AvailableReplicas != *(updatedRS.Spec.Replicas) {
        // 同步发生错误，在一定时间后重新加入queue中，等待下次同步
		rsc.enqueueReplicaSetAfter(updatedRS, time.Duration(updatedRS.Spec.MinReadySeconds)*time.Second)
	}
	return manageReplicasErr
}
```

ReplicaSet.manageReplicas(...)

```
// manageReplicas checks and updates replicas for the given ReplicaSet.
// Does NOT modify <filteredPods>.
// It will requeue the replica set in case of an error while creating/deleting pods.
func (rsc *ReplicaSetController) manageReplicas(filteredPods []*v1.Pod, rs *apps.ReplicaSet) error {
	// 实际pod与期望pod的差异值
    diff := len(filteredPods) - int(*(rs.Spec.Replicas))
	// ...
	if diff < 0 {
        // 实际pod数量<期望pod数量
        // ...
		successfulCreations, err := slowStartBatch(diff, controller.SlowStartInitialBatchSize, func() error {
            // 创建pod
			err := rsc.podControl.CreatePodsWithControllerRef(rs.Namespace, &rs.Spec.Template, rs, metav1.NewControllerRef(rs, rsc.GroupVersionKind))
            // ...
			return err
		})
        // ...
		return err
	} else if diff > 0 {
        // 实际pod数量>期望pod数量
		// ...
		podsToDelete := getPodsToDelete(filteredPods, diff)
        // ...
		for _, pod := range podsToDelete {
			go func(targetPod *v1.Pod) {
				defer wg.Done()
                // 删除pod
				if err := rsc.podControl.DeletePod(rs.Namespace, targetPod.Name, rs); err != nil {
                    // ...
				}
			}(pod)
		}
        // ...
	}
	return nil
}
```
创建/删除pod的操作均由podControl来完成，即第1步中NewReplicaSetController的controller.RealPodControl{...}对象完成该操作。
追踪controller.RealPodControl{}.CreatePodsWithControllerRef(...)即知道创建pod的本质调用了controller.RealPodControl.Recorder.Eventf({event})，即向Recorder中写入了一个事件。


### 3. 主干1
`record.NewBroadcaster()`
[k8s.io\client-go\tools\record\event.go:136]($GOPATH/k8s.io\client-go\tools\record\event.go:136)
```
// Creates a new event broadcaster.
func NewBroadcaster() EventBroadcaster {
	return &eventBroadcasterImpl{
        // LABEL: 分支1.1
		Broadcaster:   watch.NewBroadcaster(maxQueuedEvents, watch.DropIfChannelFull), // maxQueuedEvents = 1000
		sleepDuration: defaultSleepDuration, // 10*second
	}
}

type eventBroadcasterImpl struct {
	*watch.Broadcaster // 内嵌watch.Broadcaster
	sleepDuration time.Duration
	options       CorrelatorOptions
}

// 这个函数传入一个eventHandler以处理watcher监听到的event，而此处返回的watch.Interface不建议调用ResultChan()方法同eventHandler争抢事件
// 以防止eventHandler漏掉事件的处理，可调用Stop()停止该watcher，防止watcher泄露
func (e *eventBroadcasterImpl) StartEventWatcher(eventHandler func(*v1.Event)) watch.Interface {
    // 由于watch.Broadcaster是内嵌，因此调用的其实是watch.Broadcaster.Watch()方法：参考3.1节
	watcher := e.Watch()
	go func() {
		defer utilruntime.HandleCrash()
        // 监听watcher接受到的event
		for watchEvent := range watcher.ResultChan() {
			event, ok := watchEvent.Object.(*v1.Event)
			if !ok {
				// This is all local, so there's no reason this should
				// ever happen.
				continue
			}
            // 将event交由eventHandler进行处理
			eventHandler(event)
		}
	}()
	return watcher
}
```
`eventBroadcasterImpl`实现了`EventBroadcaster`接口
```
// EventBroadcaster knows how to receive events and send them to any EventSink, watcher, or log.
type EventBroadcaster interface {
	// StartEventWatcher starts sending events received from this EventBroadcaster to the given
	// event handler function. The return value can be ignored or used to stop recording, if
	// desired.
	StartEventWatcher(eventHandler func(*v1.Event)) watch.Interface

	// StartRecordingToSink starts sending events received from this EventBroadcaster to the given
	// sink. The return value can be ignored or used to stop recording, if desired.
	StartRecordingToSink(sink EventSink) watch.Interface

	// StartLogging starts sending events received from this EventBroadcaster to the given logging
	// function. The return value can be ignored or used to stop recording, if desired.
	StartLogging(logf func(format string, args ...interface{})) watch.Interface

	// NewRecorder returns an EventRecorder that can be used to send events to this EventBroadcaster
	// with the event source set to the given event source.
	NewRecorder(scheme *runtime.Scheme, source v1.EventSource) EventRecorder

	// Shutdown shuts down the broadcaster
	Shutdown()
}
```



#### 3.1. 分支1.1 watch.Broadcaster

```
// Broadcaster distributes event notifications among any number of watchers. Every event
// is delivered to every watcher.
type Broadcaster struct {
	// TODO: see if this lock is needed now that new watchers go through
	// the incoming channel.
	lock sync.Mutex

	watchers     map[int64]*broadcasterWatcher
	nextWatcher  int64
	distributing sync.WaitGroup

	incoming chan Event

	// How large to make watcher's channel.
	watchQueueLength int
	// If one of the watch channels is full, don't wait for it to become empty.
	// Instead just deliver it to the watchers that do have space in their
	// channels and move on to the next event.
	// It's more fair to do this on a per-watcher basis than to do it on the
	// "incoming" channel, which would allow one slow watcher to prevent all
	// other watchers from getting new events.
	fullChannelBehavior FullChannelBehavior
}

// new一个Broadcaster
func NewBroadcaster(queueLength int, fullChannelBehavior FullChannelBehavior) *Broadcaster {
	m := &Broadcaster{
		watchers:            map[int64]*broadcasterWatcher{},
		incoming:            make(chan Event, incomingQueueLength),
		watchQueueLength:    queueLength,
		fullChannelBehavior: fullChannelBehavior,
	}
	m.distributing.Add(1)
    // 进行循环
	go m.loop()
	return m
}


// loop receives from m.incoming and distributes to all watchers.
func (m *Broadcaster) loop() {
	// 循环从incoming中取出event，然后distribute(event)
	for event := range m.incoming {
		if event.Type == internalRunFunctionMarker {
			event.Object.(functionFakeRuntimeObject)()
			continue
		}
		m.distribute(event)
	}
	m.closeAll()
	m.distributing.Done()
}

// 分发event
func (m *Broadcaster) distribute(event Event) {
	m.lock.Lock()
	defer m.lock.Unlock()
	if m.fullChannelBehavior == DropIfChannelFull {
		for _, w := range m.watchers {
			select {
            // 将event分发到每个watcher的result中
			case w.result <- event:
			case <-w.stopped:
			default: // Don't block if the event can't be queued.
			}
		}
	} else {
		for _, w := range m.watchers {
			select {
			case w.result <- event:
			case <-w.stopped:
			}
		}
	}
}

// 新生成一个watcher，加入到Broadcaster的map中，该watcher会监听到之后的event，之前的event将监听不到
// 返回的watcher就是最后event的流出口
func (m *Broadcaster) Watch() Interface {
	var w *broadcasterWatcher
	m.blockQueue(func() {
		m.lock.Lock()
		defer m.lock.Unlock()
		id := m.nextWatcher
		m.nextWatcher++
		w = &broadcasterWatcher{
            // chan Event的长度由m.watchQueueLength定
			result:  make(chan Event, m.watchQueueLength),
			stopped: make(chan struct{}),
			id:      id,
			m:       m,
		}
		m.watchers[id] = w
	})
	return w
}

// 组装事件，并发送到Broadcaster的incoming队列中
func (m *Broadcaster) Action(action EventType, obj runtime.Object) {
	m.incoming <- Event{action, obj}
}

// 其他一些Broadcaster的Shutdown()方法不再一一详解
```
`broadcasterWatcher`的结构体且实现了watch.Interface接口
```
type broadcasterWatcher struct {
	result  chan Event
	stopped chan struct{}
	stop    sync.Once
	id      int64
	m       *Broadcaster
}

// ResultChan returns a channel to use for waiting on events.
func (mw *broadcasterWatcher) ResultChan() <-chan Event {
	return mw.result
}

// Stop stops watching and removes mw from its list.
func (mw *broadcasterWatcher) Stop() {
	mw.stop.Do(func() {
		close(mw.stopped)
		mw.m.stopWatching(mw.id)
	})
}
```



### 4. 主干2
```
// NewRecorder returns an EventRecorder that records events with the given event source.
func (e *eventBroadcasterImpl) NewRecorder(scheme *runtime.Scheme, source v1.EventSource) EventRecorder {
    // *recorderImpl实现了EventRecorder接口，也就是这个对象最终被赋值给ReplicaSet.podControl
    // 写入的event最终被写入到recorderImpl中，再由其通知watcher
	return &recorderImpl{scheme, source, e.Broadcaster, clock.RealClock{}}
}
```
recorderImpl的一些实习方法
```
func (recorder *recorderImpl) Eventf(object runtime.Object, eventtype, reason, messageFmt string, args ...interface{}) {
	recorder.Event(object, eventtype, reason, fmt.Sprintf(messageFmt, args...))
}

func (recorder *recorderImpl) Event(object runtime.Object, eventtype, reason, message string) {
	recorder.generateEvent(object, nil, metav1.Now(), eventtype, reason, message)
}

func (recorder *recorderImpl) generateEvent(object runtime.Object, annotations map[string]string, timestamp metav1.Time, eventtype, reason, message string) {
	// ...
	event := recorder.makeEvent(ref, annotations, eventtype, reason, message)
	event.Source = recorder.source
	go func() {
		// NOTE: events should be a non-blocking operation
		defer utilruntime.HandleCrash()
        // 组装事件，由Action放入到incoming队列中：详情参考3.1节
		recorder.Action(watch.Added, event)
	}()
}

```

### 5. 主干3：event的流出口

```
// StartRecordingToSink starts sending events received from the specified eventBroadcaster to the given sink.
// The return value can be ignored or used to stop recording, if desired.
// TODO: make me an object with parameterizable queue length and retry interval
func (e *eventBroadcasterImpl) StartRecordingToSink(sink EventSink) watch.Interface {
	eventCorrelator := NewEventCorrelatorWithOptions(e.options)
    // StartEventWatcher会生成一个watcher，并监听event，交由传入的eventHandler进行处理（交由recordToSink进行处理）；参考第3节
	return e.StartEventWatcher(
		func(event *v1.Event) {
            // 将event纪录到sink中
			recordToSink(sink, event, eventCorrelator, e.sleepDuration)
		})
}

func recordToSink(sink EventSink, event *v1.Event, eventCorrelator *EventCorrelator, sleepDuration time.Duration) {
	// ...
	result, err := eventCorrelator.EventCorrelate(event)
	// ...
	for {
		if recordEvent(sink, result.Event, result.Patch, result.Event.Count > 1, eventCorrelator) {
			break
		}
		// ...
	}
}

// recordEvent attempts to write event to a sink. It returns true if the event
// was successfully recorded or discarded, false if it should be retried.
// If updateExistingEvent is false, it creates a new event, otherwise it updates
// existing event.
func recordEvent(sink EventSink, event *v1.Event, patch []byte, updateExistingEvent bool, eventCorrelator *EventCorrelator) bool {
	// ...
	if updateExistingEvent {
        // update 存在的event
		newEvent, err = sink.Patch(event, patch)
	}
	// Update can fail because the event may have been removed and it no longer exists.
	if !updateExistingEvent || (updateExistingEvent && util.IsKeyNotFoundError(err)) {
		// Making sure that ResourceVersion is empty on creation
		event.ResourceVersion = ""
        // 创建新的event
		newEvent, err = sink.Create(event)
	}
	// ...
	return false
}
```

第1步中`NewReplicaSetController`里的`eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: kubeClient.CoreV1().Events("")})`,`v1core.EventSinkImpl`实现了`Sink`接口
```
type EventSinkImpl struct {
	Interface EventInterface
}

func (e *EventSinkImpl) Create(event *v1.Event) (*v1.Event, error) {
	return e.Interface.CreateWithEventNamespace(event)
}

func (e *EventSinkImpl) Update(event *v1.Event) (*v1.Event, error) {
	return e.Interface.UpdateWithEventNamespace(event)
}

func (e *EventSinkImpl) Patch(event *v1.Event, data []byte) (*v1.Event, error) {
	return e.Interface.PatchWithEventNamespace(event, data)
}
```
调用sink.Patch()/sink.Create()/sink.Update()的本质是调用了`EventSinkImpl.Interface`里的各种对应方法，因此最终均交由`kubeClient.CoreV1().Events("")`来实现了对event的处理；kubeClient正是连接到master上的一个客户端（此处的kubeClient的生成可以自己尝试看看之前的源代码，此处不再做赘述）。



## 总结

- Broadcaster是实现了一个event的producer/consumer模式。
- producer调用Action()方法产生event，最后由consumer调用Watch()消费掉event。
- 当某个consumer不想再监听event时，可调用Stop()停止掉该consumer的监听。
- 当producer不想再生产event时，可调用Broadcaster的Shutdown()方法停止。




