**相关内容：**

Workflow

1. Action

   action定义了调度各环节中需要执行的动作

1. Plugin

   plugin根据不同场景提供了action 中算法的具体实现细节

 

*来自 <<https://volcano.sh/zh/docs/schduler_introduction/>>* 





配置

由于Volcano scheduler采用了组合模式的设计，它具有高度的扩展性。用户可以根据个人需要决定使用哪些action和plugin，也可以根据action和plugin的 接口自定义实现。scheduler的配置位于名为**volcano-scheduler-configmap**的configmap内，该configmap被作为volume挂载在容器的/volcano.scheduler 路径下。



*来自 <<https://volcano.sh/zh/docs/schduler_introduction/>>* 



**如何查看Volcano scheduler的配置**

- 查看名为volcano-scheduler-configmap的configmap
  kubectl get configmap -nvolcano-system
  NAME                          DATA   AGE
  volcano-scheduler-configmap   1      6d2h
- 查看configmap的data部分详情
  kubectl get configmap volcano-scheduler-configmap -nvolcano-system -oyaml
  apiVersion: v1
  data:
  volcano-scheduler.conf: |
  **actions: "enqueue, allocate, backfill"**
  **tiers:**
      - plugins:
  `  `- name: priority
  `  `- name: gang
  `  `- name: conformance
      - plugins:
  `  `- name: drf
  `  `- name: predicates
  `  `- name: proportion
  `  `- name: nodeorder
  `  `- name: binpack
  kind: ConfigMap
  metadata:
  annotations:
  kubectl.kubernetes.io/last-applied-configuration: |
  `  `{"apiVersion":"v1","data":{"volcano-scheduler.conf":"actions: \"enqueue, allocate, backfill\"\ntiers:\n- plugins:\n  - name: priority\n  - name: gang\n  - name: conformance\n- plugins:\n  - name: drf\n  - name: predicates\n  - name: proportion\n  - name: nodeorder\n  - name: binpack\n"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"volcano-scheduler-configmap","namespace":"volcano-system"}}
  creationTimestamp: "2020-08-15T04:01:02Z"
  name: volcano-scheduler-configmap
  namespace: volcano-system
  resourceVersion: "266"
  selfLink: /api/v1/namespaces/volcano-system/configmaps/volcano-scheduler-configmap
  uid: 1effe4d6-126c-42d6-a3a4-b811075c30f5

在volcano-scheduler.conf中主要包括actions和tiers两部分。在actions中，使用逗号作为分隔符配置各需要执行的action。需要注意的是，action的配置 顺序就是scheduler的执行顺序。Volcano本身不会对action顺序的合理性进行检查。tiers中配置的plugin列表即为注册到scheduler中的plugin。plugin中实现的算法将会被action调用。



*来自 <<https://volcano.sh/zh/docs/schduler_introduction/>>* 








**自定义**

1. 自定义ACTION

 

1. 自定义Plugin
- In general, a plugin mainly consists of 3 functions: Name OnSessionOpen OnSessionClose. Name provides the name of the plugin. OnSessionOpen executes some operations when a session starts and register some functions about scheduling details. OnSessionClose clean up some resource when a session finishes.

 









