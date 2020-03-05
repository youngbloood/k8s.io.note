## 关于event的一些实现细节，此处以kube-controller-manage中的replicaset为例


1. 接着[kube-controller-manager-replicaset.md](./kube-controller-manager-replicaset.md)中的`ReplicaSetController.syncReplicaSet(key string)`方法
[k8s.io\kubernetes\pkg\controller\replicaset\replica_set.go:562]($GOPATH/k8s.io\kubernetes\pkg\controller\replicaset\replica_set.go:562)
```
// syncReplicaSet will sync the ReplicaSet with the given key if it has had its expectations fulfilled,
// meaning it did not expect to see any more of its pods created or deleted. This function is not meant to be
// invoked concurrently with the same key.
func (rsc *ReplicaSetController) syncReplicaSet(key string) error {

    // ...

	rsNeedsSync := rsc.expectations.SatisfiedExpectations(key)
	
    // ...

	allPods, err := rsc.podLister.Pods(rs.Namespace).List(labels.Everything())
	if err != nil {
		return err
	}
	// Ignore inactive pods.
	filteredPods := controller.FilterActivePods(allPods)

	// ...

	var manageReplicasErr error
	if rsNeedsSync && rs.DeletionTimestamp == nil {
        // replicaset需要同步
		manageReplicasErr = rsc.manageReplicas(filteredPods, rs)
	}
    
    // ...

	return manageReplicasErr
}
```

2. `ReplicaSetController.manageReplicas(filteredPods []*v1.Pod, rs *apps.ReplicaSet) error`方法的定义
[k8s.io\kubernetes\pkg\controller\replicaset\replica_set.go:459]($GOPATH/k8s.io\kubernetes\pkg\controller\replicaset\replica_set.go:459)
```
    // ...
    // 通过kubeClient创建pod，直接请求master的接口进行create(pod)
	newPod, err := r.KubeClient.CoreV1().Pods(namespace).Create(pod)
	if err != nil {
		r.Recorder.Eventf(object, v1.EventTypeWarning, FailedCreatePodReason, "Error creating: %v", err)
		return err
	}
	
    // ...
    // 在master上创建成功，则在Recorder中写入创建事件
	r.Recorder.Eventf(object, v1.EventTypeNormal, SuccessfulCreatePodReason, "Created pod: %v", newPod.Name)
	return nil
```

3. `ReplicaSetController.Recorder`的初始化
[k8s.io\kubernetes\pkg\controller\replicaset\replica_set.go:108]($GOPATH/k8s.io\kubernetes\pkg\controller\replicaset\replica_set.go:108)
```
// NewReplicaSetController configures a replica set controller with the specified event recorder
func NewReplicaSetController(rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int) *ReplicaSetController {
    // eventBroadcaster看下一步
	eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartLogging(klog.Infof)
	eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: kubeClient.CoreV1().Events("")})
	return NewBaseController(rsInformer, podInformer, kubeClient, burstReplicas,
		apps.SchemeGroupVersion.WithKind("ReplicaSet"),
		"replicaset_controller",
		"replicaset",
        // 此对象被赋值于Recorder
		controller.RealPodControl{
			KubeClient: kubeClient,
			Recorder:   eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "replicaset-controller"}),
		},
	)
}

```

4. `Broadcaster`的一些细节
```
```