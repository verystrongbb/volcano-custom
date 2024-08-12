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
- In general, a plugin mainly consists of 3 functions: Name OnSessionOpen OnSessionClose. Name provides the name of the plugin. OnSessionOpen executes some operations when a session starts and register some functions about scheduling details. OnSessionClose clean up some resource when a session finishes.
- 代码示例
   <br />![image](https://github.com/user-attachments/assets/9b470f31-ea2d-436f-88e1-427c5b117015)



  <br />其中 AddPrintFns需要注册到session_plugin中和action联系起来
  <br />![image](https://github.com/user-attachments/assets/b2bc569c-4138-4d30-af08-1e5c7af4a34c)

  

   <br />同时在session中定义成员变量printJobFns
   <br />![image](https://github.com/user-attachments/assets/3ab0ac3f-ebd4-46a8-8d24-e8778d997393)

  

  
4. 自定义时可能用上的api
    <br />![image](https://github.com/user-attachments/assets/572167f3-c937-4512-83d8-13f5ab90921f)


  


5. References
- 手把手教你构建自己的Action和Plugin https://www.bilibili.com/video/BV1pV4y1F7tB/?spm_id_from=333.1007.top_right_bar_window_history.content.click
- Volcano 原理、源码分析 https://www.cnblogs.com/daniel-hutao/p/17935624.html#4-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90

6. 以volcano自带的action-backfill和plugin-drf为例

      </br>backfill 算法通过筛选和优先级排序，将待调度的任务分配到最合适的节点，主要分Eexecute部分：

      </br>初始化：
      - 通过 ssn.PredicateForAllocateAction 获取用于节点筛选的函数 predicateFunc
      - 调用 backfill.pickUpPendingTasks(ssn) 获取待调度的任务列表 pendingTasks
      - *pickUpPendingTasks 是该算法里定义的函数，主要是从ssn对象中获取queue（通过QueueOrderFn获取，结果取决于使用的plugin）、job和job中的task信息，处理各种状态的task（pending、piplined）并将其添加到对应tasks优先队列中，处理task后再将job添加到jobs优先队列，最后提取各个优先队列的信息返回pendingtasks

      </br>遍历待调度任务：
      
      - 对于每个待调度任务 task，获取其所属的作业 job
      - 创建一个PredicateHelper对象 ph 和一个FitErrors对象 fe
      - 调用 ssn.PrePredicateFn(task) （结果取决于使用的plugin）执行PrePredicate检查，如果失败，则记录错误并跳过该任务。

      </br>节点筛选：

      - 调用 ph.PredicateNodes(task, ssn.NodeList, predicateFunc, true)  （predicateFunc结果取决于使用的plugin）筛选出符合条件的节点 predicateNodes，并记录 fitErrors
      - 如果没有符合条件的节点，则记录错误并跳过该任务
      

      </br>节点选择：

      - 如果有多个符合条件的节点，则调用 util.PrioritizeNodes 对节点进行优先级排序，并选择最佳节点 node
      - 如果没有最佳节点，则选择优先级最高的节点

      </br>任务分配：

      - 调用 ssn.Allocate(task, node) 将任务分配到选定的节点上，如果分配失败，则记录错误并继续处理下一个任务
      - 更新metrics

      </br>更新状态：

      - 如果任务未能成功分配，则记录适配错误




      </br>drf 主要分析onSessionOpen部分：

      </br>初始化：
      - 通过 ssn.PredicateForAllocateAction 获取用于节点筛选的函
      - 

      </br>AddPreemptableFn：
      
      - 
     

      </br>AddJobOrderFn：

      - 
      
      
      </br> 与action的联系： 

      - 

  

7. 使用自定义 plugin
- 文件 \volcano-master\docs\design\custom-plugin.md 给出了使用自定义plugin的方法

