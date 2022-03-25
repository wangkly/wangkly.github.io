## My Reading Notes

### [React 源码阅读-01](React_01.md)
> React 从16 的版本开始，引入了一个实验性的**concurrentMode** ,设计在concurrent的模式下，react的渲染过程可以中断，可以显著提升react的渲染性能。但是concurrentMode还是处于实验的阶段，官网也只有一些简单的介绍,（其实16.8后引入了hook,背后也有concurrentMode的因素...

### [React 源码阅读-02](React_02.md)

>从上面的分析我们知道，reactDom.render 内部经历2个步骤:
>- createContainer ，创建一个root ( FiberRoot ),一个 rootFiber ，并设置root.current = rootFiber ,rootFiber.stateNode = root
2者相互保存对方的引用
>- updateContainer ,创建完fiberRoot 和rootFiber后，执行updateContainer 。 new Update(),设置payload 为App. 放入rootFiber的updateQueue , 然后调用**scheduleUpdateOnFiber**，触发整个应用的更新
....

### [React 源码阅读-03](React_03.md)
> React  lane 模型简介。。。

### [React 源码阅读-04](React_04.md)
> performSyncWorkOnRoot 相关逻辑。。。

### [React 源码阅读-05](React_05.md)
> beginWork 相关逻辑

### [Vue3 源码阅读-01](vue3.x_01.md)
> 在学习源码的过程中，有很多复杂的逻辑，当时可能理解了，但是随着时间的流逝，经常会很多的重要的知识点会被遗忘，主要是因为源码的思想一方面很复杂，有很多的算法思想在里面，理解起来会有困难，另一方面是因为，我们阅读的源码在平时开发的过程中使用不到。不用，所以经常会忘记。所以我想写一些读源码的过程中、记录一些笔记，可以方便日后在复习这些内容的时候。做些提示...


### [Vue3 源码阅读-02](Vue3.x_02.md)
> vue3项目入口...

### [Vue3 源码阅读-03](Vue3.x_03.md)
> reactive ref相关


### [Vue3 源码阅读-04](vue3.x_04.md)
> watch computed 相关逻辑


### 关于
> 在学习的过程中，总是容易看了后面忘了前面，所以想着要做一些记录，方便后面再看的时候可以拿来参考，有助于记忆
认识有限，内容中可能会有错误的地方，欢迎指正
