
### 1.从简单的匿名插槽开始
```
Vue.component('button-counter', {
  template: '<div> <slot>我是默认内容</slot></div>'
})
```
```
new Vue({
	el: '#app',
	template: '<button-counter><span>我是slot传入内容</span></button-counter>'
})
```
这里先注册一个`button-counter`组件，然后使用它，上面渲染出来的结果是 我是slot内容

原理解析：

1.这里直接看到组件`button-counter`编译后的`render`函数，就不详细从`template`模板编译说起，直接看最后的render函数：
```
(function anonymous(
) {
with(this){return _c('div',[_t("default",[_v("我是默认内容")])],2)}
})
```

这里`_v`就是创建普通文本节点，主要看`_t`函数，这里`_t`也就是`renderSlot`函数的简写

```
  function installRenderHelpers (target) {
    ...
    target._t = renderSlot;
    ...
  }
```

```
 function renderSlot (
    name,
    fallback,
    props,
    bindObject
  ) {
    var scopedSlotFn = this.$scopedSlots[name];
    var nodes;
    nodes = scopedSlotFn(props) || fallback;
    return nodes;
  }
```

这里把函数`renderSlot`函数简化下，其它情况先不看，这里会把`name`和`fallback`传进来，
这里先说一下`name`属性，我们上面定义是
```
Vue.component('button-counter', {
  template: '<div><slot>我是默认内容</slot></div>'
})
```
这里`slot`没有给`props`为`name`的值，所以默认是`default`，我们可以给它一个`name`值，例如

```
Vue.component('button-counter', {
  template: '<div><slot name="header">我是默认内容</slot></div>'
})
```

那么使用时
```
new Vue({
	el: '#app',
	template: '<button-counter><span slot="header">我是slot传入内容</span></button-counter>'
})
```

但是上面的用法`slot`在2.6.0版本已经废弃，提供了新的`v-slot`代替，但是你仍然可以这么写，源码中依然保存对slot的兼容处理，我们看下用`v-slot`的写法以及需要注意的地方
```
new Vue({
	el: '#app',
	template: '<button-counter><template v-slot:header><span>我是slot传入内容</span></template></button-counter>'
})
```
注意这`里v-slot`需要使用在`template`，不可再跟上面一样直接作用于`span`标签上面

回到上面说的`renderSlot`函数，`name`这里是默认值`default`，第二个参数`fallback`就是我们在组件中的`slot`节点的默认值
```
var scopedSlotFn = this.$scopedSlots[name];
var nodes;
nodes = scopedSlotFn(props) || fallback;
return nodes;
```

如果`this.$scopredSlots`存在该`name`的值，则调用该返回函数生成一个`nodes`节点返回，否则返回`fallback`(即默认值)，所以到这里我们知道为什么在自定义组件中有传入子元素就渲染子元素，没有就使用默认插槽里面的值了，这里涉及到`this.$scopredSlots`这个变量，我们接下来看下这个值，我们首先看下`vm.$slots` 
当组件在执行`initRender`函数时
```
function initRender (vm) {
  ...
  vm.$slots = resolveSlots(options._renderChildren, renderContext);
  ...
}
```

`resolveSlots`函数会对`children`节点做归类和过滤处理，返回`slots`

```
function resolveSlots (
    children,
    context
  ) {
    if (!children || !children.length) {
      return {}
    }
    var slots = {};
    for (var i = 0, l = children.length; i < l; i++) {
      var child = children[i];
      var data = child.data;
      // remove slot attribute if the node is resolved as a Vue slot node
      if (data && data.attrs && data.attrs.slot) {
        delete data.attrs.slot;
      }
      // named slots should only be respected if the vnode was rendered in the
      // same context.
      if ((child.context === context || child.fnContext === context) &&
        data && data.slot != null
      ) {
        // 如果slot存在(slot="header") 则拿对应的值作为key
        var name = data.slot;
        var slot = (slots[name] || (slots[name] = []));
        // 如果是tempalte元素 则把template的children添加进数组中，这也就是为什么你写的template标签并不会渲染成另一个标签到页面
        if (child.tag === 'template') {
          slot.push.apply(slot, child.children || []);
        } else {
          slot.push(child);
        }
      } else {
        // 如果没有就默认是default
        (slots.default || (slots.default = [])).push(child);
      }
    }
    // ignore slots that contains only whitespace
    for (var name$1 in slots) {
      if (slots[name$1].every(isWhitespace)) {
        delete slots[name$1];
      }
    }
    return slots
  }
```
这里的`_renderChildren`就是把`children`的`name`相同的`VNodes`节点归类放到一个数组中，最终返回了一个对象，`key`为对应的`name`，值是包含该`name`对应的节点数组

接下来的`_render`函数的时候，通过`normalizeScopedSlots`得到`vm.$scopedSlots`

```
vm.$scopedSlots = normalizeScopedSlots(
  _parentVnode.data.scopedSlots,
  vm.$slots,
  vm.$scopedSlots
);
```

### 作用域插槽
有时我们需要获取子组件的一些内部数据，比如获取下面`msg`的值，但是当前执行的作用域是在父组件的实例上，通过`this.msg`是获取不到子组件里面的值，我们需要在`<slot>`元素上动态绑定一个`msg`对象属性，然后在父组件可以通过下面方式来获取

```
Vue.component('button-counter', {
  data () {
    return {
        msg: 'hi'
    }
 },
  template: '<div><slot :msg="msg">我是默认内容</slot></div>'
})
```
```
new Vue({
  el: '#app',
  template: '<button-counter><template scope="msg"><span>{{props.msg}}</span></template></button-counter>'
})
```

这是2.5以前的写法，2.5以后采用slot-scope

```
new Vue({
  el: '#app',
  template: '<button-counter><template slot-scope="msg"><span>{{props.msg}}</span></template></button-counter>'
})
```

2.6以后废弃上面两种写法，采用v-slot
```
new Vue({
  el: '#app',
  template: '<button-counter><template v-slot="msg"><span>{{props.msg}}</span></template></button-counter>'
})
```

这里能够拿到`msg`是因为在`renderSlot`的时候 执行会传入`props`，可以看到编译后的`render`函数`_t`的第三个参数就是`props`

```
(function anonymous(
) {
with(this){return _c('div',[_t("default",[_v("我是默认的")],{"msg":msg})],2)}
})
```

```
var scopedSlotFn = this.$scopedSlots[name];
var nodes;
nodes = scopedSlotFn(props) || fallback;
return nodes;
```

由`render`函数可以看到传进去的`props`后就能够拿到`msg`的值了
```
(function anonymous(
) {
with(this){return _c('button-counter',{scopedSlots:_u([{key:"default",fn:function(props){return [_c('span',[_v(_s(props.msg))])]}}])})}
})
```

### 最后
这里只是简单描述了几个关键点，插槽作用域还有支持解构赋值，支持动态插槽名称等写法，还有很多的细节处理需要更加深入去了解源码。