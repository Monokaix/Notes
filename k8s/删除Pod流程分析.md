使用kubectl删除pod时，通常会看到pod首先处于Terminating状态，正常情况下过段时间pod就会被彻底删除，而在某些异常场景下pod则会一直处于Terminating状态，直到特定条件被满足。这涉及到kubernetes对pod等资源的优雅删除，即不会立即从etcd中删除，而是等待一个`gracePeriod`让pod中的container在被彻底kill前执行`preStop`或者业务自定义的处理，避免容器的突然中止导致业务出错。那么kubernetes内部到底是如何处理pod删除事件的，pod结束Terminating状态的特定条件又有哪些？本文将从源码层面予以解释。

# kubectl行为

`kubectl delete`有两个参数`--force`和`--grace-period`，默认分别为false和-1，前者为true表示直接从etcd删除不进行优雅删除，后者可以指定优雅删除的宽限期，只有`--force`为true该值才可以设置为0，表示强制删除而不管pod残留的资源有没有被清理干净。默认使用`kubectl`如果未显式指定两个参数则忽略`grace-period`，然后转化为一个DeleteOptions作为参数传递给kube-apiserver，`GracePeriodSeconds`即为设置的宽限期。

```go
type DeleteOptions struct {
    ...
	GracePeriodSeconds *int64 
	...
}
```

# kube-apiserver行为

kube-apiserver使用go-restful作为server端web框架，因此要想清楚kube-apiserver接收到删除请求的具体处理流程，就应该找kube-apiserver对`DELETE`动作的处理，即删除操作对应的具体的http handler。

