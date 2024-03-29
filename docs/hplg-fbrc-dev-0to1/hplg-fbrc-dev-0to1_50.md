# 12.1 MVC 是什么：合理的设计我们的应用

## 目标

1.  理解 MVC 架构的概念
2.  能够在项目中应用 MVC 架构模式

## 任务实现

## 12.1 MVC

应用程序开发人员在工作中都需要考虑一些问题，如：

*   如何简化开发
*   如何降低应用程序的耦合性
*   如何提高代码的重用性
*   如何提高应用程序的可扩展性及维护性
*   ......

1979 年，由 Trygve Reenskaug 在 Smalltalk-80 系统上首次提出了 MVC 的概念，主要核心就是由专业的人做专业的事。MVC 模式代表 Model（模型）－View（视图）－Controller（控制器）模式。这种模式主要应用于应用程序的分层开发。将应用程序中显示什么数据、数据由谁处理、怎么处理进行分离，可以由不同的开发人员专注于自己的领域，而无需关注其它。

### 12.1.1 MVC 架构概念

MVC 将应用程序划分为三种组件，并明确定义了它们之间的相互作用：

*   **模型（Model）：**用于封装与应用程序的业务逻辑相关的数据以及对数据的处理法。"Model"有对数据直接访问的权力，例如对数据库的访问。"Model"不依赖"View"和"Controller"，也就是说， Model 不关心它会被如何显示或是如何被操作。但是 Model 中数据的变化一般会通过一种刷新机制被公布。为了实现这种机制，那些用于监视此 Model 的 View 必须事先在此 Model 上注册，从而，View 可以了解在数据 Model 上发生的改变
*   **视图（View）：**能够实现数据有目的的显示。在 View 中一般没有程序上的逻辑。为了实现 View 上的刷新功能，View 需要访问它监视的数据模型（Model），因此应该事先在被它监视的数据那里注册
*   **控制器（Controller）：**可以将控制器理解为一个桥梁，处理事件并作出响应。负责从视图读取数据，控制用户输入，并向模型发送数据

### 12.1.2 MVC 架构优缺点

不论是什么技术，使用什么理念，尤其是能够经过长时间的应用实践，都有其显著的优点，有优点，也必然存在一些缺点，下面我们来看一下 MVC 模式的优缺点。

#### 优点

*   高内聚性 － MVC 可以在控制器上对相关操作进行逻辑分组。特定模型的视图也可以组合在一起。
*   耦合性低 － MVC 框架的本质是模型，视图或控制器之间的耦合较低
*   重用性高 －允许使用各种不同样式的视图来访问同一个服务器端的代码；将数据和业务规则从表示层分开，可以最大化的重用代码。模型也有状态管理和数据持久性处理的功能
*   提高可扩展性与可维护性 － 由于分离视图层和业务逻辑层也使得 WEB 应用更易于维护和修改
*   利于项目的管理 － 由于不同的层各司其职，每一层不同的应用具有某些相同的特征，有利于通过工程化、工具化管理程序代码。控制器也提供了一个好处，就是可以使用控制器来联接不同的模型和视图去完成用户的需求
*   简化了分组开发－不同的开发人员可同步开发应用项目中的视图、控制器逻辑和业务逻辑

#### 缺点

*   增加系统结构及代码量－对于简单的界面，严格遵循 MVC，使模型、视图与控制器分离，会增加结构的复杂性及相应的代码量
*   不适用于中小型应用项目 － 将 MVC 应用到规模并不是很大的应用程序通常会降低开发效率，增加开发成本
*   显著的学习曲线 － 关于掌握多种技术的知识成为常态。使用 MVC 的开发人员需要熟练掌握多种技术

### 12.1.3 MVC 架构的实际应用

MVC 不是一种技术，而是一种理念。下面是一个通过 JavaScript 所实现的基于 MVC 模型，在这个简短的代码中就写成了一个具有完整 MVC 架构模式概念的示例。

```go
/** 模拟 Model, View, Controller */
var M = {}, V = {}, C = {};

/** Model 负责存储数据 */
M.data = "hello world";

/** View 负责将数据显示在屏幕上 */
V.render = (M) => { alert(M.data); }

/** Controller 作为一个 M 和 V 的桥梁 */
C.handleOnload = () => { V.render(M); }

/** 在网页加载时调用 Controller */
window.onload = C.handleOnload; 
```