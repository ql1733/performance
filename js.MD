## JS 优化


### 提升js文件加载速度

加载文件 css放到head中，js 放到body中


### js 函数与变量优化

* 尽量使用id 选择器

* 尽量避免eval

* js 函数尽量保持简洁

* 使用事件节流

* 使用事件委托

* 避免大量js 动画

* 尽量使用css3 动画

* 尽量使用 canvas 动画

* 合理使用requestAnimationFrame 替代 setTimeout, setInterval  
requestAnimationFrame 可以在正确的时间进行渲染，setTimeout, setInterval 不能保证函数执行的时机

### 合理使用缓存

* 合理缓存DOM 对象

* 缓存列表长度

* 缓存ajax 数据

### cookie

   cookie 随请求返回服务器和客户端,缓存大小有限制,一般用作会话管理，减少cookie大小.

sessionStorage 

创建一个本地存储，可用于页面传值，页面关闭数据会清除

localstorage

创建一个本地存储，可用于页面传值，页面关闭数据不会清除,比cookie 存储数据大

### indexedDB

索引数据库,可以在客户端存储大量结构化数据，没有网络情况下可使用