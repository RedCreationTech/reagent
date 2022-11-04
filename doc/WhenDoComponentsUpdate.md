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

If I suddenly got all scientific, and did this `(reset! name "Ursidae")`, Reagent would detect the change in `name`, and it would re-run any Component renderer which is dependent upon it.  That means `ask-for-forgiveness` is re-run, producing the new hiccup `[:div "Please " "Ursidae" " with me"]`.

如果说我修改了atom的值. 比如通过repl执行了以下代码`(reset! name "Ursidae")`, Reagent会监测到这个变化, 会重新渲染这个组件, 生成新的hiccpu `[:div "Please " "Ursidae" " with me"]`.


Just so we're clear: a "data input" changes (the value in a ratom) and, then, the renderer is rerun to produce new hiccup.  The Component is reactive to the ratoms it derefs.

再次澄清一下: 输入变化了(atom中的数据), 渲染器(renderer)会生成新的Hiccup. 也就是说组件会响应Atom的值的变化.

## 组合使用的例子

So that was the basics.

我们已经看了各自的基础场景.

Let's now look at how these things can combine. We're going to consider a case involving two child components, and a parent.

结下来我们看一个既有`props`也有`atom`的情况.

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

With this setup, answer this question: what rerendering happens each time the `more-button` gets clicked and `counter` gets incremented?
基于以上代码, 思考一个问题: 哪个组件会因为`more-button`的点击和`counter`计数器的增加?

Don't read on. Test yourself. Spend 30 seconds working it out.

忍住看答案的冲动, 自己思考30分组, 你的结论是什么.

Answer:
  1. Reagent will notice that `counter` has changed and that is an `input ratom` to `parent-renderer`, and it will rerun that renderer.
  2. Reagent will interpret the hiccup returned by `parent-renderer`, and it will determine that a new (integer) prop has been supplied in the `[greet-number @counter]` Component, and it will then rerender that component too.

答案:
 1. Reagent发现`parent-renderer`订阅了`counter`, 当这个atom发生变化的时候, 渲染器会重新执行, 生成新的hiccpu
 2. Reagent会解析这个Hiccup, 发现新的`[greet-number @counter]` 的prop发生了变化. 所以渲染器也会使用新的prop`(@counter)`执行`greet-number`生成Hiccup.

Wait. Is that it?  Why doesn't the `[more-button counter]` component rerender too?  After all, its `prop` `counter` has changed???

巧逃麻袋, 就这样吗? 为什么`[more-button counter]`这个组件不重新渲染?? 它的`prop`也就是`counter`不是变化了吗???

No, I promise it won't rerender. But why not?  The answer is a bit subtle.

并没有! 原因有些微妙.

You see, `counter` itself hasn't changed. It is still the same ratom it was before. The value **in** `counter` has been incremented, but `counter` itself is still the same ratom.  So from Reagent's point of view `[more-button counter]` involves the same `prop` as "last time" and it concludes that there's no need for a rerender of that component.

`counter`本身并没有变化. atom还是哪个atom, 它**里面**的值变了. 以Reagent的角度, `[more-button counter]`的`prop`和上次是一样的, 所以不需要对这个组件进行重新渲染.

Had `more-button` dereferenced the `counter` ratom THEN the change in `counter` should have triggered a rerender of `more-button`.  But if you look at `more-button` you'll see no `@counter`. There is no dereference.

如果`more-button`的形式是`[more-button @counter]`, 它的`prop`就发生了改变, 会触发改变.

If you truly understand this example, then you've gone a long way to officially getting it.

如果你理解了上面的例子, 你也就懂得了渲染器的工作原理.

## Different

Although they are both ways to trigger a reactive re-render, the two kinds of `inputs` have different properties:
  1. the definition of "changed" applied
  2. treatment of lifecycle functions

## Changed?

Till now, I've said a renderer will be re-run when an input value "changed".  But I've been carefully avoiding any definition of "changed".

You see, there's at least two definitions: `=` and `identical?`

```
(def x1  {:a 42  :b 45})    ;; at time 1, x has this value
(def x2  {:a 42  :b 45})    ;; at time 2, x has this value

(= x1 x2)                   ;; is x the same, or has it changed?
;; =>  true                 ;; answer: no change

(identical? x1 x2)          ;; is x the same, or has it changed?
;; => false                 ;; answer: different
```

So we can see different answers to the question "has x changed?" for the same values, depending on the function we use.

