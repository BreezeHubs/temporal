# 分布式任务调度框架Temporal

github：https://github.com/temporalio/temporal  
SDK：https://github.com/temporalio/samples-go  https://docs.temporal.io/go  



Temporal为golang编写的 分布式任务调度框架 ，定位是`microservice orchestration platform`，提供工作流编排、C\S架构、状态查询等一系列功能

![image-20221231153409185](./resource/20221231153409185.png)

Activity：使用Temporal提供的各种语言的SDK（go、java、python、php等等）编写的代码逻辑  
Workflow：Activity集合，多个Activity构成一个Workflow，是调度的最小单位  
Workers：不同语言写的Workflow可以注册到对应语言的Workers中，Workers是代码的真正执行者  
Temporal Server：管理注册到自己的Workers，向Workers下发任务，监听任务状态等  
cli、web、SDK：任务的发起者、监控任务进度等  
cli、web：负责任务的监控、查询等  

>     【生产者】----------------->【调度】-----------------> 【消费者】  
>   cli、web、SDK----------------->Temporal Server----------------->Workers

------

### Workflow

**特点**  
持久性：一次Workflow Execution就是我们对Workflow Definition的执行，无论这个过程会持续几秒还是几年，这就是所谓的持久性  
可靠性：可靠性代表在Workflow Execution执行过程中对故障的响应能力。Temporal对于由于技术设施中断导致的故障具有很好的恢复性，可以保持Workflow Execution在中断时的状态，以及从最新的状态恢复执行  
可扩展性：可扩展性是指Temporal的负载能力。Temporal集群和Worker进程的设计和性质，让Temporal可以支撑数十亿个工作流同时执行  
支持异步调用：其实Workflow Execution就是再重复向Temporal平台发送指令和等待指令返回的过程  

