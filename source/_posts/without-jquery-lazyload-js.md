---
title: Without jQuery:lazyload.js
date: 2016-08-05 16:32:52
tags: [javascript, no-jquery, lazyloadw]
---
Without jQuery 系列之：lazyload.js

使用原生的js实现简易的图片延时加载。
<!-- more -->

##  什么是延时加载？

>图片延迟加载也称 “懒加载”，通常应用于图片比较多的网页

##  为什么要使用延时加载？

假如一个网页中，含有大量的图片，当用户访问网页时，那么浏览器会发送n个图片的请求，加载速度会变得缓慢，性能也会下降。如果使用了延时加载，当用户访问页面的时候，只加载首屏中的图片；后续的图片只有在用户滚动时，即将要呈现给用户浏览时再按需加载，这样可以提高页面的加载速度，也提升了用户体验。而且，统一时间内更少的请求也减轻了服务器中的负担。

##  延时加载的原理

基本原理就是最开始时，所有图片都先放一张占位图片（如灰色背景图），真实的图片地址则放在 `data-src` 中，这么一来，网页在打开时只会加载一张图片。

然后，再给 `window` 或 `body` 或者是图片主体内容绑定一个滚动监听事件，当图片出现在可视区域内，即`滚动距离 + 窗体可视距离 > 图片到容器顶部的距离`时，将讲真实图片地址赋值给图片的 src，否则不加载。

##  使用原生js实现图片的延时加载

延时加载需要传入的参数：

```javascript
var selector = options.selector || 'img',
    imgSrc = options.src || 'data-src',
    defaultSrc = options.defaultSrc || '',
    wrapper = options.wrap || body;
```

其中：

* `wrapper` ：延时加载的容器。在该容器下，所有符合图片选择器条件的图片均会延时加载。
* `selector` ：图片选择器。表示需要延迟加载的图片的选择器，如 `img.lazyload-image` ，默认为所有的 img 标签。
* `imgSrc` ：图片真实地址存放属性。表示图片的真实路径存放在标签的哪个属性中，默认为 `data-src`。
* `defaultSrc` ：初始加载的图片地址，默认为空，当为空时，不处理延时加载的图片的路径，若图片本身没有路径，则显示为空。
获取容器中所有的图片。

```javascript
function getAllImages(selector){
    return Array.prototype.concat.apply([], wrapper.querySelectorAll(selector));
}
```

该函数在容器中查找出所有需要延时加载的图片，并将 NodeList 类型的对象转换为允许使用 map 函数的数组。

如果设置了初始图片地址，则加载。

```javascript
function setDefault(){
    images.map(function(img){
        img.src = defaultSrc;
    })
}
```
给 window 绑定滚动事件
```javascript
function loadImage(){
    var nowHeight = body.scrollTop || doc.documentElement.scrollTop;
    console.log(nowHeight);
    if (images.length > 0){
        images.map(function(img, index) {
            if (nowHeight + winHeight > img.offsetTop) {
                img.src = img.getAttribute(imgSrc);
                images.splice(index, 1);
            }
        })
    }else{
        window.onscroll = null;
    }
}
window.onscroll = loadImage();
```

每次滚动网页时，都会遍历所有的图片，将图片的位置与当前滚动位置作对比，当符合加载条件时，将图片的真实地址赋值给图片，并将图片从集合中移除；当所有需要延时加载的图片都加载完毕后，将滚动事件取消绑定。

##  测试是否可行

###    测试结果：

从chrome的网络请求图中可见，5张图片并不是在网页打开的时候就请求了，而是当滑动到某个区域时才触发加载，基本实现了图片的延时加载。
测试结果

###    性能调整

上述只是简单的实现了一个延时加载的 demo，还有很多地方需要调整和完善。

####    调整 1：onscroll 函数可能会被覆盖

**问题：**

因为有时候页面需要滚动无限加载时，插件会重写 window 的 onscroll 函数，从而导致图片的延时加载滚动监听失效。

**解决办法：**

需要更改为将监听事件注册到 window 上，移除时只需要移除相应的事件即可。

**调整后的代码**

