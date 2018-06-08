我们先看一下运行结果

![](http://www.imaoda.com/s/img/github/mvvm.gif)

name 和 age 被响应式的渲染出来，在 2s 后我们修改了 name 的值，同样能在页面正确更新。

我们来看一下最小化的 MVVM 的源码

```js
class Vue{
    constructor(opt){
        this.opt = opt
        this.observe(opt.data)
        let root = document.querySelector(opt.el)
        this.compile(root)
    }
    // 为响应式对象 data 里的每一个 key 绑定一个观察者对象
    observe(data){ 
        Object.keys(data).forEach(key => {
            let obv = new Observer() 
            data["_"+key] = data[key]
            // 通过 getter setter 暴露 for 循环中作用域下的 obv，闭包产生
            Object.defineProperty(data, key, {
                get(){
                    Observer.target && obv.addSubNode(Observer.target);
                    return data['_'+key]
                }, 
                set(newVal){
                    obv.update(newVal)
                    data['_'+key] = newVal
                }
            })
        })
    }
    // 初始化页面，遍历 DOM，收集每一个key变化时，随之调整的位置，以观察者方法存放起来    
    compile(node){
        [].forEach.call(node.childNodes, child =>{
            if(!child.firstElementChild && /\{\{(.*)\}\}/.test(child.innerHTML)){
                let key = RegExp.$1.trim()
                child.innerHTML = child.innerHTML.replace(new RegExp('\\{\\{\\s*'+ key +'\\s*\\}\\}', 'gm'),this.opt.data[key]) 
                Observer.target = child
                this.opt.data[key] 
                Observer.target = null
            }
            else if (child.firstElementChild) 
            this.compile(child)
        })
    }    
}
// 常规观察者类
class Observer{
    constructor(){
        this.subNode = []    
    }
    addSubNode(node){
        this.subNode.push(node)
    }
    update(newVal){
        this.subNode.forEach(node=>{
            node.innerHTML = newVal
        })
    }
}
```

#### 【重点分析】接下来梳理一下这段代码的思路，顺便了解一下 MVVM 闭包的艺术：

- [observe 函数]：首先我们会对需要响应式的 `data` 对象进行 for 循环遍历，为 `data` 的每一个 `key` 映射一个观察者对象
    - 在 ES6 中，for 循环每次执行，都可以形成闭包，因此这个观察者对象就存放在闭包中
    - 闭包形成的本质是 **内层作用域中堆地址暴露**，这里我们巧妙的用 getter/setter 函数暴露了 for 循环里的观察者 
- [compile 函数]：我们从根节点向下遍历 DOM，遇到 mustache 形式的文本，则映射成 data.key 对应的值，同时记录到观察者中
    - 当遍历到 {{xxx}} 形式的文本，我们正则匹配出其中的变量，将它替换成 data 中的值
    - 为了满足后续响应式的更新，将该节点存储在 key 对应的观察者对象中，我们用 getter 函数巧妙的操作了闭包
- 在页面初次渲染之后，后续的 `eventLoop` 中，如果修改了 `key` 的值，实际会通过 setter 触发观察者的 update 函数，完成响应式更新

#### 附上述演示案例的完整代码

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
</head>
<body>
    <div id='app'>
        <h3>姓名</h3>
        <p>{{name}}</p>
        <h3>年龄</h3>
        <p>{{age}}</p>
    </div>
</body>
</html>
<script>
document.addEventListener('DOMContentLoaded', function(){
    let opt = {el:'#app', data:{name:'检索中...', age:30}}
    let vm = new Vue(opt)
    setTimeout(() => {
        opt.data.name = '王永峰'
    }, 2000);
}, false)
class Vue{
    constructor(opt){
        this.opt = opt
        this.observe(opt.data)
        let root = document.querySelector(opt.el)
        this.compile(root)
    }
    // 为响应式对象 data 里的每一个 key 绑定一个观察者对象
    observe(data){ 
        Object.keys(data).forEach(key => {
            let obv = new Observer() 
            data["_"+key] = data[key]
            // 通过 getter setter 暴露 for 循环中作用域下的 obv，闭包产生
            Object.defineProperty(data, key, {
                get(){
                    Observer.target && obv.addSubNode(Observer.target);
                    return data['_'+key]
                }, 
                set(newVal){
                    obv.update(newVal)
                    data['_'+key] = newVal
                }
            })
        })
    }
    // 初始化页面，遍历 DOM，收集每一个key变化时，随之调整的位置，以观察者方法存放起来    
    compile(node){
        [].forEach.call(node.childNodes, child =>{
            if(!child.firstElementChild && /\{\{(.*)\}\}/.test(child.innerHTML)){
                let key = RegExp.$1.trim()
                child.innerHTML = child.innerHTML.replace(new RegExp('\\{\\{\\s*'+ key +'\\s*\\}\\}', 'gm'),this.opt.data[key]) 
                Observer.target = child
                this.opt.data[key] 
                Observer.target = null
            }
            else if (child.firstElementChild) 
            this.compile(child)
        })
    }    
}
// 常规观察者类
class Observer{
    constructor(){
        this.subNode = []    
    }
    addSubNode(node){
        this.subNode.push(node)
    }
    update(newVal){
        this.subNode.forEach(node=>{
            node.innerHTML = newVal
        })
    }
}
</script>
```
