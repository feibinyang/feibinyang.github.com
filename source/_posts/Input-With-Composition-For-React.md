---
title: Input-With-Composition-For-React
date: 2020-11-19 00:47:50
tags:
    - Input
    - CompositionEvent
    - React
categories:
    - 技术
---
##### 背景
​	目前业界主流的 React UI 组件库中，`Input` 组件都没有实现是否感知输入法功能，但是这个功能在某些情况下是很有必要的，比如搜索框根据输入关键词实时搜索时，React 提供的合成事件 `onChange` 默认是感知输入法的，在输入过程中会产生很多【无效的输入值】（下文统一用【值】指代【输入值】）被提交到后台查询（参考下图），这些查询是没有意义的。合理情况应该是等输入完成时（即非输入法状态时），再把值提交到后台查询。

{% asset_img demo.png %}

##### 方案
​	首先浏览器为 [IME](https://developer.mozilla.org/en-US/docs/Glossary/input_method_editor) 元素提供了 [CompositionEvent](https://developer.mozilla.org/en-US/docs/Web/API/CompositionEvent) 来感知输入法输入过程，其提供了 `compositionstart/compositionupdate/compositionend` 三个事件，我们可以借助它们来控制何时把有效值传出去。

​	其次在 React 提供的[合成事件](https://reactjs.org/docs/events.html#form-events)中，`onChange` 一般用在受控组件中，它是感知输入法的，为了兼容处理，我们采用另外一个合成事件 `onInput`（从语义上来说也更贴切），并增加一个 `props(composition)` 用来设置当前是否要感知输入法，这是对 `onInput` 功能的增强，没有破坏它的语义。

​	于是我们可以采用 `composition/onInput 2` 个 props，并借助 `compositionstart/compositionend` 来作开关，根据设置是否感知输入法（默认为 `false`，不感知）来实时把对应的值传出去。

##### 实现

​	首先我们需要调研 `CompositionEvent` 和 `InputEvent` 事件的调用时序，具体可以参考 [demo](https://jsbin.com/nitamigabo/1/edit?html,js,console,output)。这里直接贴上第三方结论.

```
// chrome: input(emit) -> ... -> compositionstart -> compositionupdate -> input(not emit) -> compositionend(emit)
// firefox: input(emit) -> ... -> compositionstart -> compositionupdate -> compositionend(not emit) -> input(emit)
// 可以归纳出：
//  紧跟 compositionupdate 后面的 input 不要往上 sync
//  紧跟 input 后面的 compositionend 往上 sync，即传值
```

所以我们可以在 `compositionstart` 时打开开关，`compositionend` 时关掉开关并调用 `onInput` 回调传值。原生 `input` 事件回调中，通过开关来判断是否非输入法状态（即输入完成），从而决定是否调用 `onInput` 回调传值。

​	其次是【开关】怎么设置问题。在 `Class Component` 中可以通过实例属性来设置，但在 `Function Component` 中行不通。通过 `state` 来控制也是不行的，因为 `setState` 是异步的，这会导致 `input` 事件回调执行时开关还没打开，从而把值传出去了。后来参考 [vue](https://github.com/vuejs/vue/blob/52719ccab8fccffbdf497b96d3731dc86f04c1ce/src/platforms/web/runtime/directives/model.js#L131) 里实现可以把开关放置在 `DOM` 元素上（`event.target`）。

​	最后设计 `Input` 组件结构，大致如下：


| 属性 | 类型 | 默认值 | 备注 |
| ---- | ---- | ---- | ---- |
| defaultValue | string |      | 非受控默认值 |
| value | String | | 受控值 |
| composition | boolean | false | 是否感知输入法 |
| onChange | Function | |  |
| onInput | Function | | 输入值，不感知输入法时只在非输入法状态调用 |

​	最终实现点击[这里](https://codesandbox.io/s/input-with-composition-9mk8r)。一个简单的可感知输入法 `Input` 组件实现完成。

*注意：因为 `vue` 内部对 `v-model` 的封装，导致 `vue` 组件中可能存在 `value` 值对不上问题。详情可参考[链接](https://github.com/ecomfe/veui/blob/d20/packages/veui/src/components/Input.vue)。*

##### 延伸阅读

​	之前没注意，这次看 MDN 时才发现原生 `input` 和 `change` 事件触发时机不一样（具体参考 [input event](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/input_event) 和 [change event](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/change_event)）。总结就是 `input` 是在每次输入值变化时就触发，`change` 是在 `lose focus` 或是 `commit value` 后才触发。



[参考资料]：

* https://developer.mozilla.org/en-US/docs/Web/API/CompositionEvent
* https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/input_event
* https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/change_event

* https://github.com/ecomfe/veui/blob/d20/packages/veui/src/components/Input.vue
* https://reactjs.org/docs/events.html#form-events