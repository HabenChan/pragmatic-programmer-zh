# Actors 和进程
<!-- 2020.04.12 -->

> _没有作家，就不会有故事。_
> _没有演员，故事就不可能活灵活现。_
>
> _-- 安吉·玛丽·德尔桑特_

Actors 和进程提供了实现并发的有趣方法，而不需要同步访问共享内存的负担。

然而在讲解它们之前，我们需要先定义一下我们的概念。这听起来会很学术。但是别担心，我们将在不久之后进行研究。

- 一个 actor 是一个独立的虚拟处理器，有自己的本地（私有）状态。每个 actor 都有一个邮箱。当一个消息出现在邮箱中，而执行器是空闲的时候，它就会启动并处理这个消息。当它完成处理后，它会处理邮箱中的另一条消息，或者，如果邮箱是空的，它会回到睡眠状态。

    当处理一个消息时，一个 actor 可以创建其他 actor，向它所知道的其他 actor 发送消息，并创建一个新的状态，当下一个消息被处理时，该状态将成为当前状态。

- 进程通常是一个比较通用的虚拟处理器，通常由操作系统实现，以方便并发。进程可以被约束（按照惯例）表现为 actor，这就是我们这里所说的进程的类型。

## Actors 只能是并发的
在 actors 的定义中，有一些东西是缺失的。

- 没有任何一个东西是可以控制的。没有任何东西可以安排下一步发生的事情，或者协调信息从原始数据到最终输出的传递。

- 系统中唯一的状态被保存在消息和每个 actor 的本地状态中。消息不能被检查，除非被接收者读取，否则无法检查，而本地状态在 actor 之外是无法访问的。

- 所有的消息都是单向的，没有回复的概念。如果你想让 actor 返回一个响应，你在发送的消息中包含自己的邮箱地址，它就会（最终）将响应作为另一个消息发送至该邮箱。

- 一个 actor 处理每个消息到完成，并且每次只处理一个消息。

因此，actors 是并发地、异步地执行，而且什么都不共享。如果你有足够的物理处理器，你可以在每个处理器上运行一个 actor。如果你只有一个处理器，那么一些运行时可以处理它们之间的上下文切换。无论哪种方式，在 actors 上运行的代码都是一样的。

---
## 提示 59 在没有共享状态的情况下使用 Actors 进行并发
---

## 一个单独的 Actor
让我们用 actors 来实现我们的晚餐。在这种情况下，我们有三个（顾客、服务员和馅饼盒）。

整个消息流如下所示：

- 我们会告诉顾客他们饿了

- 作为回应，他们会向服务员要馅饼

- 服务员会让馅饼盒给顾客拿一些馅饼

- 如果馅饼盒里有一块馅饼，它会把它寄给顾客，并通知服务员把它加到帐单上

- 如果没有馅饼，馅饼盒会告诉服务员，服务员会向顾客道歉。

我们选择使用 Nact 库在 JavaScript 中实现代码。我们在此基础上添加了一个小包装器，允许我们将 actors 编写为简单对象，其中键是它接收的消息类型，值是在接收到特定消息时运行的函数。（大多数 actor 系统都有类似的结构，但细节取决于宿主语言。）

让我们从顾客开始。客户可以收到三条信息：

- 你饿了（由外部上下文发送）

- 桌子上有馅饼（馅饼盒发送）

- 对不起，没有馅饼（服务员发送的）

这里是代码：

```js
const customerActor = {
    'hungry for pie': (msg, ctx, state) => {
    return dispatch(state.waiter, { type: "order", customer: ctx.self, wants: 'pie' })
  },

  'put on table': (msg, ctx, state) =>
    console.log(`${ctx.self.name} sees "${msg.food}" appear on the table`),

  'no pie left': (_msg, ctx, _state) =>
    console.log(`${ctx.self.name} sulks...`)
}
```

有趣的案例是，当我们收到一个 ''饿了想要一个馅饼'' 的消息，然后给服务员发了一条消息，在这个时候，我们就会把这个消息发给服务员。(我们很快就会看到顾客是如何知道服务员的行为者的)。
下面是服务员的代码。

