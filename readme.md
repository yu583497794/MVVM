# 使用自定义的MVVM实现Todos

## 实现效果

1. 改变待做项的状态\
通过单击每项待做事件前端的圆圈可切换事件状态;
![改变待做事件的状态](http://image.little-fairy.club/example.gif)
2. 增加待做事件\
通过在输入框中键入待做事件,以@开头作为截止时间,&开头作为共同参与事件的合作者;
![增加待做事件](http://image.little-fairy.club/addTodo.gif)
3. 删除待做事件\
鼠标移动到所需删除事件项, 点击x键即可删除;
![删除待做事件](http://image.little-fairy.club/del.gif)

## MVVM模型

### Model

数据层,往往指对象。

### View

视图层, 是直接呈现给用户的部分, 在前端中就是HTML;

### View Model

视图层应该和数据层完全分离。当对Model进行修改的时候, ViewModel会把修改自动同步到View层去; 同样当修改View, Model同样被ViewModel自动修改。

## MVVM实现

### 解析模板

1. 定义模板关键字

* data-model——用于将DOM的文本节点替换为制定内容
* data-class——用于将DOM的className替换为制定内容
* data-list——用于标识接下来将出现一个列表, 列表为制定结构
* data-list-item——用于标识列表项的内部结构
* data-event——用于为DOM节点绑定指定事件

2. 编写模板

通过在元素中设置模板关键字属性,将DOM与数据在数据结构中的路径关联在一起。

```html
<ul data-list="todos">
  <li data-list-item="todos">
    <p data-model="todos.content"></p>
    <ul data-list="todos:members">
      <li data-list-item="todos:members">
        <p data-model="todos:member:name"></p>
      </li>
    </ul>
  </li>
</ul>
```

模版中的模板关键字会被解析, 可以类比Vue。

```html
<!-- 将Model中name属性绑定到span的文本节点 -->
<!-- Vue的一种表示 -->
<p>Hello, <span>{{name}}</span>!</p>
<!-- Vue的另一种表示 -->
<p>Hello, <span v-text="name"></span>!</p>
<!-- 使用自定义MVVM -->
<p>Hello, <span data-text="name"></span>!</p>
```

3. 遍历DOM树

使用递归深度优先遍历

```javascript
function scan(node) {
  if (!node.getAttribute('data-list-item')) {
    for (let i = 0; i < node.children.length; i++) {
      let child = node.children[i];
      scan(node);
    }
    // 解析Model
    parseModel(node);
    // 解析ClassName
    parseClass(node);
    // 解析Event
    parseEvent(node);
  } else {
    parseList(node);
  }
}
```

parseModel, parseClass, parseEvent通过解析模板关键字, 获得数据在Model中的位置, 然后均通过向parseData方法传入的路径, 返回Model中对应数据的值。然后用不同的方法将数据渲染到View上\
在解析列表时, 首先找到列表中,列表项的根节点,即带有关键字"data-list-item"的节点, 解析关键字, 通过parseData得到绑定的数组对象, 依据数组大小, 多次深度复制该列表项根节点, 并为每一个列表复制节点记录路径,即与其绑定的数组的索引。由于存在多重列表,因此若该复制节点的父元素也存在索引路径, 为得到完整的索引路径,必须把父节点的索引路径也保留到当前索引路径前。之后为复制的列表项填充数据, 即继续深度遍历解析其子节点, 将填充数据完毕的列表项根节点依次插入到解析到的最初的列表项根节点, 最后从DOM树中移除这个节点, 完成对列表的解析。\

### 解析数据

在本项目中,数据对象为:

```javascript
{
  title: 'todo list',
  user: 'Lucy',
  todos: [{
      creator: 'Lucy',
      content: 'write mvvm',
      done: 'undone',
      date: '2019-03-04',
      members: [{
        name: 'Micheal'
      }, {
        name: 'Jean'
      }]
    }
}
```

parseData(str, node)方法接收解析关键字和传入的节点的索引路径获得节点绑定的数据在Model中的位置, 首先通过关键字所得路径寻址, 每当所得对象为数组时,就查询列表中节点存储的路径,直到返回保存了值和完整的路径的对象。

### 绑定模版与数据

数据劫持，指的是在访问或者修改对象的某个属性时，通过一段代码拦截这个行为，进行额外的操作或者修改返回结果。比较典型的是 Object.defineProperty() 和 ES2015 中新增的 Proxy 对象。\
在深度遍历DOM树过程中,为某一节点填充完数据后, 继续通过数据劫持为该节点绑定模板与数据, 即定义数据变动时要通知的对象, 达到当数据发生变动时，数据会触发劫持时绑定的方法，对视图进行更新的效果。\
通过parseData计算出的路径, 能找到对象data中与DOM有关联的属性, 并根据模板关键字为其增加相应的属性描述符。这一过程是在observe(data,key,callback)方法中实现的。\
根据key的类型,若是数组表面传入的是一条路径, 则通过observePath进一步寻址,直到找到最后的对象和其要设置的属性;
否则, 根据要设置属性的值, 若为基本类型,则为该对象的该属性设置属性描述符,主要是在set属性中增添调用传入的回调函数callback,若不是基本元素,则:

#### [Object.defineProperty() 的问题主要有三个](https://blog.csdn.net/mmjinglin/article/details/85097794)：

1. 不能监听数组的变化\
变异方法(mutation method):会修改原来数组的方法, 如push, pop, shift, unshift, splice, sort, reverse等, 不会触发set
不能监听数组的变化
通过Object.defineProperty实现数据劫持。通过 Object.defineProperty() 来劫持各个属性的setter，getter，在数据变动时发布消息给订阅者，触发相应监听回调。
2. 必须遍历对象的每个属性, 即不能监测对象的变化
3. 必须深层遍历嵌套的对象

##### 解决不能监测数组变化

方法重写来实现数组的劫持。

```javascript
function observeArray(arr, callback) {
  const oar = ["push", "pop", "shift","unshift", "sort", "splice", "reverse"];
  const arrProto = Array.prototype;
  const hackProto = Object.create(Array.prototype);
  oam.forEach((method) => {
    Object.defineProperty(hackProto, method, {
      configurable: true,
      enumerable: true,
      writable: true,
      value: function (...arg) {
        let me = this;
        let old = arg.slice();
        let now = arrProto[method].apply(me, arg);
        callback(old, me, ...arg);
        return now;
      }
    })
  });
  arr.__proto__ = hackProto;
}
```

##### 解决必须遍历对象的每个对象

要配合 Object.keys() 和遍历\

```JavaScript
  function observeAllKey (obj, callback) {
    Object.keys(obj).forEach((key) => {
      observe(obj, key, callback);
    })
  }
```

#### 不同的callback

* data-model: 根据元素节点,修改元素节点的value(input元素)或innerHTML。
* data-list: 删除列表内所有的列表项,将原始的未填充数据的列表项重新加入,执行初次填充的步骤,最后仍要从DOM树中移除掉原始列表项。
* data-class: 通过node.classList.add(className)或node.classList.remove(className)修改
* data-event: 本身就是绑定事件监听,不需要额外的callback

## tips

### 用Object.prototype.toString.call(p)获得数据类型

若p为数组,返回'[object Array]';\
若p为对象,返回'[object Object]';\

### Object,defineProperty(obj, prop, descriptor) 方法

1. obj:要在其上定义属性的对象。
2. prop:要定义或修改的属性的名称。
3. descriptor:将被定义或修改的属性描述符。

#### 属性描述符

1. 数据描述符\
具有值的属性，该值可能是可写的，也可能不是可写的。

* configurable\
  当且仅当该属性的 configurable为true时，该属性描述符才能够被改变，同时该属性也能从对应的对象上被删除。默认值为false。
* enumerable\
  当且仅当该属性的enumerable为true时，该属性才能够出现在对象的枚举属性中。默认为 false。
* value\
  该属性对应的值。可以是任何有效的 JavaScript 值（数值，对象，函数等）。默认为 undefined。
* writable\
  当且仅当该属性的writable为true时，value才能被赋值运算符改变。默认为 false。

2. 存取描述符\
由getter-setter函数对描述的属性。

* configurable\
* enumerable\
* get
  一个给属性提供 getter 的方法，如果没有 getter 则为 undefined。当访问该属性时，该方法会被执行，方法执行时没有参数传入，但是会传入this对象（由于继承关系，这里的this并不一定是定义该属性的对象）。默认为undefined。
* set
  一个给属性提供 setter 的方法，如果没有 setter 则为 undefined。当属性值修改时，触发执行该方法。该方法将接受唯一参数，即该属性新的参数值。默认为undefined。

## 参考文献
  
  1. [230行实现一个简单的MVVM - Div.IO](https://div.io/topic/1890)
  2. [Object.defineProperty() - JavaScript | MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)
