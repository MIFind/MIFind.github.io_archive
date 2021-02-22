---
title: mobx使用指北
date: 2019-08-15 10:22:31
tags: MobX
---
{: id="20210222104133-9w9tjkh"}

**MobX 是一个经过战火洗礼的库，它通过透明的函数响应式编程(transparently applying functional reactive programming - TFRP)使得状态管理变得简单和可扩展。**
{: id="20210222104133-obmlnd9"}

MobX 背后的哲学很简单：**任何源自应用状态的东西都应该自动地获得。**
{: id="20210222104133-fnrw2df"}

其中包括 UI、数据序列化、服务器通讯，等等。
{: id="20210222104133-r4ynmwc"}

React 和 MobX 是一对强力组合。React 通过提供机制把应用状态转换为可渲染组件树并对其进行渲染。而 MobX 提供机制来存储和更新应用状态供 React 使用。
{: id="20210222104133-8wf4eoy"}

对于应用开发中的常见问题，React 和 MobX 都提供了最优和独特的解决方案。React 提供了优化 UI 渲染的机制， 这种机制就是通过使用虚拟 DOM 来减少昂贵的 DOM 变化的数量。MobX 提供了优化应用状态与 React 组件同步的机制，这种机制就是使用响应式虚拟依赖状态图表，它只有在真正需要的时候才更新并且永远保持是最新的。
{: id="20210222104133-6ovxox5"}

### 核心概念
{: id="20210222104133-061ea4l"}

#### 1. Observable state(可观察的状态)<br>
{: id="20210222104133-hltprkg"}

MobX 为现有的数据结构(如对象，数组和类实例)添加了可观察的功能。 通过使用 @observable 装饰器(ES.Next)来给你的类属性添加注解就可以简单地完成这一切。
{: id="20210222104133-vyaix0y"}

#### 2. Computed value(计算值)
{: id="20210222104133-cl006km"}

使用 MobX， 你可以定义在相关数据发生变化时自动更新的值。
{: id="20210222104133-e2dxw44"}

```
class TodoList {
    @observable todos = [];
    @computed get unfinishedTodoCount() {
        return this.todos.filter(todo => !todo.finished).length;
    }
}
```
{: id="20210222104133-ooxdecf"}

当添加了一个新的 todo 或者某个 todo 的 finished 属性发生变化时，MobX 会确保 unfinishedTodoCount 自动更新。 像这样的计算可以类似于 MS Excel 这样电子表格程序中的公式。每当只有在需要它们的时候，它们才会自动更新。
{: id="20210222104133-crlrzyf"}

#### 3. Reactions(反应)
{: id="20210222104133-3t140e3"}

Reactions 和计算值很像，但它不是产生一个新的值，而是会产生一些副作用，比如打印到控制台、网络请求、递增地更新 React 组件树以修补 DOM、等等。
{: id="20210222104133-4ek2rbe"}

##### React
{: id="20210222104133-wu06r54"}

如果你用 React 的话，可以把你的(无状态函数)组件变成响应式组件，方法是在组件上添加 observer 函数/ 装饰器. observer 由 mobx-react 包提供的。
{: id="20210222104133-1nar0cu"}

```
import React, {Component} from 'react';
import ReactDOM from 'react-dom';
import {observer} from 'mobx-react';

@observer
class TodoListView extends Component {
    render() {
        return <div>
            <ul>
                {this.props.todoList.todos.map(todo =>
                    <TodoView todo={todo} key={todo.id} />
                )}
            </ul>
            Tasks left: {this.props.todoList.unfinishedTodoCount}
        </div>
    }
}

const TodoView = observer(({todo}) =>
    <li>
        <input
            type="checkbox"
            checked={todo.finished}
            onClick={() => todo.finished = !todo.finished}
        />{todo.title}
    </li>
)

const store = new TodoList();
ReactDOM.render(<TodoListView todoList={store} />, document.getElementById('mount'));

```
{: id="20210222104133-qm4tkdw"}

##### 自定义 reactions
{: id="20210222104133-nja470c"}

使用 autorun、reaction 和 when 函数即可简单的创建自定义 reactions，以满足你的具体场景。
{: id="20210222104133-mis2oli"}

例如，每当 unfinishedTodoCount 的数量发生变化时，下面的 autorun 会打印日志消息:
{: id="20210222104133-37eftwj"}

```
autorun(() => {
    console.log("Tasks left: " + todos.unfinishedTodoCount)
})
```
{: id="20210222104133-qxjhyr1"}

#### Actions(动作)
{: id="20210222104133-ddkq7nk"}

不同于 flux 系的一些框架，MobX 对于如何处理用户事件是完全开明的。
{: id="20210222104133-vmnqtjp"}

可以用类似 Flux 的方式完成
或者使用 RxJS 来处理事件
或者用最直观、最简单的方式来处理事件，正如上面演示所用的 onClick
最后全部归纳为: 状态应该以某种方式来更新。
{: id="20210222104133-kcomvnc"}

当状态更新后，MobX 会以一种高效且无障碍的方式处理好剩下的事情。像下面如此简单的语句，已经足够用来自动更新用户界面了。
{: id="20210222104133-lprt8b8"}

从技术上层面来讲，并不需要触发事件、调用分派程序或者类似的工作。归根究底 React 组件只是状态的华丽展示，而状态的衍生由 MobX 来管理。
{: id="20210222104133-yt25tfi"}

```
store.todos.push(
    new Todo("Get Coffee"),
    new Todo("Write simpler code")
);
store.todos[0].finished = true;
```
{: id="20210222104133-83fk1ky"}


{: id="20210222104133-kz7ivhz" type="doc"}