```js
const waiterActor = {
    "order": (msg, ctx, state) => {
    if(msg.wants == "pie") {
        dispatch(state.pieCase, { type: "get slice", customer: msg.customer, waiter: ctx.self })
    } else {
        console.dir(`Don't know how to order ${msg.wants}`);
    }
  },

  "add to order": (msg, ctx) =>
    console.log(`Waiter adds ${msg.food} to ${msg.customer.name}'s order`),

  "error": (msg, ctx) => {
    dispatch(msg.customer, { type: 'no pie left', msg: msg.msg });
    console.log(`\nThe waiter apologizes to ${msg.customer.name}: ${msg.msg}`)
  }
}
```

当它收到来自客户的 "想要" 消息时，它会检查该请求是否为馅饼的请求。如果是，它就向馅饼箱发送一个请求，同时传递对自己和客户的引用。

馅饼盒子有状态：它所持有的所有馅饼片的数组。当它收到服务员发来的 "get slice"消息时，它会查看它是否有剩余的馅饼。如果有，它就会把这片馅饼传给顾客，告诉服务员更新订单，最后返回一个更新的状态，里面少了一块馅饼。下面是代码。

```js
const pieCaseActor = {
    'get slice': (msg, ctx, state) => {
      if(state.slices.length == 0) {
          dispatch(msg.waiter, { type: 'error', msg: "no pie left", customer: msg.customer })
          return false
      } else {
          var slice = state.slices.shift() + "pie slice"
          dispatch(msg.customer, { type: 'put on table', food: slice })
          dispatch(msg.waiter, { type: 'add to order', food: slice, customer: msg.customer })
          return state
    }
  }
}
```

虽然你经常会发现 actors 是由其他 actors 动态启动的，但在我们的例子中，我们将保持简单，手动启动我们的 actors 。我们还将给每个角色传递一些初始状态。

- 我们将给馅饼箱子提供它所包含的馅饼片的初始列表
- 我们会给服务员提供一个参考，就是那个馅饼的箱子
- 我们会给客人提供服务员的参照物

```js
const actorSystem = start()

let pieCase = start_actor(
    actorSystem,
  'pie-case',
  pieCaseActor,
  { slices: ["apple", "peach", "cherry"] }
)

let waiter = start_actor(
    actorSyatem,
  'waiter',
  waiterActor,
  { pieCase: pieCase }
)

let c1 = start_actor(
    actorSyatem,
  'customer1',
  customerActor,
  { waiter: waiter }
)

let c2 = start_actor(
    actorSyatem,
  'customer2',
  customerActor,
  { waiter: waiter }
)
```

最后，我们终于把它分开了。我们的顾客很贪心。顾客一要三片馅饼，顾客二要两片。

```js
dispatch(c1, { type: 'hungry for pie', waiter: waiter });
dispatch(c2, { type: 'hungry for pie', waiter: waiter });
dispatch(c1, { type: 'hungry for pie', waiter: waiter });
dispatch(c2, { type: 'hungry for pie', waiter: waiter });
dispatch(c1, { type: 'hungry for pie', waiter: waiter });

sleep(500)
  .then(() => {
    stop(actorSystem);
  })
```

当我们运行它的时候，我们可以看到 actors 在交流，你看到的顺序很可能不一样。

```
$ node index.js
customer1 sees "apple pie slice" appear on the table
customer2 sees "peach pie slice" appear on the table
Waiter adds apple pie slice to customer1's order
Waiter adds peach pie slice to customer2's order
customer1 sees "cherry pie slice" appear on the table
Waiter adds cherry pie slice to customer1's order

The waiter apologizes to customer1: no pie left
customer1 sulks…

The waiter apologizes to customer2: no pie left
customer2 sulks…
```

## 没有显式并发
在 actor 模型中，不需要写任何代码来处理并发，因为没有共享状态。也不需要编写显式的端到端 "做这个，做那个 "的逻辑，因为 actors 根据接收到的消息自行解决。

也没有提到底层架构。这组组件在单处理器、多核或多网络机器上的工作效果都很好。

## Erlang 创造了舞台
Erlang 语言和运行时是 actor 实现的很好的例子（尽管 Erlang 的发明者没有读过最初的 Actor 论文）。Erlang 把 actor 称为进程，但它们不是常规的操作系统进程。相反，就像我们一直在讨论的 actor 一样，Erlang 进程是轻量级的（你可以在一台机器上运行数百万个），它们通过发送消息来进行通信。每个进程与其他进程是隔离的，所以不存在状态共享。

此外，Erlang 运行时实现了一个监督系统，它可以管理进程的生命周期，在出现故障时可能会重启一个进程或一组进程。而且 Erlang 还提供了热加载功能：你可以在不停止该系统的情况下替换正在运行的系统中的代码。而且Erlang 系统运行着一些世界上最可靠的代码，经常引用九死一生的可用性。

但是，Erlang（和它的后代 Elixir ）并不是唯一的，大多数语言都有 actor 实现。考虑在你的并发实现中使用它们。

## 相关内容包括

- 话题 28 [_解耦_](../Chapter5/解耦.md)
- 话题 36 [_黑板_](./黑板.md)
- 话题 30 [_转换编程_](../Chapter5/转换编程.md)

## 挑战

- 你目前是否有使用相互排斥来保护共享的数据。为什么不试试用 actors 编写相同代码的原型？

## 练习
### 练习 22

食客的 actor 代码只支持点馅饼。将其扩展到让顾客点馅饼的模式，由单独的代理管理馅饼片和冰激凌。安排好事情，这样它就可以处理一个或另一个用完的情况。