```javascript
function bindListener(element, type, callback){
    if (element.addEventListener) {
        element.addEventListener(type, callback);
    }else if (element.attachEvent) {
        //兼容至 IE8
        element.attachEvent('on'+type, callback)
    }else{
        element['on'+type] = callback;
    }
}
function removeListener(element, type, callback){
    if (element.removeEventListener) {
        element.removeEventListener(type, callback);
    }else if (element.detachEvent) {
        element.detachEvent('on'+type, callback)
    }else{
        element['on'+type] = callback;
    }
}
function loadImage(){
    var nowHeight = body.scrollTop || doc.documentElement.scrollTop;
    console.log(nowHeight);
    if (images.length > 0){
        images.map(function(img, index) {
            if (nowHeight + winHeight > img.offsetTop) {
                img.src = img.getAttribute(imgSrc);
                images.splice(index, 1);
            }
        })
    }else{
        //解绑滚动事件
        removeListener(window, 'scroll', loadImage)
    }
}
//绑定滚动事件
bindListener(window, 'scroll', loadImage)
```

####    调整2：滚动时的回调函数执行次数太多

**问题**

在本次测试中，从动图最后可以看到，当滚动网页时，loadImage 函数执行了非常多次，滚轮每向下滚动 100px 基本上就要执行 10 次左右的 loadImage，若处理函数稍微复杂，响应速度跟不上触发频率，则会造成浏览器的卡顿甚至假死，影响用户体验。

**解决办法**

使用 `throttle` 控制触发频率，让浏览器有更多的时间间隔去执行相应操作，减少页面抖动。

**调整后的代码：**
```javascript
//参考 `underscore` 的源码
var throttle = function(func, wait, options) {
    var context, args, result;
    var timeout = null;
    // 上次执行时间点
    var previous = 0;
    if (!options) options = {};
    // 延迟执行函数
    var later = function() {
        // 若设定了开始边界不执行选项，上次执行时间始终为0
        previous = options.leading === false ? 0 : _now();
        timeout = null;
        result = func.apply(context, args);
        if (!timeout) context = args = null;
    };
    return function() {
        var now = _now();
        // 首次执行时，如果设定了开始边界不执行选项，将上次执行时间设定为当前时间。
        if (!previous && options.leading === false) previous = now;
        // 延迟执行时间间隔
        var remaining = wait - (now - previous);
        context = this;
        args = arguments;
        // 延迟时间间隔remaining小于等于0，表示上次执行至此所间隔时间已经超过一个时间窗口
        // remaining大于时间窗口wait，表示客户端系统时间被调整过
        if (remaining <= 0 || remaining > wait) {
            clearTimeout(timeout);
            timeout = null;
            previous = now;
            result = func.apply(context, args);
            if (!timeout) context = args = null;
            //如果延迟执行不存在，且没有设定结尾边界不执行选项
        } else if (!timeout && options.trailing !== false) {
            timeout = setTimeout(later, remaining);
        }
        return result;
    };
};
//在调用高频率触发函数处使用 throttle 控制频率在 次/wait
var load = throttle(loadImage, 250);
//绑定滚动事件
bindListener(window, 'scroll', load);
//解绑滚动事件
removeListener(window, 'scroll', load)
```

### 调整后的测试

从动图可见，在滚动的时候，调用判断的回调的次数少了很多。而且也不影响图片的延时加载。

调整后的测试结果

