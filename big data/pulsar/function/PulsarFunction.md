Apache Pulsar Pulsar 是一个多租户、高性能的服务间消息传输解决方案，数据持久化依赖 Apache BookKeeper 实现，支持多租户、低延时、读写分离、跨地域复制、快速扩容、灵活容错等特性。本文是 Apache Pulsar 技术系列文章中的一篇，主要介绍 Apache Pulsar Function的设计原理和一些特性。

一、 Pulsar Functions 简介
经验告诉我们绝大部分数据处理应用场景都是简单、轻量级的，比如：

简单的 ETL（提取、转化、加载）
实时聚合
事件路由
Pulsar Functions 不仅能处理上述用例，还大大简化了部署，因而在减少了开发时间的同时实现了开发人员产出的最大化。

Pulsar function产生的初衷并不是为了替代spark或者flink，而是为了是用户在处理简单的stream操作时能够直接使用pulsar function完成，不需要编写生产者消费者等复杂的逻辑，同时pulsar function能够自动负载均衡，合理的分配机器资源执行不同的function。

如图所示，用户可以输入多个 topic，每输入一个 topic 都可以向用户自定义的 Pulsar Function 发送数据，Pulsar Function 的处理单元处理完成之后把结果发送到 Output topic，其中一些辅助性的 topic 可以进行日志或消息的收集。



二、 Pulsar Functions 示例
一个简单的wordCount function 如下：

package org.example.functions;
 
import org.apache.pulsar.functions.api.Context;
import org.apache.pulsar.functions.api.Function;
 
import java.util.Arrays;
 
public class WordCountFunction implements Function<String, Void> {
    // This function is invoked every time a message is published to the input topic
    @Override
    public Void process(String input, Context context) throws Exception {
        Arrays.asList(input.split(" ")).forEach(word -> {
            String counterKey = word.toLowerCase();
            context.incrCounter(counterKey, 1);
        });
        return null;
    }
}
将代码打包成jar包后可以通过命令行的形式提交到pulsar集群运行，命令行参数如下：

$ bin/pulsar-admin functions create \
  --jar target/my-jar-with-dependencies.jar \
  --classname org.example.functions.WordCountFunction \
  --tenant public \
  --namespace default \
  --name word-count \
  --inputs persistent://public/default/sentences \
  --output persistent://public/default/count
三、 Pulsar Functions 总体架构
3.1 function worker


pulsar function由worker进行管理，worker主要承担了以下几个职责：

处理用户对function的CRUD请求
保证function元数据在所有的worker上的最终一致性（这点在后面小节有提到）
worker中的master负责调度function
对function进行生命周期的管理
目前pulsar支持两种形式的worker部署，worker可以和broker一起部署或者单独部署。

3.2 worker内部组件及特殊topic介绍




Worker中主要包含四个组件，三个topic：

其中三个topic分别为:

function metaData topic : 负责存储所有的function元数据信息
assignment topic ： 负责存储function的调度信息，该topic只能够由master worker写入
coordination topic : 空topic，负责worker的leader选举
四个组件分别为：

function metaData manager: 负责存储系统内所有的function元数据变动信息，同时负责处理metaData topic的写入冲突以及触发function调度
scheduler manager: 负责存储系统内的assignment信息，同时负责触发function runtime manager
function runtime manger: 负责处理function的运行管理，通过grpc管理function运行实例
membership manager: 负责worker的leader选举
四 Pulsar Funtions 运行原理
本大节中主要讲解Pulsar Function的运行流程，包括提交、调度、运行三大过程

4.1 提交流程 
4.1.1 function验证
提交 function 的流程主要由function metaData manager来处理。

提交流程中用户需要提交 functionConfig，conifg中包括包括租户、命名空间、名称等等信息,另外还需要将用户自定义的代码打包上传。

用户提交 function 后，系统将会对 function 进行检查，确保用户有权限提交此 function 到特定的命名空间和租户。

