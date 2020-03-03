## 以kube-controller-manager中对replicaset的管理为例

1. `k8s.io\kubernetes\cmd\kube-controller-manager\app\controllermanager.go:386`中的`controllers["replicaset"] = startReplicaSetController`
2. 通过方法`replicaset.NewReplicaSetController(rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int)`创建一个`*ReplicaSetController`并执行其`Run(workers int, stopCh <-chan struct{})`函数
3. Run(workers int,stop <-chan struct{})中执行workers个worker方法
4. worker()中循环调用processNextWorkItem()
5. processNextWorkItem()中不断从ReplicaSetController的queue中取出item，交由syncHandler（值为syncReplicaSet）处理该item，如果处理成功，则直接queue中Forget(item)，否则，调用queue.AddRateLimited(item)限速放入queue中，等待再次处理
6. syncReplicaSet(key string)代码如下
```
func (rsc *ReplicaSetController) syncReplicaSet(key string) error {
	//...

    // 根据key获取namesapce和name
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}
    // 获取replicaset配置
	rs, err := rsc.rsLister.ReplicaSets(namespace).Get(name)
	if errors.IsNotFound(err) {
		klog.V(4).Infof("%v %v has been deleted", rsc.Kind, key)
		rsc.expectations.DeleteExpectations(key)
		return nil
	}
	if err != nil {
		return err
	}

    // 检查该key是否需要同步
	rsNeedsSync := rsc.expectations.SatisfiedExpectations(key)
	selector, err := metav1.LabelSelectorAsSelector(rs.Spec.Selector)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("Error converting pod selector to selector: %v", err))
		return nil
	}

	// list all pods to include the pods that don't match the rs`s selector
	// anymore but has the stale controller ref.
	// TODO: Do the List and Filter in a single pass, or use an index.
	allPods, err := rsc.podLister.Pods(rs.Namespace).List(labels.Everything())
	if err != nil {
		return err
	}
	// Ignore inactive pods.
	filteredPods := controller.FilterActivePods(allPods)

	// NOTE: filteredPods are pointing to objects from cache - if you need to
	// modify them, you need to copy it first.
	filteredPods, err = rsc.claimPods(rs, selector, filteredPods)
	if err != nil {
		return err
	}

	var manageReplicasErr error
	if rsNeedsSync && rs.DeletionTimestamp == nil {
        // 进行同步
		manageReplicasErr = rsc.manageReplicas(filteredPods, rs)
	}
	rs = rs.DeepCopy()
	newStatus := calculateStatus(rs, filteredPods, manageReplicasErr)

	// Always updates status as pods come up or die.
	updatedRS, err := updateReplicaSetStatus(rsc.kubeClient.AppsV1().ReplicaSets(rs.Namespace), rs, newStatus)
	if err != nil {
		// Multiple things could lead to this update failing. Requeuing the replica set ensures
		// Returning an error causes a requeue without forcing a hotloop
		return err
	}
	// Resync the ReplicaSet after MinReadySeconds as a last line of defense to guard against clock-skew.
	if manageReplicasErr == nil && updatedRS.Spec.MinReadySeconds > 0 &&
		updatedRS.Status.ReadyReplicas == *(updatedRS.Spec.Replicas) &&
		updatedRS.Status.AvailableReplicas != *(updatedRS.Spec.Replicas) {
		rsc.enqueueReplicaSetAfter(updatedRS, time.Duration(updatedRS.Spec.MinReadySeconds)*time.Second)
	}
	return manageReplicasErr
}
```