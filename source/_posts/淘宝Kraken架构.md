---
title: Kraken架构设计及Kraken_internal
date: 2021-02-22 11:24:32
tags:
---
# Kraken架构设计及Kraken_internal

## 目标

对于一个跨端框架，我们希望的是：

* 性能和体验接近原生
* 基于前端体系
* 可自定义扩展
* 可动态发布
* 可跨多平台
* 多端UI一致性
* 开发工具完善

## Kraken机制

![image.png](https://cdn.nlark.com/yuque/0/2020/png/297774/1594352149873-cdc720d8-7fe4-419b-aba6-dcf7c359014d.png?x-oss-process=image%2Fresize%2Cw_746)

1. Kraken的最上层是一个基于W3C标准而构建的DOM API，在下层是所依赖的JS引擎，通过C++构建Bridge与Dart通信。
2. C++ Bridge把JS所调用的一些信息，转发到Dart层。
3. Dart层通过接收这些信息，下沉到Flutter Render Engine，DOM Render Tree与Render Object进行对齐，实现高效的渲染性能。

## Kraken_internal

Kraken_internal是用于构建DOM，DOM API，Timer，并代理bridge。

![image.png](https://cdn.nlark.com/yuque/0/2020/png/297774/1594352267593-47c87366-07a8-4b09-b142-e61a530228e0.png?x-oss-process=image%2Fresize%2Cw_746)

这是Kraken DOM模型的抽象，提供了和浏览器一样的DOM API，并暴露在JS环境中，JS通过使用DOM API也就是来创建一个DOM树。

![image.png](https://cdn.nlark.com/yuque/0/2020/png/297774/1594352338873-f2ff09a1-816f-499f-84a0-49a4e2f98a75.png?x-oss-process=image%2Fresize%2Cw_746)

而构建出这个DOM树并将DOM树信息与API调用给到Dart层进行处理的库就是Kraken_internal。

### Kraken_internal API

* ES6-promise
* DOM Api

  * EventTarget
  * StyleDeclaration
  * Node

    * TextNode
    * Element

      * MediaElement
      * CanvasElement
      * ImageElement
      * IframeELement
      * AnimationPlayerElement
      * ObjectElement
      * AnchorELement
    * Comment
    * Document
  * CookieStorage
  * Event

    * PromiseEvent
    * ErrorEvent
    * CustormEvent
  * Network

    * matchMedia
    * websocket
    * fetch
    * URL
    * location
    * navigator
    * requestAnimationFrame
    * XMLHttpRequest
    * Blob
    * asyncStorage
    * Performance
    * MQTT
    * methodChannel
  * history
  * window