如果使用 Java 语言，提交时manager就会加载这些类，确保指定的类在 JAR 文件中，因此出现错误时，用户可以很快收到提示消息，而不用自行查看错误日志。

public class FunctionConfig {
       private String tenant;
       private String namespace;
       private String name;
       private String className;
       private Collection<String> inputs;
       private String output;
       private ProcessingGuarantees processingGuarantees;
       private Map<String, Object> userConfig;
       private Map<String, Object> secrets;
       private Integer parallelism;
       private Resources resources;
       ...
}
正确提交之后下一步function metaData manager会复制代码文件到 BookKeeper，并将提交信息中的所有参数以protobuf形式表示为 functionMetaData并写到特定的function metaData topic中，function metaData信息示例如下：

message FunctionMetaData {
    FunctionDetails functionDetails ;
    PackageLocationMetaData packageLocation;
    uint64 version ;
    uint64 createTime;
    map<int32 , FunctionState> instanceStates ;
    FunctionAuthenticationSpec functionAuthSpec ;
}
4.1.2 function冲突处理
Funtion meta manager中包含一个function metatopic tailer用来读取系统中meta data topic的所有数据，同时它还承载了避免数据冲突的职责。



在function meta manager中处理function的变更请求时需要经过以下几步：

复制当前状态：将当前function复制一份，记录当前的版本号

进行状态合并更新

增加副本中版本号

将数据写入 MetaData Topic

Tailer 进行数据读取和验证

如果没有冲突，则实际更新本地状态

上面的整个流程是在单台 function worker 的情况下，但是在多台 function worker 中就有可能会出现冲突。

在多个 function worker 运行的场景下，当对某一个 function 进行并发更新时，会出现冲突的情况。当出现冲突时，其解决方式是使用 First Writer Win 的策略，即当第一个请求被成功接收之后，其它请求将会被拒绝。

在上图中，由于右侧worker先写入metaData，worker2 在写入之后tailer进行验证的时候发现之前已经有过version3的版本号存在，那么该次更新将不会生效。

4.1.3 提交流程优劣点分析
优点
1、function可以提交到任意 worker

2、状态最终一致（first write wins)

3、meta data topic可以充当为一个audit log使用

缺点
1、meta data topic 数据会增长却没有相关数据压缩处理

2、由于worker重启时都需要读取所有的meta data，worker启动的时间会非常的长

3、每个worker都存储所有的function信息是一种资源的浪费

4.2 调度流程
4.2.1 worker master选举
worker中会选举出一个master来承担function的调度职责，maste的选取过程巧妙的使用了pulsar中的failover订阅模式，该模式下pulsar能够保证同一时间内只有一个consumer处于active状态

worker的选举流程就是在worker启动时订阅coordination topic，处于active的状态的worker可以成为master，coordination topic为一个空的topic。





4.2.2 调动触发条件
Function worker 会在以下状态时开始一次调度。

Function CRUD 操作：创建/更新/删除

Worker 变动：如创建新 worker、leadership 发生变化等

同时这里又有一个新的topic用来存储assignment信息，由master进行写入，其他的worker订阅该topic即能够知道自身需要执行的操作。



4.3 执行流程






在上图中，scheduler中的assignment tailer 监听到 topic 中的变化，此时就会将此动作变化传递给 function runTime manager。同时借由 Spawner 进行一些列后续操作。

Spawner 是使用 Functions 时的一个抽象执行环境，通过查阅源码我们发现目前pulsar支持三种runtime即

thread：通过线程运行运行function
process: 通过进程运行function
kubenetes: 通过容器运行function
同时spawner 也具有 Functions 生命周期管理的功能，两者之间的通信由 GRPC 通道执行

五 Pulsar IO 与 Pulsar Function
通过对pulsar function的了解我们知道，当function为identity function时，function架构可以很好的作为sink 以及 source的实现方式，实际上pulsar确实是这么做的。

Pulsar IO 充分利用了现有的 Pulsar Functions 框架，作为 Pulsar IO 的组成部分，source 和 sink 拥有 Pulsar Functions 的所有优势：

