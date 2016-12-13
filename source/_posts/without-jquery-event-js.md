---
title: Without jQuery:event.js
date: 2016-07-23 15:30:03
tags: [javascript, without-jquery]
---
Without jQuery 系列之：event.js

使用原生的js实现简易的事件委托。
<!-- more -->

##  什么是事件委托

>什么是事件委托：通俗的讲，事件就是onclick，onmouseover，onmouseout，等就是事件，委托呢，就是让别人来做，这个事件本来是加在某些元素上的，然而你却加到别人身上来做，完成这个事件。

##  事件委托的原理

>利用冒泡的原理，把事件加到父级上，触发执行效果。

##  为什么要使用事件委托

比如，现在有一个列表，需要有这么一个交互：点击列表中每一项，都在控制台打印出该项的文字。

```html
<ul class="list">
    <li class="item"> Item 1 </li>
    <li class="item"> Item 2 </li>
    <li class="item"> Item 3 </li>
    <li class="item"> Item 4 </li>
    <li class="item"> Item 5 </li>
    <li class="item"> Item 6 </li>
</ul>
```
如过不使用事件委托，那么每一项的点击事件都要这么写：
```javascript
var list = document.getElementById('list');
for (var i = list.length - 1; i >= 0; i--) {
    (function(index){
        list[index].addEventListener('click', function(ev){
            ev.preventDefault();
            console.log(this.innerText);
        })
    })(i)
}
```

这么写有两个大问题：

1.  效率太低下，需要给每一个li都绑定一个事件，如果有100个li，那么会消耗大量的时间和内存。
2.  对动态的元素不友好：每个新添加的元素都需要独立绑定一次事件，又带来新的效率问题。

但是，使用事件委托的话，这两个问题就迎刃而解了。
```javascript
list.addEventListener('click', function(ev){
    ev.preventDefault();
    var target = ev.target;
    if (target.tagName.toLowercase() == 'li') {
        console.log(target.innerText);
    }
})
```

##  怎么使用事件委托？

1.  在原生的js中，事件委托可以写成上述那样，每次执行的时候都判断该元素是否是目标元素。
2.  使用jquery的on函数：`$(element).on(type, selector, data, func)`

##  为什么要自己写一个而不是用jquery？

有时候在写一些小网站，交互并不多，使用jquery的地方很少很少，而且，现在的现代浏览器对DOM的操作已经很方便了，jquery有时就显得过于臃肿，为了使用一两个简单的功能而引入jquery，是得不偿失的。

所以，那就自己手动写一个和jquery的on函数类似的吧。

jquery的on函数可以实现两个最基本的功能：

1.  给元素绑定相应的事件：$(element).on(type, func)
2.  给元素中的子元素绑定事件，委托至父元素，让父元素执行：`$(element).on(type, selector, func)`
首先是第一个需求，给元素绑定相应的事件：

```javascript
function bindEvent(el, type, func){
    if (el.addEventListener) {
        el.addEventListener(type, func, false);
    }else if (el.attachEvent) {
        el.attachEvent(type, func);
    }else{
        el['on'+type] = func;
    }
}
```
就这样可以给一个元素el绑定相应的事件了。

再来就是委托。在这里，采用了最古老的方法，先在父元素中，能否查找到相应的子元素，若找到，则用子元素和点击事件的目标元素做对比；若在父元素中能找到目标元素，则执行相应的操作。

```javascript
function delegateEvent(el, type, selector, func){
    var agent = function(ev){
        var target = ev.target || ev.srcElement;
        var targets = el.querySelectorAll(selector);
        var matchTarget = null;
        for (var i = 0; i < targets.length; i++) {
            if (target === targets[i] || targets[i].contains(target)) {
                matchTarget = target;
                break;
            }
        }
        matchTarget && func.call(matchTarget, ev);
    }
    bindEvent(el, type, agent);
}
```
ok，一个简易的事件委托的框架大体完成了，接下来就是要合并一下，合并成我们常用的。

```javascript
Element.prototype._on = function(type, selector, func){
    if (selector && func == undefined) {
        func = selector;
        selector = null;
        bindEvent(this, type, func);
        console.log('on');
    }else if (selector == undefined && func == undefined) {
        bindEvent(this, type, function(ev){ev.preventDetault()})
        console.log('null');
    } else{
        delegateEvent.apply(this, arguments)
        console.log('delegate');
    }
}
```

该_on方法，接受3个参数：`type`(事件类型), `selector`(指定触发事件的元素), `func`(触发的事件)。

使用方法如`document.body._on('click', 'a', function(){console.log(this)})`，给body绑定一个点击事件，当点击a标签后，打印出该标签在控制台上。

##  结语

最近已经越来越少写jq的，网站交互少是一个方面，使用了angular也是另一个方面。

完整的代码：
```javascript
(function(window,undefined){
    function bindEvent(el, type, func){
        if (el.addEventListener) {
            el.addEventListener(type, func, false);
        }else if (el.attachEvent) {
            el.attachEvent(type, func);
        }else{
            el['on'+type] = func;
        }
    }
    function delegateEvent(type, selector, func){
        var agent = function(ev){
            var target = ev.target || ev.srcElement;
            var targets = this.querySelectorAll(selector);
            // console.log(targets);
            var matchTarget = null;
            for (var i = 0; i < targets.length; i++) {
                if (target === targets[i] || targets[i].contains(target)) {
                    matchTarget = target;
                    break;
                }
            }
            matchTarget && func.call(matchTarget, ev);
        }
        bindEvent(this, type, agent);
    }
    Element.prototype._on = function(type, selector, func){
        if (selector && func == undefined) {
            func = selector;
            selector = null;
            bindEvent(this, type, func);
            console.log('on');
        }else if (selector == undefined && func == undefined) {
            bindEvent(this, type, function(ev){ev.preventDetault()})
            console.log('null');
        } else{
            delegateEvent.apply(this, arguments)
            console.log('delegate');
        }
    }
})(window);
```
第一次写这种封装的小东西，如果有什么地方不妥的，尽情吐槽。