**有状态的**  
Workflow Execution的状态分为Open和Closed两种：  
![image](https://user-images.githubusercontent.com/57607319/210150534-40cb8b77-acc0-4761-9a07-bf011cdcb0c0.png)  
只有Running状态属于Open，无论是正在处理流程还是在阻塞等待，状态都体现为Running  
  
- Closed包含六种状态，表示工作流已经结束，六种状态分别为：  
  Completed：Workflow Execution已经成功结束  
  Continued-As-New：定时Workflow到达定时周期时，开启新的Workflow，之前的Workflow会变成此状态  
  Terminated：Workflow Execution被终止  
  Cancelled：Workflow Execution被取消，意味着成功处理了一个取消请求  
  Failed：Workflow Execution发生异常并执行失败  
  Timed Out：Workflow Execution触发了超时限制  

上面说到一个Workflow Execution的执行时长是没有限制的，但是对于事件的个数和数据量是有限制的。所以还是尽量不要去写无限期运行的工作流。如果真的有无限期执行的需求，可以使用Continue-As-New功能，结束本工作流，并启动一个具有相同WorkflowId并且RunId不同的Workflow Execution  

**支持的配置参数**  
- 超时设置  
  ExecuteTimeout：Workflow的最大运行时间，包括失败后重试的时间。默认值是10年  
  RunTimeout：Workflow单次运行的时间，默认值为 ExecuteTimeout  
  TaskTimeout：从Worker从任务队列拉取到Workflow任务，到Worker开始执行Workflow的时间。如果超时，Server会认为Worker已经挂掉，会重新调度该Workflow给其他Worker，默认值10s  
 
- 重试策略  
  InitialInterval：第一次重试前，需要等待多久。无默认值  
  BackoffCeofficient：退避系数，表示多次重试时，下次等待的时间是上次的多少倍，默认值：2  
  MaximumnInterval：下次重试时，最大等待时间。默认值：100*初始等待时间  
  MaximumAttempts：做大重试次数，默认值：0，表示无限重试  
  Non-Retryable：表示Workflow遇到哪些Error后，不再进行重试  
  
**Workflow Id**  
一个Workflow，可由 命名空间，Workflow Id和Run id 唯一标识  
启动Workflow的时候，可以指定一个ID，这个ID一般采用业务级的ID，如一个要处理的客户的ID或订单ID  
- 多个Workflow使用相同ID时的策略配置  
  Allow duplicate failed only policy：只有前一个相同ID的Workflow失败后，才可以再启动下一个相同ID的Workflow  
  Allow duplicate policy：允许两个相同ID的Workflow同时运行。默认策略  
  Reject duplicate policy：任何时候，不允许有相同ID的Workflow  
  
**定时运行**  
启动Workflow的时候，可以设置为定时启动。如果到了下次运行Workflow的时候，但上次的Workflow还没执行完（可能任务执行耗时长，或由于失败后重试等原因），会跳过下次运行Workflow  
  
------

### Activity  
根据工作流的定义：工作流指业务过程的部分或整体在计算机应用环境下的自动化，是对工作流程及其各操作步骤之间的业务规则的抽象、概括描述,Activities可以理解为一个业务操作单元    
在Workflow执行过程中，会将Activity放入消息队列，由其他Worker获取后，执行该Activity，并将结果再返回给Workflow  

- 超时配置  
ScheduleToStart：表示Activity任务放到消息队列，到Worker获取到的超时时间。如果超时后，也不会触发重试。非有特殊原因，不要设置该值  
StartToClose：Activity实际执行超时时间。如果Activity执行时间不确定，最好按照最长时间设置。比如一个Activity可能需要2分钟、有时需要5分钟，那就设置为5分钟  
ScheduleToClose：从Activity放入消息队列，到Activity执行完成的时间  
Heartbeat：Activity和Server的心跳超时时间。在Activity运行需要较长时间时需要。用于Server检查执行Activity的Worker是否已经挂掉  

- 重试策略  
和Workflow的重试策略完全一致

- 执行时间超长  
如果一个Activity运行时间较长，最好设置一个心跳间隔超时。这样当执行Activity的Woker挂掉时，Server可以及时知道  

------

### Temporal集群整体架构
Temporal集群由Temporal Server和数据库服务组成  
Temporal Server包含：Fronted、Matching、History、Worker  
数据库支持：Mysql、PostgreSQL、Cassandra、ES    
![image](https://user-images.githubusercontent.com/57607319/210149853-5f2244db-6f58-4330-b452-dd3692be7a88.png)  
 
#### 前端组件（Frontend Service）  
前端组件是一个单点网关，提供 Proto API。可以接受来自浏览器、tctl（Temporal的命令行工具）、以及业务方的调用请求。  
前端组建主要用于 接口限速、授权认证、校验和请求路由  

#### 记录服务（History service）   
记录服务用于记录Workflow的执行状态，并且支持横向拓展  

#### 匹配服务（Matching service）  
匹配服务用于管理任务队列，及任务分发，并且支持横向拓展  

#### 后台服务（Worker service）  
后台服务主要用于维护拷贝队列和执行一些Temporal服务自己的Wrokflow  

#### 数据库  
数据库保存了用于分发的任务信息、以及Workflow的执行状态、命名空间元数据，前端可视化配置  

------  

### 示例

模拟场景是：延时1小时向用户发送短信和邮件

#### 1.编写`Activity`代码

  ```go
  type MessageRequest struct {
     PhoneNum string
     Content  string
     Tags     []string
  }
  
  func SendMessage(ctx context.Context, mr MessageRequest) (mresp MessageResponse, error) {
     fmt.Printf(
        "\nSending message to %s \n Content is %s \n ",
        mr.PhoneNum,
        mr.Content,
     )
     return nil, nil
  }
  
  type EmailRequest struct{
      From string 
      To string 
      Content stirng 
  }
  
  func SendEmail(ctx context.Context, er EmailRequest) error{
         fmt.Printf(
        "\nSending email to %s \n Content is %s \n ",
        er.To,
        er.Content,
     )
     return nil, nil
  }
  ```

  Activity代码的限制很少，什么样的函数都可以成为Activity，甚至结构体上的方法也可以

  
#### 2.编写`Workflow`代码

   ```go
   func SendMessageWorkflow(ctx workflow.Context, msq MessageRequest, er EmailRequest) error {
      options := workflow.ActivityOptions{
         StartToCloseTimeout: time.Minute,
      }
      ctx = workflow.WithActivityOptions(ctx, options)
      
      //工作流
      //1. 先执行SendMessage
      err := workflow.ExecuteActivity(ctx, SendMessage, msq).Get(ctx, nil)
      if err != nil {
         return err
      }
      //2. 再执行SendEmail
      err = workflow.ExecuteActivity(ctx, SendEmail, msq).Get(ctx, nil)
      if err != nil {
         return err
   }
   return nil
}
```

其实Workflow也是一段函数，但是Workflow里的代码有些限制，比如不能和外部系统交互（读写文件、访问网络）等，所以需要划分Activity的原因就是Activity没有这种限制


#### 3.启动 Workers

   ```
   func main() {
       //连接到Temporal Server进行注册
      c, err := client.NewClient(client.Options{})
      if err != nil {
         log.Fatalln("unable to create Temporal client", err)
      }
      defer c.Close()
      
      w := worker.New(c, app.TaskQueue, worker.Options{})
      w.RegisterWorkflow(app.SendMessageWorkflow)
      w.RegisterActivity(app.SendMessage)
      w.RegisterActivity(app.SendEmail)
      
      //Start listening to the Task Queue
      err = w.Run(worker.InterruptCh())
      if err != nil {
      log.Fatalln("unable to start Worker", err)
   }
}
```

启动一个Worker，将自身注册到Temporal Server，再把Workflow和Activity注册到它内部，这个Worker就成功启动了。如果结合K8s，还能编排不同的Worker，比如：给Worker扩缩容，监控Worker等


#### 4.启动`Template Server`
  根据官方github文档，使用docker启动服务，且可以配合`Temporal Web UI `使用

  

#### 5.发起任务

  ```go
  func main() {
     //连上Template Server
     c, err := client.NewClient(client.Options{})
     if err != nil {
        log.Fatalln("unable to create Temporal client", err)
     }
     defer c.Close()
     
     options := client.StartWorkflowOptions{
        TaskQueue: app.TaskQueue,
     }
     r1 := app.MessageRequest{
        ...
     }
     r2 := app.EmailRequest{
        ...
     }
     we, err := c.ExecuteWorkflow(context.Background(), options, "SendMessageWorkflow", transferDetails)
     if err != nil {
        log.Fatalln("error starting SendMessageWorkflow", err)
     }
     fmt.Println(we.GetID(), we.GetRunID())
  }
  ```

  最后使用SDK提供的方法启动Workflow，TaskQueue用于路由到Workers，再提供Workflow的函数名，就可以调用成功了


内容参考链接：  
https://www.pudn.com/news/62df6aa8864d5c73ac07eb3f.html  
http://timd.cn/temporal/concepts/activity/  
https://blog.csdn.net/csdnYF/article/details/120367446  
https://cloud.tencent.com/developer/article/1982039?from=15425  
https://cloud.tencent.com/developer/article/2018460?from=15425  