优势
详细介绍
执行灵活性	Source 和 sink 都可以作为现有集群的一部分或作为本地进程来运行 。
并发性	要增加 source 或 sink 的吞吐量，只需添加简单的配置即可运行更多 source 和 sink 实例。
负载均衡	当 source 和 sink 以集群模式运行时，能达到负载均衡。
容错、监控、metrics	如果 source 和 sink 以“集群”模式运行，则作为 Pulsar function 框架一部分的 worker 服务将自动监控已部署的 source 和 sink。当节点发生故障时，将自动重新部署 source 和 sink 到运作节点，并自动收集 metrics。
动态更新	动态更新多项配置，如：单个 connector 的并行性、源代码、输入/输出 topic 等。
数据本地化	由于 broker 为 topic 的读写请求提供服务，因此在 broker 附近运行 source 和 sink 可以减少网络延迟和网络带宽的使用率。


六 Function Mesh
Function Mesh 是一组 function 的集合，它能让多个 function 在一起协调完成数据处理目标，并且每个 function 有各自明确任务和被定义好的 stage。



如上图所示，在 Function Mesh 之前，我们所使用的都是单个 Pulsar Function。引入 Function Mesh 之后，多个 function 就有了关联和数据联络，最后产生想要的结果。

6.1 实现方案一：Pulsar原生支持
目前 Pulsar 提供了命令行工具，可用来管理单个 function，如需要多个function组合，则需要在 Pulsar 命令行工具启动 function 1 到 function 6 才行，如此会带来管理的重复和复杂度；同时很难追踪 function，无法将它们当作组合处理；也无法很好地了解各自 function 的上下游和处理顺序。

面对上述提到的几个问题，pulsar针对性提出解决方案，详情可参见https://github.com/apache/pulsar/wiki/PIP-66:-Pulsar-Function-Mesh。主要的思路是在 Pulsar 中提供对 Function Mesh 的原生支持，即通过 Pulsar 命令行提交 Function Mesh，并在 Function Mesh YAML 配置文件中定义所包含的每个 function 参数及组织关系、输入和输出来源等。

下图是根据上述思路产生的一个 Function Mesh 调度方案，最大化利用已有的 Pulsar Function 调度机制实现 Function Mesh 的设计目标，当然也引入了 FunctionMeshManager 来管理 Function Mesh 的元数据。



6.2 实现方案二：基于 Kubernetes
基于 Kubernetes 实现 Function Mesh 是非常有意义和价值的事情，因为k8s对容器有强大的管理功能。可以直接利用 Kubernetes 命令行工具创建CRD，CRD类型就是 FunctionMesh，function之间的连接关系可在CRD中定义。

在该模式下，Function Mesh 不再是运行在 Pulsar 上，而是运行在 Kubernetes 云平台之上。基于 Kubernetes 的 Function Mesh 调度方案，如下图所示：



6.3 方案对比：Pulsar vs Kubernetes
对比基于 Pulsar 和基于 Kubernetes 的 Function Mesh 实现方案，主要有几点思考：

如果能够利用 Kubernetes 的调度能力是十分有利的事情，对不同任务的调度是 Kubernetes 的专项能力，并且也能提供高可用、容错性保障等功能。

在云环境中，Function 成为一等公民，地位与 Pulsar 提供的服务一样。

如果能将 Function 从 Pulsar 当中抽离出来，那它则具备与其他消息系统进行数据对接、处理的潜力，类似 AWS 提供的 Lamba Function，也方便用户进行事件驱动型的模式设计。

目前function mesh还在内测阶段，代码尚未开源，如果想体验相关功能欢迎联系pulsar官方。

写在最后
目前TDbank上线了对Pulsar的支持，公司内也有很多业务正在使用和调研Pulsar；同时，腾讯云TDMQ、计平、数平等多个团队也在一起共建Pulsar，对Pulsar感兴趣的小伙伴，欢迎加入下面的企业微信群一起交流。