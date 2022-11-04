当前教程说明何时以及为何Reagent组件会重新render.

## Components Are Reactive(组件是响应式的)

Reagent组件是响应式的
  - 每个组件都有自己的render函数
  - render函数的作用是把 `输入` 转化为`Hiccpu`
  - 当 `输入`变化了, render会重新执行, 得到新的 Hiccup
  - Hiccup经由Reagent处理后, 生成为HTML

重新渲染(Render函数重新执行)的过程, 也就是组件响应`输入`, 重新生成新的输出的过程.

本页的作用是帮助理解如何以及何时这个机制发生作用.

### Reactive To What?(对什么响应)

首先我们来看看输入, 到底是什么东西, 什么时候这些东西发生变化, 触发组件的重新渲染.

有两种类型的 `输入`:
  - `props`
  - `ratoms`

我们很快就会明白两者的不同, 而且触发的方式不一样.

## 1. Props

第一种类型的输入是`props`

注意以下组件
```clj
(defn greet
  [name]          ;; name is a string
  [:div "Hello " name])
```

`name` 就是一个 `prop` (property的简写).  每次`name`的值发生变化, `greet`就会重新渲染.

等下, 你说什么, `name`的值怎么会变化呢? 它就是个函数的参数啊! 参数这东西只会给函数调用时候传参呀!

你应该还记得`greet`会被升格成为一个组件的render函数. 作为一个render函数, 至少会被调用一次, 更多时候是会被反复调用. 所以时间线上, `greet`被调用的时候你会有机会遇到不同的输入参数, 从这个意义上, `name`是一个可以变化的输入.

为了深入理解, 我们构造一个父组件, 它使用`greet`:
```clj
(defn greet-family
  []
  [:div
    [greet "Dad"]
    [greet (str "Bro-" (rand-int 10))]])
```

当Reagent解析`greet-family`返回的hiccup的时候, Reagent会创建三个组件:
  - 一个 `:div`组件, 有两个`greet`子组件
  - 第一个`greet`组件, `name`的参数始终是 "Dad"
  - 第二个`greet`组件, `name`参数是个随机数, 每次只有1/10的机会和上次一样.

生成的Hiccp会经过Reagent的分析, 判断这三个组件是否需要重新渲染. 判断规则是: 对于每个组件, 本次的 `props`是否和上次一致, 也就是说他们是否有`变化`.

If the `props` are different, then that Component's render will be called to create new hiccup.  But if the `props` to that Component are the same as last time, then no need to rerender it.

如果`props`不同, 我们定义的render函数就会重新被调用, 生成新的hiccpu. 如果相同, 函数就不会被调用, 也就不发生重新渲染.

显而易见, `[greet "Dad"]` 组件, `greet-family` 每次要渲染的内容都是一样, 是不需要重新渲染的, 所以只会在程序启动的时候执行一次, 无论它的父组件`greet-family`被渲染多少次.

而对于`[greet (str "Bro-" (rand-int 10))]`来说, 得到一个不同的`name`是大概率(90%)事件, 所以`greet-family`重新渲染时候, 它也常常会渲染.


Which means we can now answer the question posed above - how can the value of `name` change over time for a given `greet` component?  Answer: when the parent Component re-renders, and supplies a new value as the `prop`.

至此, 我们回答了上面`name`这个`prop`如何会变化的问题.

`props` flow from the parent. **A Component can't get new `props` unless its parent rerenders.**

`props`来自于父组件, **当前仅当父组件渲染**, 才有可能拿到新的`propos`.

## 2. Ratoms


接下来我们看看第二种情况, 也就是`输入数据`是Reagent Atom的情况.

接下来这个例子有点牵强, 你忍一下:
```clj
(def name  (reagent.ratom/atom "Bear"))

(defn ask-for-forgiveness
  []           ;; <--- no props
  [:div "Please " @name " with me"])   ;; notice that @
```

`ask-for-forgiveness`会产生一个Hiccup`[:div "Please " "Bear" " with me"]`

最初, `name`这个atom的值是"Bear".

输入数据输入的方式是经由`name`这个atom. Reagent会监测到这个render函数有一个atom输入, 会自动订阅这个atom的变化.


如果说我修改了atom的值. 比如通过repl执行了以下代码`(reset! name "Ursidae")`, Reagent会监测到这个变化, 会重新渲染这个组件, 生成新的hiccpu `[:div "Please " "Ursidae" " with me"]`.

再次澄清一下: 输入变化了(atom中的数据), 渲染器(renderer)会生成新的Hiccup. 也就是说组件会响应Atom的值的变化.

## 组合使用的例子

我们已经看了各自的基础场景.
接下来我们看一个既有`props`也有`atom`的情况.

