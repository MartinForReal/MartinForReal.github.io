kubernetes调度器原理和应用
===


[![hackmd-github-sync-badge](https://hackmd.io/Rxn1mH6FTsiXu9lBV9o32w/badge)](https://hackmd.io/Rxn1mH6FTsiXu9lBV9o32w)

目录：

1. [调度的基本知识](#基本知识)
    1. [调度器的定义](#调度器的定义)
    2. [调度器的考量标准](#调度器的考量标准) 
    3. [锁对调度器设计的影响](#锁对调度器设计的影响)
2. [Kubernetes中的调度原理](#Kubernetes中的调度原理)
    1. [调度器的设计](#调度器的设计)
    2. [调度器的实现](#调度器的实现)
    3. [调度器发展方向](#调度器发展方向)
3. [Kubernetes调度器的不足和改进](#Kubernetes调度器的不足和改进)
    1. [典型的几个问题和思路](#典型的几个问题和思路)
    2. [Kubernetes调度器的定制扩展](#Kubernetes调度器的定制扩展)
4. [其他调度系统的设计](#其他调度系统的设计)
5. [企业场景应用的思考](#企业场景应用的思考)


云环境中或者计算仓库级别（将整个数据中心当做单个计算池）的集群管理系统通常会给出工作负载的规范，并使用调度器将使用规范描述的工作负载放置到集群不同位置。好的调度器可以让集群的工作处理效率更高，资源利用率更高，更加节省能源。但设计大规模共享集群的调度器并不是一件容易的事情。调度器不仅要了解集群资源的使用情况和分布，还要兼顾分配速度和任务执行速度。

本文主要从原理设计和代码层面介绍Kubernetes的调度器和社区对调度器的补充，以及与业界常用调度器的设计进行比较。在文章开始，我们先概括的谈一下调度器的来龙去脉，从而能够准确的分析什么样的调度器适合自己的场景。

## 基本知识

### 调度器的定义

[通用调度](https://en.wikipedia.org/wiki/Scheduling_(computing))的定义是指使用某种方法将某种工作分配到特定资源以完成相关工作的方法，这些工作可以是虚拟计算元素，如线程、进程或数据流，而特定资源一般是指处理器、网络、磁盘等资源。调度器是完成这些调度行为的具体实现，目的是尽可能高效利用资源，允许用户共享系统资源，降低等待时间，提高吞吐率等等。

而我们这里讨论的调度器是特指大规模集群下的调度，比较典型的有[Mesos](https://amplab.cs.berkeley.edu/wp-content/uploads/2011/06/Mesos-A-Platform-for-Fine-Grained-Resource-Sharing-in-the-Data-Center.pdf)/[Yarn](https://www.cse.ust.hk/~weiwa/teaching/Fall15-COMP6611B/reading_list/YARN.pdf)(Apache)、[Borg](https://ai.google/research/pubs/pub43438)/[Omega](https://ai.google/research/pubs/pub41684)(Google)、[Quincy](https://www.microsoft.com/en-us/research/publication/quincy-fair-scheduling-for-distributed-computing-clusters/)(Microsoft)、[伏羲](http://www.vldb.org/pvldb/vol7/p1393-zhang.pdf)（阿里云）等。构建大规模计算集群,如典型数据中心的成本是非常高的，因此精心设计调度器，合理使用计算资源非常重要。
常见几种类型的调度器的对比分析如下表所示：

类型 | 资源选择 | 排他性 | 分配粒度 | 集群策略 
---|---|---|---|---
中央调度器 | 全局 | 无，时序 | 全局策略 | 严格的优先级（抢占式）
静态分区调度器 |	静态资源集	|无|资源集|	资源集策略	基于调度器实现
两层调度调度器|	动态资源集	|悲观锁  |	增量囤积 |	严格公正
共享状态 | 全局 | 乐观锁 | 	调度器策略	|优先级抢占 
> 摘自[Omega](http://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/41684.pdf)

### 调度器的考量标准

我们首先回顾调度器需要哪些信息来进行调度工作，并了解哪些指标可以用来衡量调度工作质量。

调度器的主要工作是将资源需求与资源提供方做全局最优的匹配。所以一方面需要了解不同类型资源拓扑。充分掌握环境拓扑信息能够更充分的利用资源（例如，经常访问数据的任务如果距数据近可以显著减少执行时间）而且调度器可以基于信息定义更加复杂的策略，但全局资源信息的维护消耗会限制集群的整体规模和调度执行时间。也让调度器难以扩展，限制集群规模。
另一方面调度器还需要对工作负载有充分的认识，不同类型的工作负载会有不同的甚至截然相反的特性。例如服务类任务需求资源少而但会长时间运行，对调度时间不敏感，批处理类任务需要资源运行时间多短暂，任务可能相关，对调度时间要求较高。
同时，调度器也要满足使用方的特殊要求。如任务尽量集中/分散，保证多个任务同时进行等。

综上所述，调度的调度结果要满足但不限于下列条件：
    
* 资源使用率最大化。
* 满足用户指定的调度需求。
* 满足自定义优先级要求
* 调度决定时间小。
* 保证调度的同时支持调度器的更新
* 各种层级的公平

好的调度器需要平衡好单次调度（调度时间，质量），同时要考虑到环境变化对调度结果的影响，保持结果最优（必要时重新调度），保证集群规模，同时还要能够支持用户无感知的升级和扩展。


### 锁对调度器设计的影响

类似Mesos等两层调度器，一般采用悲观锁的设计实现在当任务需要的所有资源满足时就启动任务的需求；而共享状态的调度器，才考虑使用乐观锁的实现。
通过一个简单的例子，我们比较下悲观锁和乐观锁处理逻辑的不同，假设有如下的一个场景：
1. 作业A读取对象O
2. 作业B读取对象O
3. 作业A在内存中更新对象O
4. 作业B在内存中更新对象O
5. 作业A写入对象O实现持久化
6. 作业B写入对象O实现持久化

悲观锁的设计在对象O实现独占锁，直到作业A完成对对象O的更新并写入持久化数据之前，阻断其他读取请求。
乐观锁的设计在对象O实现共享锁，假设所有的工作都能够正常完成，直到有冲突产生，记录冲突的发生并拒绝冲突的请求。往往乐观锁会配合资源版本实现，同样是上述中的例子，当前对象O的版本为v1，作业A首先完成对象O的写入持久化操作，并标记对象O的版本为v2，作业B在更新时发现对象版本已经变化，会取消更改。


## Kubernetes中的调度原理

Kubernetes的中的计算任务大多通过pod来运行。pod为用户定义的一个或多个共享资源的容器，是可供调度器分配的单元。kubernetes的调度器是控制平面的一部分，它主要监听api server并查找没有被调度的pod，并为这些pod分配运行的节点。调度器主要依据资源消耗的[描述](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/resources.md)得到一个调度结果。

### 调度器的设计

kubernetes的调度设计参考了[Omega](https://ai.google/research/pubs/pub41684)。主要采用两层调度架构，使用全局状态进行调度，通过乐观锁控制资源归属。
Omega论文中对两层架构，全局状态和乐观锁进行了详细的分析。

两层架构帮助调度器屏蔽了很多底层实现细节，让调度器能够更灵活适应资源变化和满足用户指定的调度需求。用户可以将策略和限制分别实现，过滤可用资源。相比较单体架构，更容易添加自定义规则和对集群动态伸缩的支持更好，潜在支持多调度器因而支持集群规模更大。

全局状态和乐观锁的好处是调度器可以看到集群所有可以支配的资源。然后抢占低优先级任务的资源，达到策略要求的状态。相比较使用悲观锁和部分环境视图的架构（如mesos），它资源分配更符合策略要求。避免了作业囤积资源导致集群死锁的问题。当然这会有被抢占任务的开销，和冲突导致的重试。但总体来看资源的使用率更高了。

Kubernetes中默认只有一个调度器。而omega的设计本身支持一个资源分配管理器共享环境信息给多个调度器。所以在设计上Kubernetes兼容多个调度器。

### 调度器的实现
调度器的工作流程如下图所示。通过不断轮询Pod创建、更新、删除的事件启动调度流程。
1. 顺利的调度过程，实现快速assume和bind，从而完成pod的调度
2. 遇到错误的调度过程，通过优先级抢占的方式，获取优先调度的能力

![](https://i.imgur.com/tf9zb4i.png)


#### 调度循环的完整逻辑

1. [监听api]( https://github.com/kubernetes/kubernetes/blob/9cbccd38598e5e2750d39e183aef21a749275087/pkg/scheduler/factory/factory.go#L631)

```go
func NewPodInformer(client clientset.Interface, resyncPeriod time.Duration) coreinformers.PodInformer {
	selector := fields.ParseSelectorOrDie(
		"status.phase!=" + string(v1.PodSucceeded) +
			",status.phase!=" + string(v1.PodFailed))   #选择监听状态异常的pod
	lw := cache.NewListWatchFromClient(client.CoreV1().RESTClient(), string(v1.ResourcePods), metav1.NamespaceAll, selector)
	return &podInformer{
		informer: cache.NewSharedIndexInformer(lw, &v1.Pod{}, resyncPeriod, cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc}),
	}
}
```

2. 将没有调度的pod[放入队列](https://github.com/kubernetes/kubernetes/blob/9cbccd38598e5e2750d39e183aef21a749275087/pkg/scheduler/eventhandlers.go#L156)

```go
func (sched *Scheduler) addPodToSchedulingQueue(obj interface{}) {
	if err := sched.config.SchedulingQueue.Add(obj.(*v1.Pod)); err != nil {
		utilruntime.HandleError(fmt.Errorf("unable to queue %T: %v", obj, err))
	}
}
```

3. 然后给每个pod[执行调度](https://github.com/kubernetes/kubernetes/blob/9cbccd38598e5e2750d39e183aef21a749275087/pkg/scheduler/scheduler.go#L457)。单个Pod的调度分为预选和优选两个阶段，预选阶段通过检查资源的满足情况，选择合适的节点；优选阶段通过打分的方式，选择分值最高的节点进行调度。

```go
	pod := sched.config.NextPod()
	// pod could be nil when schedulerQueue is closed
	if pod == nil {
		return
	}
	if pod.DeletionTimestamp != nil {
		sched.config.Recorder.Eventf(pod, v1.EventTypeWarning, "FailedScheduling", "skip schedule deleting pod: %v/%v", pod.Namespace, pod.Name)
		klog.V(3).Infof("Skip schedule deleting pod: %v/%v", pod.Namespace, pod.Name)
		return
	}

	klog.V(3).Infof("Attempting to schedule pod: %v/%v", pod.Namespace, pod.Name)

	// Synchronously attempt to find a fit for the pod.
	start := time.Now()
	scheduleResult, err := sched.schedule(pod)
```

4. 如果遇到错误，就尝试[抢占低优先级的pod]( https://github.com/kubernetes/kubernetes/blob/01c9edf7fa587090458e03e3a10884817f5aff8d/pkg/scheduler/scheduler.go#L464)

```go
if !util.PodPriorityEnabled() || sched.config.DisablePreemption {
	klog.V(3).Infof("Pod priority feature is not enabled or preemption is disabled by scheduler configuration." +
					" No preemption is performed.")
} else {
	preemptionStartTime := time.Now()
	sched.preempt(pod, fitError)
	metrics.PreemptionAttempts.Inc()
```

5. 找到一个然后继续执行。如果抢占失败，调度程序退出。调度结果不保存意味着pod仍然会出现在未分配的列表 

6. 接下来检查用户提供的[reserve插件](https://github.com/kubernetes/kubernetes/blob/9cbccd38598e5e2750d39e183aef21a749275087/pkg/scheduler/scheduler.go#L508)的条件是否满足。
满足用户提供reserve条件的host，可以被分配到

7. 调度成功后，尝试写回调度器缓存[保存调度结果]( https://github.com/kubernetes/kubernetes/blob/9cbccd38598e5e2750d39e183aef21a749275087/pkg/scheduler/scheduler.go#L379)

```go
	// assume modifies `assumedPod` by setting NodeName=scheduleResult.SuggestedHost
	err = sched.assume(assumedPod, scheduleResult.SuggestedHost)
	if err != nil {
		klog.Errorf("error assuming pod: %v", err)
		metrics.PodScheduleErrors.Inc()
		return
	}
```

启动新的协程将卷绑定到pod上，[调用prebind插件](https://github.com/kubernetes/kubernetes/blob/9cbccd38598e5e2750d39e183aef21a749275087/pkg/scheduler/scheduler.go#L536)，在绑定到节点之前执行用户插件的逻辑。然后通过创建绑定对象将pod绑定到节点。这样释放了主协程。

调度完成后，主协程返回。

#### 单个pod的调度流程
![](https://i.imgur.com/uExB2hh.png)


[过滤](https://github.com/kubernetes/kubernetes/blob/9cbccd38598e5e2750d39e183aef21a749275087/pkg/scheduler/core/generic_scheduler.go#L184)不符合条件节点 

```go
	trace.Step("Computing predicates")
	startPredicateEvalTime := time.Now()
	filteredNodes, failedPredicateMap, err := g.findNodesThatFit(pod, nodes)
	if err != nil {
		return result, err
	}
```

k8s内部预置了许多[过滤规则]( https://github.com/kubernetes/kubernetes/tree/01c9edf7fa587090458e03e3a10884817f5aff8d/pkg/scheduler/algorithm/predicates) 会按照事先定义好的[顺序]( https://github.com/kubernetes/kubernetes/blob/01c9edf7fa587090458e03e3a10884817f5aff8d/pkg/scheduler/algorithm/predicates/predicates.go#L143)  进行过滤

```go
var (
	predicatesOrdering = []string{CheckNodeConditionPred, CheckNodeUnschedulablePred,
		GeneralPred, HostNamePred, PodFitsHostPortsPred,
		MatchNodeSelectorPred, PodFitsResourcesPred, NoDiskConflictPred,
		PodToleratesNodeTaintsPred, PodToleratesNodeNoExecuteTaintsPred, CheckNodeLabelPresencePred,
		CheckServiceAffinityPred, MaxEBSVolumeCountPred, MaxGCEPDVolumeCountPred, MaxCSIVolumeCountPred,
		MaxAzureDiskVolumeCountPred, MaxCinderVolumeCountPred, CheckVolumeBindingPred, NoVolumeZoneConflictPred,
		CheckNodeMemoryPressurePred, CheckNodePIDPressurePred, CheckNodeDiskPressurePred, MatchInterPodAffinityPred}
)
```

例如检查pod使用端口是否与节点可用[端口冲突](
https://github.com/kubernetes/kubernetes/blob/01c9edf7fa587090458e03e3a10884817f5aff8d/pkg/scheduler/algorithm/predicates/predicates.go#L1072)，检查pod是否[满足污点条件]( https://github.com/kubernetes/kubernetes/blob/01c9edf7fa587090458e03e3a10884817f5aff8d/pkg/scheduler/algorithm/predicates/predicates.go#L1530),还有检查是否满足csi最大[可挂载卷限制](
https://github.com/kubernetes/kubernetes/blob/01c9edf7fa587090458e03e3a10884817f5aff8d/pkg/scheduler/algorithm/predicates/csi_volume_predicate.go#L46)。

然后根据预置[实现](
http://releases.k8s.io/HEAD/pkg/scheduler/algorithm/priorities/)按照[默认的规则](https://github.com/kubernetes/kubernetes/blob/3a6d4c10bf70ce2d244b3d9068bf7127b06afdba/pkg/scheduler/algorithmprovider/defaults/defaults.go#L108)进行[打分](
https://github.com/kubernetes/kubernetes/blob/9cbccd38598e5e2750d39e183aef21a749275087/pkg/scheduler/core/generic_scheduler.go#L215)


然后选择[分数最高的节点](
https://github.com/kubernetes/kubernetes/blob/9cbccd38598e5e2750d39e183aef21a749275087/pkg/scheduler/core/generic_scheduler.go#L264)

```go
func (g *genericScheduler) selectHost(priorityList schedulerapi.HostPriorityList) (string, error) {
	if len(priorityList) == 0 {
		return "", fmt.Errorf("empty priorityList")
	}

	maxScores := findMaxScores(priorityList)
	ix := int(g.lastNodeIndex % uint64(len(maxScores)))
	g.lastNodeIndex++

	return priorityList[maxScores[ix]].Host, nil
}
```

### 调度器发展方向

*  [多调度器](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/multiple-schedulers.md)应对不同类型任务，工作负载可以选择调度器执行调度。同一个集群中可以有多个调度器存在。
* 多种调度模式，[coscheduling]( https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/34-20180703-coscheduling.md)(具体实现[kubernetes-batch](https://github.com/kubernetes/kubernetes/issues/61012)）等。
* 获取更多资源分布信息，如[基于拓扑分配卷](https://kubernetes.io/blog/2018/10/11/topology-aware-volume-provisioning-in-kubernetes/)，[获得节点级别的拓扑信息](https://github.com/kubernetes/community/pull/1680)
* 明晰调度过程，定义扩展点，实现[调度框架](https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/20180409-scheduling-framework.md)

## Kubernetes调度器的不足和改进

### 典型的几个问题和思路

#### 目前调度器的实现只关心是否能将pod与节点绑定，对剩余资源并无统计

集群的使用量只能通过监控数据间接推导。如果k8s集群剩余资源不多时，并没有直观数据可以直接用来触发扩容或者告警。社区启动了[cluster-capacity framework项目]( https://github.com/kubernetes-incubator/cluster-capacity/) 提供集群的容量数据，方便集群的维护程序或者管理员基于这些数据做集群扩容等。也有项目抓取监控数据自己计算集群的整体负载情况给调度算法参考，如[poseidon](
https://github.com/kubernetes-sigs/poseidon/blob/master/docs/design/README.md)。

#### 调度去会根据当前状况进行一次调度，一旦分配就没办法更改。

pod只有在自己退出，用户删除，和集群资源不足等情况下才变更。但资源的拓扑是在不停变化的。批处理任务会结束，节点会新增或崩溃，这导致调度的结果可能在调度时最优，但在拓扑变化后调度质量下降。社区[经过讨论]( https://github.com/kubernetes/kubernetes/issues/12140) 决定需要重新找出不满足调度策略的pod,[删除并创建替代者来重新调度](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/rescheduling.md) 根据这个设计启动了项目[descheduler](https://github.com/kubernetes-incubator/descheduler/)。

#### 调度是单个pod进行，因而调度互相关联的工作负载会难以实现。

但大数据分析，机器学习等计算多依赖批处理任务，这类工作负载相关性大，互相之间有依赖。社区提出了[coscheduling]( https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/34-20180703-coscheduling.md) 一次调度一组pod. 优化这类任务的执行速度。

### Kubernetes调度器的定制扩展

如上节描述，通用调度器在某些场景不能满足需求，在自己运行集群时，往往需要根据环境做特殊定制。定制调度器有以下选择：

* 如果默认实现的规则够用，可以更改默认调度算法用到的过滤规则和打分规则，k8s提供了[默认的配置](http://releases.k8s.io/HEAD/pkg/scheduler/algorithmprovider/defaults/defaults.go)， 用户可以更改默认的策略文件[样例](https://git.k8s.io/examples/staging/scheduler-policy-config.json) 来覆盖默认配置

```json
{
"kind" : "Policy",
"apiVersion" : "v1",
"predicates" : [
	{"name" : "PodFitsHostPorts"},
	{"name" : "PodFitsResources"},
	{"name" : "NoDiskConflict"},
	{"name" : "NoVolumeZoneConflict"},
	{"name" : "MatchNodeSelector"},
	{"name" : "HostName"}
	],
"priorities" : [
	{"name" : "LeastRequestedPriority", "weight" : 1},
	{"name" : "BalancedResourceAllocation", "weight" : 1},
	{"name" : "ServiceSpreadingPriority", "weight" : 1},
	{"name" : "EqualPriority", "weight" : 1}
	],
"hardPodAffinitySymmetricWeight" : 10,
"alwaysCheckAllPredicates" : false
}
```
* 如果条件不满足需求，可以添加自定义过滤条件和打分条件，重新编译调度器，并结合策略文件进行配置
执行逻辑基本不变，添加自定义的条件

* 实现调度器各阶段[扩展接口](https://github.com/kubernetes/kubernetes/blob/9cbccd38598e5e2750d39e183aef21a749275087/pkg/scheduler/algorithm/scheduler_interface.go#L28 )，更改调度器的 过滤，打分，抢占，预留等过程 
更改调度器的默认行为（将执行条件计算的逻辑覆盖）
* 也可以实现网络服务[通过钩子函数定制]( https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/scheduler_extender.md)
  执行条件计算后调用外部webhook服务。调整调度结果
* 更改调度器调度[算法](https://github.com/kubernetes/kubernetes/blob/9cbccd38598e5e2750d39e183aef21a749275087/pkg/scheduler/core/generic_scheduler.go#L107) 

```go
type ScheduleAlgorithm interface {
	Schedule(*v1.Pod, algorithm.NodeLister) (scheduleResult ScheduleResult, err error)
	// Preempt receives scheduling errors for a pod and tries to create room for
	// the pod by preempting lower priority pods if possible.
	// It returns the node where preemption happened, a list of preempted pods, a
	// list of pods whose nominated node name should be removed, and error if any.
	Preempt(*v1.Pod, algorithm.NodeLister, error) (selectedNode *v1.Node, preemptedPods []*v1.Pod, cleanupNominatedPods []*v1.Pod, err error)
	// Predicates() returns a pointer to a map of predicate functions. This is
	// exposed for testing.
	Predicates() map[string]predicates.FitPredicate
	// Prioritizers returns a slice of priority config. This is exposed for
	// testing.
	Prioritizers() []priorities.PriorityConfig
}
```

实现其他的算法代替两层调度。但重用调度器查找pod,和进入调度循环的逻辑

* 自己从头实现调度器逻辑。这方面的教程社区仍在[讨论](https://github.com/kubernetes/kubernetes/issues/17208)中


k8s提供了默认的算法满足pod能够正常运行。但当涉及底层资源种类众多，工作负载类型多样，相关程度大时，定制还是很必要的。

## 企业场景应用的案例
### 通用计算场景
Kubernetes default-scheduler满足通用计算的需求，在大规模集群中用于web应用程序的调度过程。

### 大规模集群中基于图应用数据局部性减少任务执行时间同时混合调度算法提升不同场景的调度速度:[poseidon](https://github.com/kubernetes-sigs/poseidon)

poseidon是基于Firmament算法的调度器，它通过接收heapster数据来构建资源使用信息。调用Firmament实现进行调度。

[Firmament](https://www.usenix.org/conference/osdi16/technical-sessions/presentation/gog)算法受[Quincy](https://www.microsoft.com/en-us/research/publication/quincy-fair-scheduling-for-distributed-computing-clusters/)（请见下文介绍）启发，构建一个从任务到节点的图，但作者为减少调度时间，将两种计算最短路径的算法合并，将全量环境信息同步改为增量同步。让Firmament处理短时间批量任务时快于Quincy，在资源短缺时没有kubernetes默认调度器超时的问题。

### 批处理场景执行[群组调度](https://en.wikipedia.org/wiki/Gang_scheduling):[kube-batch](https://github.com/kubernetes-sigs/kube-batch)

大数据分析和机器学习类任务执行时需要大量资源，多个任务同时进行时，资源很快会用尽，部分任务会需要等待资源释放。这类型任务的步骤往往互相关联，单独运行步骤可能会影响最终结果。使用默认的调度器在集群资源紧张时，甚至会出现占用资源的pod都在等待依赖的pod运行完毕，而集群没有空闲资源去运行依赖任务，导致死锁。所以在调度这类任务时，支持群组调度（all or nothing）减少了pod数量，因而降低调度器的负载。同时避免了很多资源紧张带来的问题。

与默认调度器一次调度一个pod不同，kube-batch定义了PodGroup 定义一组相关的pod资源，并实现了一个全新的调度器。调度器的流程基本与默认调度器相同。podgroup保证一组pod可以同时被调度。

## 其他调度系统的设计

### [Quincy](https://www.microsoft.com/en-us/research/publication/quincy-fair-scheduling-for-distributed-computing-clusters/)使用图解决数据局部性需求同时达到调度公平
数据访问密集型任务除受资源限制外网络访问对执行效率的影响也很大。如果数据和计算资源在一块，应用数据局部性原理将任务调度到距依赖数据更近的资源可以大幅度减少网络访问延迟。从而极大地提高执行效率。

批处理任务在进入准备状态前，依赖的数据和依赖步骤的结果需要先准备好。如果可以将访问数据的代价，驱逐低优先级任务的代价换算成一个路径的代价。我们就可以找出调度任务的一个最低代价组合。图的结构非常适合描述这类有依赖的元素之间最短路径的问题。

#### 最小代价流

如上所述，我们可以把任务当成图左边的起点。中间的节点是任务的依赖数据和限制描述。右边的节点是最终被调度的节点。构建好任务到资源的流后可以很容易找到最小代价流。并使用这个结果进行调度。


用户可以通过调整每个路径的代价来调整调度器的输出。而将访问数据代价增大会让调度器更偏向于优化局部性访问。如果环境变化调度器会重新构建任务资源流找到全局最优解，并按结果需要去抢占任务。

## 总结
本文介绍了调度器的问题背景和工作原理，通过了解默认调度器的工作方式，可以了解默认情况下工作负载是如何分配到资源节点上。并简单介绍了社区中对调度器不足的常用应对方法。还通过比较其他调度系统和分析调度器实现，给定制调度器提供了设计思路和实现参考。结合这些思路，后续我们将分享我们在实际应用中的一些实践思路。


