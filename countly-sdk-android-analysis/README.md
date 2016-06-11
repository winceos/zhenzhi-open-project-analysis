$countly-sdk-android 源码解析
====================================
>  项目地址：[countly-sdk-android](https://github.com/Countly/countly-sdk-android)，分析的版本：[16.02.01 Release](https://github.com/Countly/countly-sdk-android/commit/c105d06e573703d9e29d5c92068a41befa36f43f)，Demo 地址：[countly-sdk-android-demo](https://github.com/Labmem003/zhenzhi-open-project-analysis/tree/master/countly-sdk-android-demo)    
 分析者：[振之](https://github.com/Labmem003)，分析状态：未完成


###1. 功能介绍  
Countly 是一款类似于友盟的移动&Web应用通用的实时统计分析系统，专注于易用性、扩展性和功能丰富程度。不同之处是Countly是开源的，任何人都可以将Countly客户端部署在自己的服务器并将开发工具包整合到他们的应用程序中。比友盟要简洁干净，而且这下数据和程序都完全处于自己控制之下了。

这里分析的是其Android端的sdk, 以了解和学习移动应用统计类的工具收集App的使用情况和终端用户的行为的机制。主要功能包括App基本数据收集、自定义事件记录、崩溃报告。

###2. 详细设计
###2.1 类详细介绍
类及其主要函数功能介绍、核心功能流程图，流程图可使用 [Google Drawing](https://docs.google.com/drawings)、[Visio](http://products.office.com/en-us/visio/flowchart-software)、[StarUML](http://staruml.io/)。  
###2.2 类关系图
类关系图，类的继承、组合关系图，可是用 [StarUML](http://staruml.io/) 工具。  

**完成时间**  
- 根据项目大小而定，目前简单根据项目 Java 文件数判断，完成时间大致为：`文件数 * 7 / 10`天，特殊项目具体对待  

###3. 流程图
主要功能流程图  
- 如 Retrofit、Volley 的请求处理流程，Android-Universal-Image-Loader 的图片处理流程图  
- 可使用 [Google Drawing](https://docs.google.com/drawings)、[Visio](http://products.office.com/en-us/visio/flowchart-software)、[StarUML](http://staruml.io/) 等工具完成，其他工具推荐？？  
- 非所有项目必须，不需要的请先在群里反馈  

**完成时间**  
- `两天内`完成  

###4. 总体设计
整个库分为哪些模块及模块之间的调用关系。  
- 如大多数图片缓存会分为 Loader 和 Processer 等模块。  
- 可使用 [Google Drawing](https://docs.google.com/drawings)、[Visio](http://products.office.com/en-us/visio/flowchart-software)、[StarUML](http://staruml.io/) 等工具完成，其他工具推荐？？  
- 非所有项目必须，不需要的请先在群里反馈。  

**完成时间**  
- `两天内`完成  

###5. 杂谈
该项目存在的问题、可优化点及类似功能项目对比等，非所有项目必须。  

**完成时间**  
- `两天内`完成  

###6. 修改完善  
在完成了上面 5 个部分后，移动模块顺序，将  
`2. 详细设计` -> `2.1 核心类功能介绍` -> `2.2 类关系图` -> `3. 流程图` -> `4. 总体设计`  
顺序变为  
`2. 总体设计` -> `3. 流程图` -> `4. 详细设计` -> `4.1 类关系图` -> `4.2 核心类功能介绍`  
并自行校验优化一遍，确认无误后将文章开头的  
`分析状态：未完成`  
变为：  
`分析状态：已完成`  

本期校对会由专门的`Buddy`完成，可能会对分析文档进行一些修改，请大家理解。  

**完成时间**  
- `两天内`完成  

**到此便大功告成，恭喜大家^_^**  