**相关内容：**

Workflow
![image](https://github.com/user-attachments/assets/21cb8231-8674-410a-a607-f2da79c1bb78)
1. Action

   action定义了调度各环节中需要执行的动作

1. Plugin

   plugin根据不同场景提供了action 中算法的具体实现细节

*来自 <<https://volcano.sh/zh/docs/schduler_introduction/>>* 

**如何查看Volcano scheduler的配置**

查看名为volcano-scheduler-configmap的configmap
```
#kubectl get configmap -nvolcano-system
NAME                          DATA   AGE
volcano-scheduler-configmap   1      6d2h
```
查看configmap的data部分详情
```
#kubectl get configmap volcano-scheduler-configmap -nvolcano-system -oyaml
apiVersion: v1
data:
volcano-scheduler.conf: |
actions: "enqueue, allocate, backfill"
tiers:
- plugins:
  - name: priority
  - name: gang
  - name: conformance
- plugins:
  - name: drf
  - name: predicates
  - name: proportion
  - name: nodeorder
  - name: binpack
kind: ConfigMap
metadata:
annotations:
kubectl.kubernetes.io/last-applied-configuration: |
  {"apiVersion":"v1","data":{"volcano-scheduler.conf":"actions: \"enqueue, allocate, backfill\"\ntiers:\n- plugins:\n  - name: priority\n  - name: gang\n  - name: conformance\n- plugins:\n  - name: drf\n  - name: predicates\n  - name: proportion\n  - name: nodeorder\n  - name: binpack\n"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"volcano-scheduler-configmap","namespace":"volcano-system"}}
creationTimestamp: "2020-08-15T04:01:02Z"
name: volcano-scheduler-configmap
namespace: volcano-system
resourceVersion: "266"
selfLink: /api/v1/namespaces/volcano-system/configmaps/volcano-scheduler-configmap
uid: 1effe4d6-126c-42d6-a3a4-b811075c30f5
```
在volcano-scheduler.conf中主要包括actions和tiers两部分。在actions中，使用逗号作为分隔符配置各需要执行的action。需要注意的是，action的配置 顺序就是scheduler的执行顺序。Volcano本身不会对action顺序的合理性进行检查。tiers中配置的plugin列表即为注册到scheduler中的plugin。plugin中 实现的算法将会被action调用。

![image](https://github.com/user-attachments/assets/8a008e41-70c1-46df-8e4a-2c58340a35b1)



**自定义**

1. 自定义调度器主要是自定义action和plugin这两个部分，ACTION & Plugin的接口如下，两者通过session对象进行关联
    <br />![image](https://github.com/user-attachments/assets/a1f6be1b-aa85-4bf4-a9b1-1853a2da1f98)

2. 自定义ACTION
- action主要是调用经plugin处理的结果
- 代码示例
   <br />以下是自定义了一个打印job信息的action，需要将具体的打印函数注册到session_plugin中
   <br />![image](https://github.com/user-attachments/assets/249b3f22-d874-4d91-9862-dc22a2879b7c)



3. 自定义Plugin
- *\volcano-master\docs\user-guide\how_to_configure_scheduler.md*中给出了一个plugin的核心部分：In general, a plugin mainly consists of 3 functions: Name OnSessionOpen OnSessionClose. Name provides the name of the plugin. OnSessionOpen executes some operations when a session starts and register some functions about scheduling details. OnSessionClose clean up some resource when a session finishes.
- 代码示例
   <br />![image](https://github.com/user-attachments/assets/9b470f31-ea2d-436f-88e1-427c5b117015)



  <br />其中 AddPrintFns需要注册到session_plugin中和action联系起来
  <br />![image](https://github.com/user-attachments/assets/b2bc569c-4138-4d30-af08-1e5c7af4a34c)

  

   <br />同时在session中定义成员变量printJobFns
   <br />![image](https://github.com/user-attachments/assets/3ab0ac3f-ebd4-46a8-8d24-e8778d997393)

4. 自定义时可能用上的api
    <br />![image](https://github.com/user-attachments/assets/572167f3-c937-4512-83d8-13f5ab90921f)

5. 以volcano自带的***action-preempt***、***action-backfill***和***plugin-drf***为例：
      </br>***preempt 动作根据优先级回收job或者task，涉及到的plugin有如下几个：***
      1.  TaskOrderFn(Plugin: Priority),
      2.  JobOrderFn(Plugin: Priority, DRF, Gang),
      3.  NodeOrderFn(Plugin: NodeOrder),
      4.  PredicateFn(Plugin: Predicates),
      5.  PreemptableFn(Plugin: Conformance, Gang, DRF).

      </br>***主要的逻辑如下：***
      
      </br>先用IsPending、JobValid、JobStarving等筛选出需要加入抢占队列的job和task:

      - IsPending
      - JobValid
      - JobStarving
      - 用map将task和job优先队列映射到其所属的job和queue，优先级由TaskOrderFn、JobOrderFn两个函数决定，其定义则取决于调用的plugin，比如DRF就定义了一个JobOrderFn，通过AddJobOrderFn注册到ssn对象中

      </br>然后定义抢占行为，preempt 动作的抢占分为两种粒度，一个是同一queue中的job互相抢占，另一个是同一job的task间互相抢占：

      - 同一queue中的job可以互相抢占，对每个queue建立一个循环进行如下逻辑：

      1. 检查抢占队列preemptorsMap是否存在preemptors(一组job)
      2. 若存在，最高优先级的job出队作为preemptorJob
      3. new一个statement对象用于commit和dicard操作
      4. 对每一个作为preemptor的job，检查还有没有preemptorTasks，若没有则跳过
      5. 从task优先队列pop一个task进行preempt操作
      6. 检查并提交，preemptorJob重新入队进入下一次循环

      - 同一job的task间可以互相抢占，对每个underRequest的job进行如下逻辑（代码有个问题，这里是写在queue的循环里面的但是并没有用上queue，感觉可以将这个循环分离出来）：
      
      1. 重新初始化一个优先队列preemptorTasks（因为之前的代码更新过preemptorTasks），将处于Pending的task加入其中，以jobuid为索引
      2. 检查抢占队列preemptorTasks是否存在preemptors(一组task)
      3. 若存在，最高优先级的task出队作为preemptor
      4. new一个statement对象用于commit和dicard操作
      5. preemptor进行preempt操作
      6. 检查并提交

      - ***preempt操作逻辑如下***，PrePredicateFn、PrioritizeNodes（含BatchNodeOrderFn、NodeOrderMapFn、NodeOrderReduceFn）BuildVictimsPriorityQueue（含TaskOrderFn、JobOrderFn）Allocatable（含allocatableFns）Preemptable（含preemptableFns）这些需要找到对应的plugin实现，下面是部分plugin实现的fn：
      </br>![image](https://github.com/user-attachments/assets/8171f05b-3c35-46df-9be0-0e5719069b1c)

      </br>图源：https://yost.top/2020/08/04/volcano-code-review/ </br>*\volcano-master\volcano-master\docs\user-guide\how_to_configure_scheduler.md*里面有更多的

      1. 初始化变量
      2. 预处理检查：调用 ssn.PrePredicateFn 函数进行预处理检查，如果失败则返回错误。
      3. 获取节点列表：调用 ssn.GetUnschedulableAndUnresolvableNodesForTask 获取不可调度和不可解决的节点列表。
      4. 节点过滤：使用 predicateHelper.PredicateNodes 函数过滤节点。
      5. 节点优先级排序：调用 util.PrioritizeNodes 函数对节点进行优先级排序。
      6. 节点排序：调用 util.SortNodes 函数对节点进行排序。
      7. 获取作业和队列：从 ssn.Jobs 中获取作业信息，并从 ssn.Queues 中获取当前队列信息。
      8. 遍历节点：遍历排序后的节点列表。
      9. 过滤任务：根据 filter 函数（此处的filter在上文通过lambda函数的形式定义了，过滤掉不可抢占的、不同queue/job的）过滤任务，如果没有指定就之间添加该节点的task。
      ```
      	for _, task := range node.Tasks {
			if filter == nil {
				preemptees = append(preemptees, task.Clone())
			} else if filter(task) {
				preemptees = append(preemptees, task.Clone())
			}
		}
      ```
      10. 获取可抢占任务：调用 ssn.Preemptable 函数获取可抢占的任务victims。
      11. 更新抢占victim计数：调用 metrics.UpdatePreemptionVictimsCount 更新抢占受害者计数。
      12. 验证victim资源：调用 util.ValidateVictims 函数验证victim资源是否满足抢占者的资源需求，不满足则跳过。
      ```
      if err := util.ValidateVictims(preemptor, node, victims); err != nil {
			klog.V(3).Infof("No validated victims on Node <%s>: %v", node.Name, err)
			continue
		}
      ```
      13. 构建victim优先级队列：调用 ssn.BuildVictimsPriorityQueue 构建victim优先级队列，如果victim的jobid相同就调用TaskOrderFn构建，否则调用JobOrderFn。
      ```
      func (ssn *Session) BuildVictimsPriorityQueue(victims []*api.TaskInfo) *util.PriorityQueue {
	victimsQueue := util.NewPriorityQueue(func(l, r interface{}) bool {
		lv := l.(*api.TaskInfo)
		rv := r.(*api.TaskInfo)
		if lv.Job == rv.Job {
			return !ssn.TaskOrderFn(l, r)
		}
		return !ssn.JobOrderFn(ssn.Jobs[lv.Job], ssn.Jobs[rv.Job])
	})
	for _, victim := range victims {
		victimsQueue.Push(victim)
	}
	return victimsQueue
   }

      ```
      14. 抢占任务：从优先级队列中弹出任务并进行抢占，直到满足资源需求或队列为空。
      
      ```
      for !victimsQueue.Empty() {
			// If reclaimed enough resources, break loop to avoid Sub panic.
			// Preempt action is about preempt in same queue, which job is not allocatable in allocate action, due to:
			// 1. cluster has free resource, but queue not allocatable
			// 2. cluster has no free resource, but queue not allocatable
			// 3. cluster has no free resource, but queue allocatable
			// for case 1 and 2, high priority job/task can preempt low priority job/task in same queue;
			// for case 3, it need to do reclaim resource from other queue, in reclaim action;
			// so if current queue is not allocatable(the queue will be overused when consider current preemptor's requests)
			// or current idle resource is not enougth for preemptor, it need to continue preempting
			// otherwise, break out
			if ssn.Allocatable(currentQueue, preemptor) && preemptor.InitResreq.LessEqual(node.FutureIdle(), api.Zero) {
				break
			}
			preemptee := victimsQueue.Pop().(*api.TaskInfo)
			klog.V(3).Infof("Try to preempt Task <%s/%s> for Task <%s/%s>",
				preemptee.Namespace, preemptee.Name, preemptor.Namespace, preemptor.Name)
			if err := stmt.Evict(preemptee, "preempt"); err != nil {
				klog.Errorf("Failed to preempt Task <%s/%s> for Task <%s/%s>: %v",
					preemptee.Namespace, preemptee.Name, preemptor.Namespace, preemptor.Name, err)
				continue
			}
			preempted.Add(preemptee.Resreq)
		}
      ```

      15. 注册PreemptionAttempt：调用 metrics.RegisterPreemptionAttempts 注册PreemptionAttempt。
      16. 分配任务：如果资源满足条件，则调用 stmt.Pipeline 函数分配任务。
      ```
      // If preemptor's queue is overused, it means preemptor can not be allocated. So no need care about the node idle resource
		if ssn.Allocatable(currentQueue, preemptor) && preemptor.InitResreq.LessEqual(node.FutureIdle(), api.Zero) {
			if err := stmt.Pipeline(preemptor, node.Name); err != nil {
				klog.Errorf("Failed to pipeline Task <%s/%s> on Node <%s>",
					preemptor.Namespace, preemptor.Name, node.Name)
			}

			// Ignore pipeline error, will be corrected in next scheduling loop.
			assigned = true

			break
		}
      ```
      17. 返回结果：返回 assigned 和 nil判断是否成功preempt和有无错误。

      </br>最后调用victimTasks去evict这些victim（定义的preempt函数还没有真正地evict），需要plugin实现里面的VictimTasks的victimTasksFns来获取victimSet
      ```
      func victimTasks(ssn *framework.Session) {
	stmt := framework.NewStatement(ssn)
	tasks := make([]*api.TaskInfo, 0)
	victimTasksMap := ssn.VictimTasks(tasks)
	for victim := range victimTasksMap {
		if err := stmt.Evict(victim.Clone(), "evict"); err != nil {
			klog.Errorf("Failed to evict Task <%s/%s>: %v",
				victim.Namespace, victim.Name, err)
			continue
		}
	}
	stmt.Commit()
   }
      ```
      ```
      func (ssn *Session) VictimTasks(tasks []*api.TaskInfo) map[*api.TaskInfo]bool {
	// different filters may add the same task to victims, so use a map to remove duplicate tasks.
	victimSet := make(map[*api.TaskInfo]bool)
	for _, tier := range ssn.Tiers {
		for _, plugin := range tier.Plugins {
			if !isEnabled(plugin.EnabledVictim) {
				continue
			}
			fns, found := ssn.victimTasksFns[plugin.Name]
			if !found {
				continue
			}
			for _, fn := range fns {
				victimTasks := fn(tasks)
				for _, victim := range victimTasks {
					victimSet[victim] = true
				}
			}
		}
		if len(victimSet) > 0 {
			return victimSet
		}
	}
	return victimSet
   }
      ```

      </br>***backfill 算法通过筛选和优先级排序，将待调度的任务分配到最合适的节点，主要分析execute部分：***

      </br>初始化：
      - 通过 ssn.PredicateForAllocateAction 获取用于节点筛选的函数 predicateFunc
      - 调用 backfill.pickUpPendingTasks(ssn) 获取待调度的任务列表 pendingTasks
        </br>![image](https://github.com/user-attachments/assets/fc2cbbdd-768f-4040-b01a-9bf43ebf55ad)


      - *pickUpPendingTasks* 是该算法里定义的函数，主要是从ssn对象中获取queue（通过QueueOrderFn获取，结果取决于使用的plugin）、job和job中的task信息，处理各种状态的task（pending、piplined）并将其添加到对应tasks优先队列中，处理task后再将job添加到jobs优先队列，最后提取各个优先队列的信息返回pendingtasks

      </br>遍历待调度任务：
      
      - 对于每个待调度任务 task，获取其所属的作业 job
      - 创建一个PredicateHelper对象 ph 和一个FitErrors对象 fe
      - 调用 ssn.PrePredicateFn(task) （结果取决于使用的plugin）执行PrePredicate检查，如果失败，则记录错误并跳过该任务。
        </br>![image](https://github.com/user-attachments/assets/7f4a8be5-63a5-4863-87e8-5abcdb3b788a)


      </br>节点筛选：

      - 调用 ph.PredicateNodes(task, ssn.NodeList, predicateFunc, true)  （predicateFunc结果取决于使用的plugin）筛选出符合条件的节点 predicateNodes，并记录 fitErrors
      - 如果没有符合条件的节点，则记录错误并跳过该任务
        </br>![image](https://github.com/user-attachments/assets/37b6dfd5-3b76-4cb0-b06c-cb6c61847bdd)

      

      </br>节点选择：

      - 如果有多个符合条件的节点，则调用 util.PrioritizeNodes 对节点进行优先级排序，并选择最佳节点 node
      - 如果没有最佳节点，则选择优先级最高的节点,其中BestNodeFn、PrioritizeNodes的结果都和选取的plugin有关（后者是作为传入参数的函数与plugin有关）
        </br>![image](https://github.com/user-attachments/assets/0efd4305-73c4-4dd8-8a69-0d44a58fc18d)
        </br>![image](https://github.com/user-attachments/assets/562e24e7-1e37-4ec1-ab91-a435622680db)
      - SelectBestNode选取具有较高的score的node，有相同最高score的随机选取



      </br>任务分配：

      - 调用 ssn.Allocate(task, node) 将任务分配到选定的节点上，如果分配失败，则记录错误并继续处理下一个任务
        </br>![image](https://github.com/user-attachments/assets/464323b0-7888-4699-a42b-b9097beeafdd)

      - 更新metrics
        </br>![image](https://github.com/user-attachments/assets/f0f1510b-cacf-42b8-aa18-331e3362c20e)


      </br>更新状态：

      - 如果任务未能成功分配，则记录适配错误
        </br>![image](https://github.com/user-attachments/assets/febb49b0-2c89-423b-a838-176aa389b331)





      </br>***drf 主要分析onSessionOpen部分，里面主要是提供了两个函数供调度使用，其他的算法有关的计算函数如calculateShare也写在drf.go里面，最后还写了一个event handler：***

     

      </br>AddPreemptableFn：
      
      - 先根据drf算法编写preemptableFn计算可以被抢占的任务，要是根据资源份额决定哪些任务可以被抢占。它通过比较抢占者和被抢占者的share值，选择share值较大的任务进行抢占，其中的抢占任务和被抢占任务对象已经在api中的job_info中封装好了（TaskInfo），可以获取Resreq（resource that used when task running）、所属的job等信息。

         </br>![image](https://github.com/user-attachments/assets/0a64faa4-3a97-4ac3-9f22-5caf3b8ed849)

      - 在session_plugin中编写AddPreemptableFn将preemptableFn注册到ssn对象中（用一个map储存），此时action就可以调用相应的接口（session_plugin.go中的Preemptable函数）经由ssn对象获取victims, 比如preempt.go中的
         </br>![image](https://github.com/user-attachments/assets/4a5b8db3-42f7-4e15-8ff1-0123e9b11309)

      
     

      </br>AddJobOrderFn：

      - 同上先编写jobOrderFn比较两个job，比较计算出的share值分别返回0，-1，1
         </br>![image](https://github.com/user-attachments/assets/e2b6f50d-d4e1-4bcd-8d6d-3c23d1ecb089)

      - 然后在session_plugin中编写AddJobOrderFn将jobOrderFn注册到ssn对象中，此时action就可以调用相应的接口经由ssn对象获取比较结果，比如preempt.go中的
         </br>![image](https://github.com/user-attachments/assets/772809f6-8167-4fa9-8cb3-075f1ee8bc6e)


      </br>calculateShare计算share值：
      - 就是根据drf定义计算
         </br>![image](https://github.com/user-attachments/assets/1fde66e2-bee5-4529-83a8-75f4d1254837)

         </br>![image](https://github.com/user-attachments/assets/4eaa2f66-3a4b-4b7f-9489-29f4597b3522)

      </br>*开启了hierarchy之后还有AddQueueOrderFn和AddReclaimableFn，原理同AddJobOrderFn和AddPreemptableFn

      </br>![image](https://github.com/user-attachments/assets/d0ef20bb-0ce9-4158-988b-b52c419dff51)

      
7. 使用自定义 plugin
- 文件 *\volcano-master\docs\design\custom-plugin.md* 给出了使用自定义plugin的方法

7. References
- 手把手教你构建自己的Action和Plugin https://www.bilibili.com/video/BV1pV4y1F7tB/?spm_id_from=333.1007.top_right_bar_window_history.content.click
- Volcano 原理、源码分析 https://www.cnblogs.com/daniel-hutao/p/17935624.html#4-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90