For `props`,  `=` is used to determine if a new value have changed with regard to an old value.

For ratoms, `identical?` is used (on the value inside the ratom) to determine if a new value has changed with regard to an old value.

So, it is only when values are deemed to have "changed", that a re-run is triggered, but the inputs use different definitions of "changed".  This can be confusing.

The `identical?` version is very fast. It is just a single reference check.

The `=` version is more accurate, more intuitive, but potentially more expensive. Although, as I'm writing this I notice that `=` uses `identical?` [when it can](https://github.com/clojure/clojurescript/blob/1b7390450243693d0b24e8d3ad085c6da4eef204/src/main/cljs/cljs/core.cljs#L1108-L1124).

**Update:**

> As of Reagent 0.6.0, ratoms use `=` (instead of `identical?`) is to determine if a new value is different to an old value. So, `ratoms` and `props` now have the same `changed?` semantics.

### Efficient Re-renders

It's only via rerenders that a UI will change.  So re-rendering is pretty essential.

On the other hand, unnecessary re-rendering should be avoided.  In the worst case, it could lead to performance problems.  By unnecessary rendering, I mean rerenders which result in unchanged HTML. That's a whole lot of work for no reason.

So this notion of "changed" is pretty important.  It controls if we are doing unnecessary, performance-sapping re-rendering work.

### Lifecycle Functions

When `props` change, the entire underlying React machinery is engaged. Reagent Components can have lifecycle methods like `component-did-update` and these functions will get called, just as they would if you were dealing with a React Component.

But ... when the re-render occurs because an input ratom changed, **Lifecycle functions are not run**.  So, for example, `component-did-update` will not be called on the Component.

Careful of this one. It trips people up.


## Appendix 1

In the previous Tutorial, we looked at the difference between `()` and `[]`.

Towards the end, I claimed that using `[]` was more efficient at "re-render time".  Hopefully, after the tutorial above, our knowledge is a bit deeper and we can now better appreciate the truth in this claim.

Remember this code from the previous tutorial:
```clj
(defn greet-family-square
  [member1 member2 member3]
  [:div
    [greet member1]     ;; using [] not ()
    [greet member2]
    [greet member3]])
```

Now, imagine it used like this:
```clj
 ;; the 3rd member of the family to greet
(def extra (reagent.core/atom "Aunt Edith"))

(defn top-level-component
    []
    [greet-family-square  "Mum" "Dad" @extra])

(reagent.core/render [top-level-component] (.-body js/document)))
```

The first time the page is rendered, the DOM created will greet three cherished people.  All good.  At this point, the `round` and `square` versions of `greet-family` would be equally good at getting the initial DOM into our browser.

But then, out of nowhere, comes information that our rich and eccentric Uncle John is rewriting his Last Will And Testament, and we need a fast, realtime change in our page. Luckily we have a repl handy, and we type `(reset! extra "Uncle John")`.  We've changed that `extra` r/atom which holds the 3rd cherished family member - sorry "Aunt Edith", you're out.

What happens next?

1. Reagent will recognize that a `top-level-component` component relies on `extra` which has changed.
2. So it will rerender that component. In the hiccup produced (by the rerender), it will see that `greet-family-square` has a new 3rd prop. And, by that, I mean that the value for the 3rd prop ("Uncle John") will not compare `=` to the value last rendered ("Aunt Edith").
3. So Reagent will trigger a rerender of `greet-family-square` with the new props ("Mum" "Dad" and "Uncle John")
4. In the hiccup produced by this rerender, Reagent will notice that the first two `greet` components have the same prop as that last rendered ("Mum" and "Dad") but that the 3rd `greet` component has a new prop value ("Uncle John").
5. So it will NOT rerender the first two `greet`, but the renderer for the 3rd will be rereun.

As you can see, only the right parts of the tree are re-rendered.  Nothing unnecessary is done.

In the alternative `greet-family-round` version we looked at, the one which used `()` instead of `[]`, that efficiency is not possible.

A re-render of `greet-family-round` always triggers three calls to `greet`, no matter what, accumulating a large amount of hiccup for `greet-family-round` to return.  It would then be left to React to diff all this new DOM with existing DOM and for it to work out that, in fact, parts of the tree (the first two greet parts) remain the same, and should be ignored. Which is a whole lot of unnecessary work!

When we use `[]`, we get independent React Components which will only be re-rendered if their `props` change (or ratoms change). More efficient, more minimal re-renderings.