封装为插件形式
```javascript
;(function(window, undefined){
    function _now(){
        return new Date().getTime();
    }
    //辅助函数
    var throttle = function(func, wait, options) {
        var context, args, result;
        var timeout = null;
        // 上次执行时间点
        var previous = 0;
        if (!options) options = {};
        // 延迟执行函数
        var later = function() {
            // 若设定了开始边界不执行选项，上次执行时间始终为0
            previous = options.leading === false ? 0 : _now();
            timeout = null;
            result = func.apply(context, args);
            if (!timeout) context = args = null;
        };
        return function() {
            var now = _now();
            // 首次执行时，如果设定了开始边界不执行选项，将上次执行时间设定为当前时间。
            if (!previous && options.leading === false) previous = now;
            // 延迟执行时间间隔
            var remaining = wait - (now - previous);
            context = this;
            args = arguments;
            // 延迟时间间隔remaining小于等于0，表示上次执行至此所间隔时间已经超过一个时间窗口
            // remaining大于时间窗口wait，表示客户端系统时间被调整过
            if (remaining <= 0 || remaining > wait) {
                clearTimeout(timeout);
                timeout = null;
                previous = now;
                result = func.apply(context, args);
                if (!timeout) context = args = null;
                //如果延迟执行不存在，且没有设定结尾边界不执行选项
            } else if (!timeout && options.trailing !== false) {
                timeout = setTimeout(later, remaining);
            }
            return result;
        };
    };
    //分析参数
    function extend(custom, src){
        var result = {};
        for(var attr in src){
            result[attr] = custom[attr] || src[attr]
        }
        return result;
    }
    //绑定事件，兼容处理
    function bindListener(element, type, callback){
        if (element.addEventListener) {
            element.addEventListener(type, callback);
        }else if (element.attachEvent) {
            element.attachEvent('on'+type, callback)
        }else{
            element['on'+type] = callback;
        }
    }
    //解绑事件，兼容处理
    function removeListener(element, type, callback){
        if (element.removeEventListener) {
            element.removeEventListener(type, callback);
        }else if (element.detachEvent) {
            element.detachEvent('on'+type, callback)
        }else{
            element['on'+type] = null;
        }
    }
    //判断一个元素是否为DOM对象，兼容处理
    function isElement(o) {
        if(o && (typeof HTMLElement==="function" || typeof HTMLElement==="object") && o instanceof HTMLElement){
            return true;
        }else{
            return (o && o.nodeType && o.nodeType===1) ? true : false;
        };
    };
    var lazyload = function(options){
        //辅助变量
        var images = [],
            doc = document,
            body = document.body,
            winHeight = screen.availHeight;
        //参数配置
        var opt = extend(options, {
            wrapper: body,
            selector: 'img',
            imgSrc: 'data-src',
            defaultSrc: ''
        });
        if (!isElement(opt.wrapper)) {
            console.log('not an HTMLElement');
            if(typeof opt.wrapper != 'string'){
                //若 wrapper 不是DOM对象 或者不是字符串，报错
                throw new Error('wrapper should be an HTMLElement or a selector string');
            }else{
                //选择器
                opt.wrapper = doc.querySelector(opt.wrapper) || body;
            }
        }
        //查找所有需要延时加载的图片
        function getAllImages(selector){
            return Array.prototype.concat.apply([], opt.wrapper.querySelectorAll(selector));
        }
        //设置默认显示图片
        function setDefault(){
            images.map(function(img){
                img.src = opt.defaultSrc;
            })
        }
        //加载图片
        function loadImage(){
            var nowHeight = body.scrollTop || doc.documentElement.scrollTop;
            console.log(nowHeight);
            if (images.length > 0){
                images.map(function(img, index) {
                    if (nowHeight + winHeight > img.offsetTop) {
                        img.src = img.getAttribute(opt.imgSrc);
                        console.log('loaded');
                        images.splice(index, 1);
                    }
                })
            }else{
                removeListener(window, 'scroll', load)
            }
        }
        var load = throttle(loadImage, 250);
        return (function(){
            images = getAllImages(opt.selector);
            bindListener(window, 'scroll', load);
            opt.defaultSrc && setDefault()
            loadImage();
        })()
    };
    window.lazyload = lazyload;
})(window);
```
上述代码拷贝到项目中即可使用，使用方式：

```javascript
//使用默认参数
new lazyload();
//使用自定义参数
new lazyload({
    wrapper: '.article-content',
    selector: '.image',
    src: 'data-image',
    defaultSrc: 'example.com/static/images/default.png'
});
```
若在 IE8 中使用，没有 map 函数时，请在引用插件前加入下列处理 map 函数兼容性的代码：
```javascript
// 实现 ECMA-262, Edition 5, 15.4.4.19
// 参考: http://es5.github.com/#x15.4.4.19
if (!Array.prototype.map) {
    Array.prototype.map = function(callback, thisArg) {
        var T, A, k;
        if (this == null) {
            throw new TypeError(" this is null or not defined");
        }
        // 1. 将O赋值为调用map方法的数组.
        var O = Object(this);
        // 2.将len赋值为数组O的长度.
        var len = O.length >>> 0;
        // 3.如果callback不是函数,则抛出TypeError异常.
        if (Object.prototype.toString.call(callback) != "[object Function]") {
            throw new TypeError(callback + " is not a function");
        }
        // 4. 如果参数thisArg有值,则将T赋值为thisArg;否则T为undefined.
        if (thisArg) {
            T = thisArg;
        }
        // 5. 创建新数组A,长度为原数组O长度len
        A = new Array(len);
        // 6. 将k赋值为0
        k = 0;
        // 7. 当 k < len 时,执行循环.
        while (k < len) {
            var kValue, mappedValue;
            //遍历O,k为原数组索引
            if (k in O) {
                //kValue为索引k对应的值.
                kValue = O[k];
                // 执行callback,this指向T,参数有三个.分别是kValue:值,k:索引,O:原数组.
                mappedValue = callback.call(T, kValue, k, O);
                // 返回值添加到新数组A中.
                A[k] = mappedValue;
            }
            // k自增1
            k++;
        }
        // 8. 返回新数组A
        return A;
    };
}
```
Enjoy!!!
