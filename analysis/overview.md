在阅读一个框架/库的源码时，由外而内，一层一层拆分理解，明白作者的设计思路之精妙后再去研究细节实现比较有趣（当然不是先看脸，再看内在，肤浅！）。
### 外层结构
与许多第三方库一样，underscore.js也是通过IIFE来包裹自己的业务逻辑

```
(function(){
    ...
}.call(this))
```
并传入 ```this``` (浏览器环境下相当于window)来改变函数的作用域。
### 调用方式
```underscore.js``` 既支持普通函数调用，也支持oop形式调用。
```
const _ = require('underscore');
let funcMax = _.max([1, 3, 5]);
let oopMax = _([1, 3, 5]).max()
```
看到 ```require``` 自然就想到了 ```module.exports```, 源码的实现在53~60行
```
// 外层if-else区分当前环境是node环境还是浏览器环境
// nodeType是为了确定是否为html元素
if (typeof exports != 'undefined' && !exports.nodeType) {
    if (typeof module != 'undefined' && !module.nodeType && module.exports) {
      // 导出_时，module.exports指向_, 导致exports与module.exports之间的连接中断，需要重新建立连接
      exports = module.exports = _;
    }
    // 兼容对module.exports兼容不好的浏览器
    exports._ = _;
} else {
    // 浏览器环境下，挂载在root上
    root._ = _;
}
```
看完上述代码不禁想 ```_``` 和 ```root```是什么？

#### root object：root
```root``` 定义在6~9行
```
var root = typeof self == 'object' && self.self === self && self ||
    typeof global == 'object' && global.global === global && global ||
    this ||
    {};
```
```root``` 表示的是在不同环境下的顶级对象，如浏览器下的```'window'('self')```， 服务端的```global```，虚拟机上的```this```。
因此在上述导出_对象时，若执行环境为````node```，则```exports = module.exports = _ ```，若执行环境为浏览器，则将 _ 对象挂载在```window```上。

#### undersource object： _
```_``` 定义在 42~46行
```
var _ = function(obj) {
    // 若obj已经是_的实例，直接返回
    if (obj instanceof _) return obj;
    // 当前作用域不是_，生成_实例
    if (!(this instanceof _)) return new _(obj);
    // 参数挂载在实例的_wrapped属性上
    this._wrapped = obj;
};
```
```_```其实是一个构造函数，而且支持“无new构造”``` _([1, 3, 5])```的结果就是生产了一个_的实例，该实例有个``` _wrapped```属性，属性值是```[1, 3, 5]```。
实例本身要调用```max ```方法，其本身又没有这个方法，那么应该来自于原型链，也就是说```_.prototype```上应该有这个方法。但是纵观全文，又没有```_.prototype.max = function() {...} ```
那么，方法是如何挂载在_上的呢？

### 方法挂载
在```_```中定义方法的形式为```_.max = function() {...}``` ，也就是说这些方法已挂载在了```_``` 对象上，那么遍历```_```上的属性，若该属性值的类型为函数的话，那么就将该属性挂载在```_.prototype```上即可。

源码中用来完成这件事的是```_.mixin```方法 1633~1646行
```
_.mixin = function(obj) {
    _.each(_.functions(obj), function(name) {
      // 方法挂载在 _[name] 上
      var func = _[name] = obj[name];
      _.prototype[name] = function() {
        // oop调用时存储的参数
        var args = [this._wrapped];
        // arguments为name方法需要的其他参数
        push.apply(args, arguments);
        // 支持链式调用
        return chainResult(this, func.apply(_, args));
      };
    });
    return _;
};
_.mixin(_);
```
其中，```_.functions``` 返回一个对象中所有的属性值为函数的属性，源码实现在1066~1072行。
```
_.functions = _.methods = function(obj) {
    var names = [];
    for (var key in obj) {
      if (_.isFunction(obj[key])) names.push(key);
    }
    return names.sort();
};
```
再看看 ```ChainResult``` 链式调用