[staging/src/k8s.io/apiserver/pkg/endpoints/installer.go](https://github.com/kubernetes/kubernetes/blob/99e36a93b2170292a4d7b675470cf64ce4fb4b56/staging/src/k8s.io/apiserver/pkg/endpoints/installer.go#L190)文件会对api资源的handler进行安装，绑定url与其对应的handler，其中`registerResourceHandlers`函数注册了`DELETE`对应的handler，部分代码如下

```go
		case "DELETE": // Delete a resource.
			article := GetArticleForNoun(kind, " ")
			doc := "delete" + article + kind
			if isSubresource {
				doc = "delete " + subresource + " of" + article + kind
			}
			deleteReturnType := versionedStatus
			if deleteReturnsDeletedObject {
				deleteReturnType = producedObject
			}
			handler := metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, restfulDeleteResource(gracefulDeleter, isGracefulDeleter, reqScope, admit))
			if enableWarningHeaders {
				handler = utilwarning.AddWarningsHandler(handler, warnings)
			}
```

对应的handler即为`restfulDeleteResource`，最终该handler调用了[staging/src/k8s.io/apiserver/pkg/registry/generic/registry/store.go](https://github.com/kubernetes/kubernetes/blob/99e36a93b2170292a4d7b675470cf64ce4fb4b56/staging/src/k8s.io/apiserver/pkg/registry/generic/registry/store.go#L1002)的DELETE方法，该方法会从请求参数中构造`DeleteOptions`参数，然后调用[staging/src/k8s.io/apiserver/pkg/registry/rest/delete.go](https://github.com/kubernetes/kubernetes/blob/99e36a93b2170292a4d7b675470cf64ce4fb4b56/staging/src/k8s.io/apiserver/pkg/registry/rest/delete.go#L75)的`rest.BeforeDelete`判断是否是优雅删除，主要逻辑

```go
func BeforeDelete(strategy RESTDeleteStrategy, ctx context.Context, obj runtime.Object, options *metav1.DeleteOptions) (graceful, gracefulPending bool, err error) {
    ...
	gracefulStrategy, ok := strategy.(RESTGracefulDeleteStrategy)
	if !ok {
		// If we're not deleting gracefully there's no point in updating Generation, as we won't update
		// the obcject before deleting it.
		return false, false, nil
	}
    ...
    // 调用不同资源的设置的优雅删除策略设置优雅删除等参数
	if !gracefulStrategy.CheckGracefulDelete(ctx, obj, options) {
		return false, false, nil
	}
    // 重点：设置不同资源的DeletionTimestamp和DeletionGracePeriodSeconds，标记为删除状态
	now := metav1.NewTime(metav1.Now().Add(time.Second * time.Duration(*options.GracePeriodSeconds)))
	objectMeta.SetDeletionTimestamp(&now)
	objectMeta.SetDeletionGracePeriodSeconds(options.GracePeriodSeconds)
```

该函数中首先会有另外一个回调来针对不同的资源设置不同的优雅删除策略，接口定义如下

```go
type RESTGracefulDeleteStrategy interface {
	// CheckGracefulDelete should return true if the object can be gracefully deleted and set
	// any default values on the DeleteOptions.
	CheckGracefulDelete(ctx context.Context, obj runtime.Object, options *metav1.DeleteOptions) bool
}
```

这里关注pod策略的具体实现，如果请求参数里设置了`GracePeriodSeconds`，则设置为用户指定的值，如果没有设置就使用pod的spec中`TerminationGracePeriodSeconds`字段设置的值，这个值默认为30s，是在pod创建的时候赋的，这也就解释了为什么使用`kubectl`删除时并未指定`gracePeriod`参数而实际上pod的`ObjectMeta`中的`DeletionGracePeriodSeconds`字段却被设置为了30s，这一现象可以通过`kubectl get po nginx-deployment-xxx -w -oyaml`观察pod删除过程中pod的`DeletionGracePeriodSeconds`值的变化来证实。

```go
func (podStrategy) CheckGracefulDelete(ctx context.Context, obj runtime.Object, options *metav1.DeleteOptions) bool {
    ...
	// 使用用户指定的优雅删除时间
	if options.GracePeriodSeconds != nil {
		period = *options.GracePeriodSeconds
	} else {
        // 使用pod设置的默认的TerminationGracePeriodSeconds值，默认为30s
		if pod.Spec.TerminationGracePeriodSeconds != nil {
			period = *pod.Spec.TerminationGracePeriodSeconds
		}
	}
    ...
	// ensure the options and the pod are in sync
	options.GracePeriodSeconds = &period
	return true
}
```

接下来`BeforeDelete`的逻辑就是设置pod的`DeletionTimestamp`和`DeletionGracePeriodSeconds`字段值，然后更新后数据库，因此可以看到有了优雅删除，pod的删除其实被分成了两步，第一步是kube-apiserver仅仅更新了pod的两个字段，将pod标记为删除状态，其余的几乎什么也没干，因为kube-apisever并不感知pod中的container是否已经运行完毕可以kill，也不应该由kube-apiserver来负责管理，因此第二步就由节点上的kubelet来进行了。

顺便提一句，我们经常看到pod处于所谓的Terminating状态，其实这一状态在etcd中pod的具体字段中并不存在，而是kubectl get时可以在请求Header中设置Accept的具体值，当设置的value中有`as=table`时，kube-apiserver返回的是`Table`类型，即人类可读的格式，而这个Terminating状态就是kube-apiserver设置的一种转化形式，当资源的`DeletionTimestamp`不为空时，将`Table`中的Status设置为Terminating。

# kubelet行为

kubelet通过informer监听本节点pod事件，kubelet启动时，syncLoop会处理所有的pod更新事件，然后根据不同的时间类型dispatch分发到podWorker进行处理，在每次处理中会将podStatus的变化放进一个channel，kubelet启动了一个异步的协程进专门进行podStatus变化的处理，消费该channel处理podStatus的变化，这部分逻辑由status_manager进行处理，status_manager在处理podStatus时调用syncPod方法进行处理，主要逻辑如下

```go
func (m *manager) syncPod(uid types.UID, status versionedPodStatus) {
    ...
    // 判断是否可以删除从etcd中删除pod的逻辑
	if m.canBeDeleted(pod, status.status) {
		deleteOptions := metav1.DeleteOptions{
			GracePeriodSeconds: new(int64),
			// Use the pod UID as the precondition for deletion to prevent deleting a
			// newly created pod with the same name and namespace.
			Preconditions: metav1.NewUIDPreconditions(string(pod.UID)),
		}
        // 设置GracePeriodSeconds为0强制删除pod
		err = m.kubeClient.CoreV1().Pods(pod.Namespace).Delete(context.TODO(), pod.Name, deleteOptions)
		if err != nil {
			klog.InfoS("Failed to delete status for pod", "pod", klog.KObj(pod), "err", err)
			return
		}
		klog.V(3).InfoS("Pod fully terminated and removed from etcd", "pod", klog.KObj(pod))
		m.deletePodStatus(uid)
	}
}
```

主要的判断为canBeDeleted函数，其中就包含了对前文说的pod是否满足特定条件的判断，这些条件包括

- pod中的container不再running
- 从runtime侧获取的container被清理掉
- pod挂载的volume被清理掉
- container的cgroup被清理掉

因此只有这些条件都被满足时，kuelet才会发送一个GracePeriodSeconds为0的deleteOptions给kube-apiserver，kube-apiserver此时才会真正从etcd中删除该pod，然后pod消失。

pod处于terminating原因分析

了解了pod结束Terminating的条件，再分析pod处于Terminating的原因就比较简单了。通常有以下原因

- kubelet与runtime的通信有问题，导致kuelet获取不到container状态或者更新podStatus状态失败
- container由于某些原因无法清理，比如容器进程变成D状态，无法接受信号，导致container残留。
- pod的volume卸载出错，比如csi场景下与csi通信出现问题，或者pod的volume出现重复挂载，而调用csi接口只卸载了一次，且在挂载时没有校验重复挂载。kubernetes1.20版本之后，在对csi类型的卷进行挂载操作时，把校验重复挂载的逻辑去掉了，社区认为校验重复挂载的逻辑应该由csi插件来做，因此如果csi插件没有检验重复挂载，则一旦kubelet重启进行了重复挂载，删除pod时由于只进行了一次卸载 ，导致pod挂载的卷仍然残留，从而阻塞pod删除，相关[issue](https://github.com/kubernetes/kubernetes/pull/88759)见链接。

# 总结

对pod的删除操作并不是我们想象的那样一次性删除，优雅删除的实现需要`kubectl/kube-apiserver/kubelet`的协作完成，通过对kubelet最终删除pod的流程进行分析，可以帮助我们快速分析到阻塞pod删除的原因，同时也学习到了kubernetes的优秀设计。