# Sedna概述

Sedna是在KubeEdge SIG AI中孵化的一个边云协同AI项目。得益于KubeEdge提供的边云协同能力，Sedna可以实现跨边云的协同训练和协同推理能力，如联合推理、增量学习、联邦学习、终身学习等。Sedna支持目前广泛使用的AI框架，如TensorFlow/Pytorch/MindSpore等，现有AI类应用可以无缝迁移到Sedna, 快速实现边云协同的训练和推理，可在降低成本、提升模型性能、保护数据隐私等方面获得提升。

项目主页：
https://github.com/kubeedge/sedna

文档参考：
https://sedna.readthedocs.io

## 整体架构

Sedna的边云协同基于KubeEdge提供的如下能力实现
* 跨边云应用统一编排
* Router: 管理面云边高可靠消息通道
* EdgeMesh: 数据面跨边云微服务发现和流量治理

![](images/architecture.png)

**基本组件**：


* **GlobalManager**
  * 统一边云协同AI任务管理
  * 跨边云协同管理与协同
  * 中心配置管理


* **LocalController**

  * 边云协同AI任务的本地流程控制

  * 本地通用管理: 模型，数据集，状态同步等


* **Lib**

  * 面向AI开发者和应用开发者，暴露边云协同AI功能给应用

* **Worker**

  * 执行训练或推理任务, 基于现有AI框架开发的训练/推理程序

  * 不同特性对应不同的worker组，worker可部署在边上或云上，并进行协同

## 工程目录

| 目录       | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| .github    | Sedna github CICD流水线配置。                                |
| LICENSES   | Sedna Licenses以及相关vendor Licenses。                      |
| build      | GM/LC等管理面构建的Dockersfile；<br />生成的CRD定义yaml文件；<br />CRD样例yaml文件; |
| cmd        | GM/LC管里面的启动函数。                                      |
| components | 监控和图形化展示的组件。                                     |
| docs       | proposals和安装文档。                                        |
| examples   | 协同推理、增量学习、终身学习、联邦学习的使用样例。           |
| hack       | 面向开发者的代码生成工具、及其他开发会用到的脚本。           |
| lib        | Sedna Library，用于开发边云协同AI应用的Python依赖库。        |
| pkg        | API定义；<br />生成的CRD的client-go代码；<br />Sedna GM/LC 管里面的核心代码。 |
| scripts    | 面向使用者的安装脚本。                                       |
| test       | E2E测试代码及测试工具。                                      |
| vendor     | 依赖的第三方项目源码。                                       |



# Sedna管理面源码解析（Go）

## GM: Global Manager
### GM，一个K8S operator

**operator是什么？**

> An Operator is an application-specific controller that extends the Kubernetes API to create, configure and manage instances of complex stateful applications on behalf of a Kubernetes user.  It builds upon the basic Kubernetes resource and controller concepts, but also includes domain or application-specific knowledge to automate common tasks better managed by computers.  [^1]



对于Sedna，Sedna控制了边云协同AI应用中，如何配置worker部署启动参数、如何协同、如何流转等，那么我们可以这么定义：**Sedna GM是“边云协同AI应用“这个特定领域的控制器**。



组件构成了operator的主要部分

![Sedna](./images/Sedna源码结构.jpg)



接下来的章节会按照组成K8S operator的几个组件来展开说明，包括CR、CRD、API、Controller，其中Controller是主要的控制逻辑模块。

### CR

Sedna本身支持边云协同推理、增量学习、终身学习、联邦学习，为了方便解读代码，本文结合终身学习具体特性和样例来分析。其他三个特性的代码实现存在共通之处，可以类比参考。

**CR样例**

