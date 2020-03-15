# Vue组件间通信方式
## 快速原型开发
可以快速识别.vue文件封装组件插件等功能
```bash
sudo npm install @vue/cli -g
sudo npm install -g @vue/cli-service-global
vue serve App.vue
```
## 一.Props传递数据（父->子）
```tree
components
   ├── Grandson1.vue // 孙子1
   ├── Grandson2.vue // 孙子2
   ├── Parent.vue   // 父亲
   ├── Son1.vue     // 儿子1
   └── Son2.vue     // 儿子2
```
在父组件中使用儿子组件
```html
<template>
 <div>
  父组件:{{mny}}
  <Son1 :mny="mny"></Son1>
 </div>
</template>
<script>
import Son1 from "./Son1";
export default {
 components: {
  Son1
 },
 data() {
  return { mny: 100 };
 }
};
</script>
```

子组件接受父组件的属性
<template>
 <div>子组件1: {{mny}}</div>
</template>
<script>
export default {
 props: {
  mny: {
   type: Number
  }
 }
};
</script>


## 二.$emit使用(子->父)
父组件自定义一个方法,子组件通过回调的方式将修改的内容传递给父组件.
父组件:
```js
<template>
 <div>
  父组件:{{mny}}
  //这里自定义一个@input方法
  <Son1 :mny="mny" @input="change"></Son1>
 </div>
</template>
<script>
import Son1 from "./Son1";
export default {
 methods: {
  change(mny) {
   this.mny = mny;
  }
 },
 components: {
  Son1
 },
 data() {
  return { mny: 100 };
 }
};
</script>
```

子组件触发绑定自己身上的方法
子组件
```html
<template>
 <div>
  子组件获取父组件的mny数据: {{mny}}
//子组件调用$emit方法,参数1是父组件自定义的方法名,参数2是要改变的值
  <button @click="$emit('input',200)">更改</button>
 </div>
</template>
<script>
export default {
 props: {
  mny: {
   type: Number
  }
 }
};
</script>
```
总结:其实就是子组件中通过调用$emit()方法,里面参数1是写自定义方法属性名,参数2是要改变的值,然后在父组件自己的方法中来改变自己的data,从而改变父组件的数据.

> 下面介绍同步父子组件的数据的两个语法糖写法

### .sync 父组件事件语法糖
```js
//通常父组件自定义事件名会写成update名:值写法
 //子,注意点:事件名要用update
 <button @click="$emit('update:value',200)">更改</button>
//父,change中还要像上面那样赋值,比较多代码
<Son1  @update:value="change"></Son1>
//父,简写为下面:
<Son1 :value.sync="mny"></Son1>
```

### v-model 事件和值的语法糖
```js

//常规写法
<Son1 :value="mny" @input="mny=>this.mny=mny"></Son1>
//简写,注意点:父自定义事件名只能是input,子接收的子属性名只能是value
<Son1 v-model="mny"></Son1>
<template>
 <div>
  子组件1: {{value}} // 触发的自定义事件名只能是input
  <button @click="$emit('input',200)">更改</button>
 </div>
</template>
<script>
export default {
 props: {
  value: { // 接收到的属性名只能叫value
   type: Number
  }
 }
};
</script>
```


## 三.$parent、$children
继续将属性传递
​```html
<Grandson1 :value="value"></Grandson1>
<template>
 <div>
  孙子:{{value}}
  <!-- 调用父组件的input事件 -->
  <button @click="$parent.$emit('input',200)">更改</button>
 </div>
</template>
<script>
export default {
 props: {
  value: {
   type: Number
  }
 }
};
</script>
```

> 如果层级很深那么就会出现$parent.$parent.....我们可以封装一个$dispatch方法向上进行派发

### $dispatch
​```javascript
Vue.prototype.$dispatch = function $dispatch(eventName, data) {
  let parent = this.$parent;
  while (parent) {
    parent.$emit(eventName, data);
    parent = parent.$parent;
  }
};
```

既然能向上派发那同样可以向下进行派发

### $broadcast
```javascript
Vue.prototype.$broadcast = function $broadcast(eventName, data) {
  const broadcast = function () {
    this.$children.forEach((child) => {
      child.$emit(eventName, data);
      if (child.$children) {
        $broadcast.call(child, eventName, data);
      }
    });
  };
  broadcast.call(this, eventName, data);
};
```

## 四.$attrs、$listeners

### $attrs
批量向下传入属性
```html
<Son2 name="小珠峰" age="10"></Son2>

<!-- 可以在son2组件中使用$attrs属性,可以将属性继续向下传递 -->
<div>
  儿子2: {{$attrs.name}}
  <Grandson2 v-bind="$attrs"></Grandson2>
</div>


<template>
 <div>孙子:{{$attrs}}</div>
</template>
```

### $listeners
批量向下传入方法
```html
<Son2 name="小珠峰" age="10" @click="()=>{this.mny = 500}"></Son2>
<!-- 可以在son2组件中使用listeners属性,可以将方法继续向下传递 -->
<Grandson2 v-bind="$attrs" v-on="$listeners"></Grandson2>

<button @click="$listeners.click()">更改</button>
```

## 五.Provide & Inject
### Provide
在父级中注入数据

```javascript
provide() {
  return { parentMsg: "父亲" };
},
```
### Inject
在任意子组件中可以注入父级数据

```javascript
inject: ["parentMsg"] // 会将数据挂载在当前实例上
```

## 六.Ref使用
获取组件实例
```html
<Grandson2 v-bind="$attrs" v-on="$listeners" ref="grand2"></Grandson2>
mounted() { // 获取组件定义的属性
  console.log(this.$refs.grand2.name);
}
```

## 七.EventBus
用于跨组件通知(不复杂的项目可以使用这种方式)
```javascript
Vue.prototype.$bus = new Vue();
```

Son2组件和Grandson1相互通信

```javascript
 mounted() {
  this.$bus.$on("my", data => {
   console.log(data);
  });
 },
```

```javascript
mounted() {
  this.$nextTick(() => {
   this.$bus.$emit("my", "我是Grandson1");
  });
 },
```

## 八.Vuex通信
状态管理
![](https://www.fullstackjavascript.cn/vuex.png)



![](https://www.fullstackjavascript.cn/wx.png)