Child Component 1:
```clj
(defn greet-number
  "I say hello to an integer"
  [num]                             ;; an integer
  [:div (str "Hello #" num)])       ;; [:div "Hello #1"]
```

Child component 2:
```clj
(defn more-button
  "I'm a button labelled 'More' which increments counter when clicked"
  [counter]                                ;; a ratom
  [:div  {:class "button-class"
          :on-click  #(swap! counter inc)} ;; increment the int value in counter
   "More"])
```

一个Form-2 parent Component使用这俩子组件.
```clj
(defn parent
  []
  (let [counter  (reagent.ratom/atom 1)]    ;; the render closes over this state
    (fn  parent-renderer
      []
      [:div
        [greet-number @counter]      ;; notice the @. The prop is an int
        [more-button counter]])))    ;; no @ on counter
```

基于以上代码, 思考一个问题: 哪个组件会因为`more-button`的点击和`counter`计数器的增加?

忍住看答案的冲动, 自己思考30分组, 你的结论是什么.

答案:
 1. Reagent发现`parent-renderer`订阅了`counter`, 当这个atom发生变化的时候, 渲染器会重新执行, 生成新的hiccpu
 2. Reagent会解析这个Hiccup, 发现新的`[greet-number @counter]` 的prop发生了变化. 所以渲染器也会使用新的prop`(@counter)`执行`greet-number`生成Hiccup.

Wait. Is that it?  Why doesn't the `[more-button counter]` component rerender too?  After all, its `prop` `counter` has changed???

巧逃麻袋, 就这样吗? 为什么`[more-button counter]`这个组件不重新渲染?? 它的`prop`也就是`counter`不是变化了吗???

并没有! 原因有些微妙.

`counter`本身并没有变化. atom还是哪个atom, 它**里面**的值变了. 以Reagent的角度, `[more-button counter]`的`prop`和上次是一样的, 所以不需要对这个组件进行重新渲染.

Had `more-button` dereferenced the `counter` ratom THEN the change in `counter` should have triggered a rerender of `more-button`.  But if you look at `more-button` you'll see no `@counter`. There is no dereference.

如果`more-button`的形式是`[more-button @counter]`, 它的`prop`就发生了改变, 会触发改变.

如果你理解了上面的例子, 你也就懂得了渲染器的工作原理.

## 两种触发机制的差别

尽管使用`props`和`atom`都能触发重新渲染, 它嗯还是有差异的.
  1. 变化的涵义差别(这个只针对reagent 6.0以前的版本), 不展开了.
  2. react生命周期函数的差异.

### 渲染的效率


UI只有通渲染器(renderer)才能实现改变, 所以这个机制非常重要.


另一方面, 不必要的重新刷新是需要避免的. 在性能敏感的情况下, 额外的刷新是会引起麻烦.

### 生命周期函数

`proos`的改变会触发React生命周期函数 `component-did-update`

But ... when the re-render occurs because an input ratom changed, **Lifecycle functions are not run**.  So, for example, `component-did-update` will not be called on the Component.

但是`atom`的改变不会. `component-did-update` 不会被触发!!!!


## 再回到[]和()的问题

下面我在解释下为什么[]比()更高效.

再看一下这个例子:
```clojure

(defn greet-family-square
  [member1 member2 member3]
  [:div
   [greet member1]     ;; using [] not ()
   [greet member2]
   [greet member3]])

(defn greet-family-round
  [member1 member2 member3]
  [:div
   (greet member1)  
   (greet member2)
   (greet member3))]
```

在这个使用场景下:
```clj
 ;; the 3rd member of the family to greet
(def extra (reagent.core/atom "Aunt Edith"))

(defn top-level-component
    []
    [greet-family-square  "Mum" "Dad" @extra])

(reagent.core/render [top-level-component] (.-body js/document)))
```

页面载入的时候, 两者都会执行, 结果也没有什么区别.

但是如果执行`(reset! extra "Uncle John")`, []和()会有什么区别呢?

1. Reagent会监测到`top-level-component`函数有一个atom`extra`输入, 而且这个值变化了
2. 所以`top-level-component`会经由渲染器重新生成Hicuup.
3. 接下来Reagent会尝试对组件`greet-family-square`进行重新渲染, 使用prop:`("Mum" "Dad" and "Uncle John")`.
4. 在对生成的Hicuup中, Reagent会发现前两个和上次渲染的值一样, 只有第三个不同
5. 头两个不需要重新渲染, 只会渲染第三个, 也只有第三个的函数得到了执行


如上所见, 没有多余的渲染!

在使用()的版本中, 每次父组件渲染, 这三个函数都会执行一遍.

本质上, `[]`为每个reagent组件创造了对应的react组件, 每个组件都有自己的独立的刷新机制.