这里贴了一段终身学习[CR样例](https://github.com/kubeedge/sedna/blob/main/build/crd-samples/sedna/lifelonglearningjobv1alpha1.yaml)，可以基于这个CR通过kubectl来创建对应的终身学习资源对象，详细使用步骤可以参考[这里](https://github.com/kubeedge/sedna/tree/main/examples/lifelong_learning/atcii)。其中关键的字段解释如下：

- dataset：指定数据集对象名称，数据集也是一个CR资源。

- trainSpec：终身学习中，训练worker的启动参数，包括镜像和环境变量等容器配置。
- trigger：终身学习中，启动训练worker的触发条件。

- evalSpec：终身学习中，评估work的启动参数，包括镜像和环境变量等容器配置。
- deploySpec：终身学习中，推理work的启动参数，包括镜像和环境变量等容器配置。
- outputDir：终身学习中，训练生成的模型文件输出路径。

`build/crd-samples/sedna/lifelonglearningjobv1alpha1.yaml`

```yaml
apiVersion: sedna.io/v1alpha1
kind: LifelongLearningJob
metadata:
  name: atcii-classifier-demo
spec:
  dataset:
    name: "lifelong-dataset"
    trainProb: 0.8
  trainSpec:
    template:
      spec:
        nodeName:  "edge-node"
        containers:
          - image: kubeedge/sedna-example-lifelong-learning-atcii-classifier:v0.3.0
            name:  train-worker
            imagePullPolicy: IfNotPresent
            args: ["train.py"]
            env:
              - name: "early_stopping_rounds"
                value: "100"
              - name: "metric_name"
                value: "mlogloss"
    trigger:
      checkPeriodSeconds: 60
      timer:
        start: 02:00
        end: 24:00
      condition:
        operator: ">"
        threshold: 500
        metric: num_of_samples
  evalSpec:
    template:
      spec:
        nodeName:  "edge-node"
        containers:
          - image: kubeedge/sedna-example-lifelong-learning-atcii-classifier:v0.3.0
            name:  eval-worker
            imagePullPolicy: IfNotPresent
            args: ["eval.py"]
            env:
              - name: "metrics"
                value: "precision_score"
              - name: "metric_param"
                value: "{'average': 'micro'}"
              - name: "model_threshold"
                value: "0.5"
  deploySpec:
    template:
      spec:
        nodeName:  "edge-node"
        containers:
        - image: kubeedge/sedna-example-lifelong-learning-atcii-classifier:v0.3.0
          name:  infer-worker
          imagePullPolicy: IfNotPresent
          args: ["inference.py"]
          env:
          - name: "UT_SAVED_URL"
            value: "/ut_saved_url"
          - name: "infer_dataset_url"
            value: "/data/testData.csv"
          volumeMounts:
          - name: utdir
            mountPath: /ut_saved_url
          - name: inferdata
            mountPath: /data/
          resources:
            limits:
              memory: 2Gi
        volumes:
          - name: utdir
            hostPath:
              path: /lifelong/unseen_task/
              type: DirectoryOrCreate
          - name: inferdata
            hostPath:
              path:  /data/
              type: DirectoryOrCreate
  outputDir: "/output"
```



### CRD 

CRD可以看作是CR的模板，在k8s集群能创建对应CR之前需要将对应的CRD在k8s集群中进行声明。CRD对应的yaml文件可以手动编写或自动生成，对于一些相对复杂的CRD定义建议采用通过k8s相关工具生成。比如Sedna这里使用的是kubebuilder的[controller-gen](https://book.kubebuilder.io/reference/controller-gen.html#controller-gen-cli)进行自动生成与更新，Sedna项目提供了封装好的脚本，直接通过`make crds`命令即可生成和更新对应`build/crds/`目录下的CRD文件。相关shell脚本可以参考这个文件`Makefile`中的`crds: controller-gen`。

想要完成一个CRD定义，最重要的是需要指定group、version和kind，通常简称为GVK。而CR资源对象本身称为Resource，相较于面向对象中的概念，Resouce类比为Object，Kind类比于Class，也就可以说Resource是Kind的实例。下表展示了终身学习CRD和CR对应的GVR和GVK：

|      | Group                | Version  | Resource                      | Kind                     |
| ---- | -------------------- | -------- | ----------------------------- | ------------------------ |
| CRD  | apiextensions.k8s.io | v1       | lifelonglearningjobs.sedna.io | CustomResourceDefinition |
| CR   | sedna.io             | v1alpha1 | lifelonglearningjob           | LifelongLearningJob      |

在K8S集群中资源是以REST URI的形式来组织的，组织的路径如下：

![img](./images/k8s-gvr.jpg)

了解了上述的规则后，我们可以快速的拼接好要管理的k8s资源对象的REST URI地址，这为某些不能依赖k8s client（kubectl, client-go等）的情况下访问集群资源提供了简便的方式。比如：

通过Rest接口查看终身学习CRD描述：

```shell
curl -k --cert ./client.crt --key ./client.key https://127.0.0.1:5443/apis/apiextensions.k8s.io/v1beta1/customresourcedefinitions/lifelonglearningjobs.sedna.io
```

通过Rest接口查看终身学习CR列表：

```shell
curl -k --cert ./client.crt --key ./client.key https://127.0.0.1:5443/apis/sedna.io/v1alpha1/lifelonglearningjobs
```

比如如果某些编程语言没有官方的k8s client SDK， 那么可以统一采用如上Rest接口形式进行封装。



下面是Sedna 终身学习CRD定义，一些需要关注的字段如下：

- apiVersion: apiextensions.k8s.io/v1，当前所有的CRD都扩展自apiextensions.k8s.io/v1这个Version。
- kind: CustomResourceDefinition，当前所有的CRD都继承自CustomResourceDefinition这个Kind。
- spec.group: sedna.io，自定义资源的Group名称为sedna.io。
- spec.names.kind: LifelongLearningJob, 自定义资源新增加的类型，这里是LifelongLearningJob。
- spec.names.shortNames: - ll，在使用kubectl可以使用这个缩写”ll“查询到LifelongLearningJob资源。

`build/crds/sedna.io_lifelonglearningjobs.yaml`

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.4.1
  creationTimestamp: null
  name: lifelonglearningjobs.sedna.io
spec:
  group: sedna.io
  names:
    kind: LifelongLearningJob
    listKind: LifelongLearningJobList
    plural: lifelonglearningjobs
    shortNames:
    - ll
    singular: lifelonglearningjob
  scope: Namespaced
  versions:
  - name: v1alpha1
	...
status:
  acceptedNames:
    kind: ""
    plural: ""
  conditions: []
  storedVersions: []
```

### API

上面提到我们的CRD是自动生成的，那生成这些CRD所需要的API基础定义在哪里呢？

`pkg/apis/sedna/v1alpha1/lifelonglearningjob_types.go`

```go
package v1alpha1

import (
	v1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// 这里展示了
// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
// +kubebuilder:resource:shortName=ll
// +kubebuilder:subresource:status

// 整体的LifelongLearningJob的API定义，主要包含Spec和Status定义，分别代表期望状态和实际状态。
type LifelongLearningJob struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata"`
	Spec              LLJobSpec   `json:"spec"`
	Status            LLJobStatus `json:"status,omitempty"`
}

// 在创建LifelongLearningJob时候需要配置的参数；如果需要扩展终身学习字段的接口，可以在这里修改。
type LLJobSpec struct {
	Dataset    LLDataset    `json:"dataset"`
	TrainSpec  LLTrainSpec  `json:"trainSpec"`
	EvalSpec   LLEvalSpec   `json:"evalSpec"`
	DeploySpec LLDeploySpec `json:"deploySpec"`

	// the credential referer for OutputDir
	CredentialName string `json:"credentialName,omitempty"`
	OutputDir      string `json:"outputDir"`
}

type LLDataset struct {
	Name      string  `json:"name"`
	TrainProb float64 `json:"trainProb"`
}

// 剩下还有一些结构体定义省略了。
```

上面的代码片段中，补充了额外的说明，需要注意的有如下几点：

- `// +kubebuilder...` 的注释是给kubebuilder等代码自动生成工具的配置参数，会被这些工具解析。
- `type LifelongLearningJob struct{...}`定义了终身学习CRD整体API，主要包含Spec和Status定义，分别代表期望状态和实际状态。
- `type LLJobSpec struct {...}`在创建LifelongLearningJob CR时需要配置的参数；如果需要扩展终身学习字段的接口，可以在这里修改。

其他协同推理、增量学习、联邦学习相关的API定义都可以在`pkg/apis/sedna/v1alpha1/`这个目录下找到。



**更新client-go代码**

一旦新增或者更新了`*_types.go`中的定义，则需要执行如下命令进行client-go代码更新：

```shell
bash hack/update-codegen.sh
```

生成的代码位于`pkg/client`：

```shell
➜  pkg tree client -L 2
client
├── clientset
│   └── versioned
├── informers
│   └── externalversions
└── listers
    └── sedna
```

client-go中的代码会在后面的Contrller逻辑中用到。

**更新CRD定义**

一旦新增或者更新了`*_types.go`中的定义，则需要执行如下命令进行CRD代码更新：

```shell
make crds
```

生成的CRD定义yaml文件位于`build/crds`。更新这些定义之后，也需要同步在K8s集群中重新`kubectl apply`一下，以将新的CRD在集群中生效。

### Controller

终身学习最主要的控制逻辑在这个`pkg/globalmanager/controllers/lifelonglearning/lifelonglearningjob.go`文件里面，包括训练评估Worker什么时候触发、Worker参数如何同步到边缘等。

在进入到终身学习的控制逻辑之前，我们先看看整体的调用流程：

```go
cmd/sedna-gm/sedna-gm.go/main()
manager.go/New() 读取GM配置文件。
manager.go/Start() 启动GM进程。
    - clientset.NewForConfig()：调用client-go生成了Sedna CRD client。
    - NewUpstreamController()：创建UpstreamController，每个GM进程有一个UpstreamController
    - uc.Run(stopCh)：启动一个for循环协程，来处理
        - pkg/globalmanager/controllers/upstream.go/syncEdgeUpdate() 
    - NewRegistry()：注册所有controller。
        - f.SetDownstreamSendFunc()
            - pkg/globalmanager/controllers/lifelonglearning/downstream.go
        - f.SetUpstreamHandler()
            - pkg/globalmanager/controllers/lifelonglearning/upstream.go/updateFromEdge()
        - f.Run()
    - ws.ListenAndServe()
```



这里的client收到了

![sednd-crd-client](./images/sednd-crd-client.png)



### 消息同步

for循环持续的监听`upstreamChannel`， 一旦有消息则通过`uc.updateHandlers[kind]`根据kind类型获取对应的handler。`uc.updateHandlers`是一个map，里面存储了协同推理、增量学习、联邦学习、终身学习对应的updateHandlers.

```
// syncEdgeUpdate receives the updates from edge and syncs these to k8s.
func (uc *UpstreamController) syncEdgeUpdate() {
   for {
      select {
      case <-uc.messageLayer.Done():
         klog.Info("Stop sedna upstream loop")
         return
      default:
      }

      update, err := uc.messageLayer.ReceiveResourceUpdate()
	  ...

      handler, ok := uc.updateHandlers[kind]
      if ok {
         err := handler(name, namespace, operation, update.Content)
         ...
      }
   }
}
```

```go
// ReceiveResourceUpdate receives and handles the update
func (cml *ContextMessageLayer) ReceiveResourceUpdate() (*ResourceUpdateSpec, error) {
   nodeName, msg, err := wsContext.ReceiveFromEdge()
   ...
}
```

`ReceiveFromEdge`提供一个阻塞的通道，来接收来自边缘节点LC发送的消息，消息的类型为`nodeMessage`。

`pkg/globalmanager/messagelayer/ws/context.go`

```go
// ReceiveFromEdge receives a message from edge
func ReceiveFromEdge() (nodeName string, msg model.Message, err error) {
	nodeMsg := <-context.upstreamChannel
	nodeName = nodeMsg.nodeName
	msg = nodeMsg.msg
	return
}
```



## LC: Local Controller

### 本地任务管理

增量、联邦、xx等

### 数据集管理

#### 触发器

### 模型管理

minio

### 消息代理


# Sedna Lib源码解析（Python）

## 

##



[^1]: https://www.redhat.com/en/topics/containers/what-is-a-kubernetes-operator