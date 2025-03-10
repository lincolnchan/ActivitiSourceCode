# Activiti 如何解析BPMN文件的呢

## 文档解析技术

### DOM模型

- 优点: 文档解析的时候允许客户端编辑和更新XML文档内容,并可以额随机访问文档中定义的元素数据

- 缺点: 文档解析的时候会将需要解析的XML文档内容一次性加载到内存,进而映射为Document对象的树形结构

- 因为是一次性进内存,当解析大文件的时候,内存占用率大元素遍历查找慢,很容易造成性能问题

### 流模型

-  优点:该方式解析流程文档的时候,每一次操作只会将需要解析的节点放置到内存中,从头部开始,读取一段,处理一段。这样就解决了文档对象模型可能引发的问题, 该方式内存占用率少,性能大幅提升

- 缺点该方式解析流程文档的时候是只读的,不可以编辑, 文件流只能前进不能回退

- 由于Activiti将流程文档完全交给客户端定义和使用 ,所以最终生成XML大小 完全由用户决定,是不可控因素
  软件设计之初应该尽量屏蔽这种不可控的风险, 所以Activiti选择 流模型 进行文档解析

流模型解析技术有 SAX 和 STAX

sax 是 推模型, stax是啦模型

1)推模型:
    SAX 处理方式,是一种事件驱动机制,文档解析的时候,每当元素发现一个元素节点就会触发相应的事件,
    因此客户端需要编写监听触发事件的处理程序, 使用该方式解析流程文档无疑增加了客户端操作的复杂度,不灵活
2)拉模型
    STAX 处理方式, 文档解析时 客户端可以定制自己感兴趣的节点主动从读取器(Reader)中进行拉去,
    选择性的处理节点事件,对比推模型,该方式的灵活性大大提高











基于上述方案 ,Activiti最终选用STAX方式解析流程文档, STAX 提供了两套 xml 解析的API
分别是基于指针的 基于迭代器的API  ,解析原理类似      Activiti用的是基于指针方式

Activiti在5.12.1 版本中开始使用STAX ,之前使用的是SAX 解析

可以把STAX模型理解为一个不可回流的流水管道,
该管道用于存储流程定义信息, 读取器可以理解为水龙头, 每次读取器读取文档内容的时候
客户端对于读取的流进行处理,也可以不理会

STAX模型中  定义很多种事件类型的名称, 含义 和值

名称
START_ELEMENT                   元素的开始,              1
END_ELEMENT                     元素的结尾               2
PROCESSING_INSTRUCTION          处理指令                 3

STAX 方式解析XML文档的核心点在于程序可以根据当前的标记或者事件类型做出相应的处理,

    元素解析功能架构设计
    
    
    
    元素定义层     绘制
        元素定一层完全交给客户端,开发人员可以结合自己的业务场景组装一系列元素,
        最终绘制好流程图文档
    元素解析层    解析器初始化     --> 解析器解析,  -->结果映射
        进行部署之后, ,此层负责定位流程文档,初始化元素解析器(包括子元素) 加载自定义元素解析器,查找元素解析器
        最终将需要解析元素以及属性进行解析,并封装映射为引擎中的一个个实体对象
    基础支撑层   元素配置 --  元素解析器, , 元素属性承载类, ,自定义元素解析器
        例如DB连接管理事务管理等.Activiti 将这些共用的组件抽取出来作为基础模块
        为 上面2层进行 基础服务支撑
    
    父类都是BaseBpmnXMLConverter
    
        endEvent    EndEvent     EndEventXMLConverter
        startEvent   StartEvent  StartEventXMLConverter
        特殊的
        dataObject   ValuedDataObject   ValuedDataObjectXMLConverter
        subProcess    SubProcess                SubProcessParser
        signal        Signal                    SignalParser
        resource      Resource                  ResourceParser
        process       Process                   ProcessParser
        participant     Pool                 ParticipantParser
        multiInstaceLoopCharacteristics   MultiInstaceLoopCharacteristics  MultiInstanceParser
    
    
    
    一些元素承载类公有的()
            1)HasExtensionAttributes
            这是扩展属性的() , 因为所有的元素属性承载类 都实现了该接口 ,所以所有的元素都是可以扩展的
              setAttributes()
            2) BaseElement
             封装了 id , xmlRowNumber,xmlColumnNumber等属性 以及克隆baseElement实例的()  clone()BaseElement对象
            3) FlowElement
                name()  documentation()  executionListeners()  克隆FlowElement对象的抽象()
            4) FlowNode
                asynchronous  notExclusive    incomingFlows和outgoingFlow ()
            5) Activity
                    defaultFlow  loopCharacteristics
