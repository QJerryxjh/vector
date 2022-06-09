#技术 #原理 #可观察 #observable-util 

## 故事
大四那一年我开始寻找前端相关的实习，在投递n份简历的情况下终于有一家小型公司发了面试邀约。漂亮的hr姐姐与我一番攀谈交心后，漂亮姐姐说你的底子还不错，让这边的前端主管给你进一步面试。
具体细节我已经记不清了，但我怎么也不会忘记他问的我一个问题。
他问：“你知道`vue`的双向绑定是怎么实现的吗？” 
我答：“应该是观察者模式吧”。
回答完之后我发现他的表情不对劲，我连忙说道：“或者是发布订阅模式吧”。
他说话了：“不对”。
我瞬间不淡定了，我好想知道答案，我好想知道这到底是如何实现的：“那是怎么实现的”。
他说了一遍答案：“&@%$&...”。
我没听清：”啊？啥？“  
”是JS实现的“
## 引言
分享一个迷你响应式库[`@nx-js/observer-util`](https://github.com/nx-js/observer-util)来了解如何实现响应式数据。这将让你了解：
- 怎么创建一个可观察对象
- 响应函数如何收集内部使用到的可观察值，又是如何在依赖值改变时触发响应
- 为什么状态管理库内的批处理是重要的
## 基本使用
### 示例
```js
import { observable, observe } from '@nx-js/observer-util'

// 创建可观察对象
const originObj = {
	name: 'nian',
	age: 1,
	abilities: {
		shakeHands: true
	}
}
const observableObj = observable(originObj)
// 添加reaction，自动运行
const reactionFn = observe(() => {
	console.log(observableObj.name)
}, {})

observableObj.name = 'gao'
```

`observable`:  接收一个值，可以将其创建为可观察，且是深度可观察
`observe`: 创建reaction，运行时收集依赖，在依赖变化后自动运行改reaction函数
- `observe` 接受一个函数。 在函数运行时，为其同步任务收集依赖，在依赖更改的时候，运行该函数；
- 返回reaction函数，可直接调用执行，执行期间可以重新收集依赖。
- 可接受第二个参数`options`选项如下：
	- `lazy` 如果赋`true`，则不会自动首次运行，只有在手动调用返回的`reactionFn`之后才会收集依赖进行相应
	- `debugger` 调试用，接受一个函数，每次有`operation`操作的时候，传入`operation`作为参数调用`debugger`，一般只在调试或者开发者工具内使用
	- `scheduler`调度函数
## 读源码
### 下文涉及术语、变量及其含义
- `reaction`: 对可观察对象的观察者函数，在目标发生变更时将会被调用，该库利用`observe`api接受函数所得的返回值就是一个reaction
- `reactionStack`: reaction执行栈，一个函数调用栈，保存的是正在运行的`reaction`，当某个`reaction`执行期间引发其他`reaction`观察目标变更，则新的`reaction`将会被调用，此时把该新调用的`reaction`压到栈顶。而正处于栈顶的`reaction`，则视为全局正在运行的`reaction`，在可观察对象属性发生变更时，会查看当前是否有正在运行的`reaction`，如果有，则视当前栈顶的`reaction`为自己的观察者，此时执行添加观察者操作。
- `handlers`: 传递给 new Proxy()的处理器，该库核心在于定义的`get/set`处理器
- `connectionStore`: 一个`weakMap`结构，其键为用于创建可观察对象的原始对象，值为一个`map`结构，该`map`结构用于存储可观察对象的键与其`reaction`（键为可观察对象的属性名，值为观察该属性的`reaction`）。用途是在可观察对象上的属性值变更的时候调用相应的`reaction`。
- `rawToProxy`: 一个`weakMap`结构，键为原始对象，值为使用原始对象创建的可观察对象引用。
- `proxyToRaw`: 一个`weakMap`结构，与`rawToProxy`相反。
### 创建可观察对象
该库并未考虑es5兼容，仅使用`proxy` api 实现观察者。
```js
/**
* 创建可观察对象，接受一个目标，目标为内部规定的map/set/array/object，返回观察值
*/
function createObservable (obj) {
	// ...
	// builtIns.getHandlers 区分出（map、set）和其他数据类型，非set、map数据使用基础处理器对象
	const handlers = builtIns.getHandlers(obj) || baseHandlers
	const observable = new Proxy(obj, handlers)
	// 在全局使用两个map对象，储存原始对象与可观察对象相互引用
	rawToProxy.set(obj, observable)
	proxyToRaw.set(observable, obj)

	// 为可观察对象创建属性的reaction
	storeObservable(obj)
	return observable
}
```
`baseHandlers`为一个处理器对象，该对象含有`get/set/has/ownKeys/deleteProperty`方法，下面列出其中的`get/set`方法的部分代码来理解其用途
```js
// baseHandlers 内容
{
	/**
	* 利用get来为reaction收集依赖
	*/
	get(target, key, receiver) {
		// 获取值
		const result = Reflect.get(target, key, receiver)	
		// 为目标对象key属性注册reaction，收集依赖
		registerRunningReactionForOperation({ target, key, receiver, type: 'get' })
		// 获取对应可观察对象
		const observableResult = rawToProxy.get(result)
		// ... 如果observableResult是可观察对象则返回，不是可观察对象时在该键为可配置时创建为可观察对象返回（因此实现深度可观察）
		return observableResult || result
	}，
	set(target, key, value, receiver) {
		// 目标对象上是否已经有此键
		const hadKey = hasOwnProperty.call(target, key)
		const oldValue = target[key]
		const result = Reflect.set(target, key, value, receiver)
		if (!hadKey) {
			// 之前没有该键值,则是创建,排队运行reactions
			queueReactionsForOperation({ target, key, value, receiver, type: 'add' })
		} else if (value !== oldValue) {
			// 是更新update,此时新旧值不相等,则排队运行reactions
			queueReactionsForOperation({
				target,
				key,
				value,
				oldValue,
				receiver,
				type: 'set'
			})
		}
		return result
	}
}
```

在访问可观察对象上的某个key属性时，会使用`handlers`中的`get`方法，此过程发生的事情有：
- **1**: 如果当前是在一个reaction中访问可观察对象上的属性（后文介绍该库如何判断是否在一个reaction中访问属性），此时为可观察对象的被访问属性添加观察者。
- **2**: 在`reaction`上的待清空观察的`cleaners`上存上该可观察对象被访问属性的`reactions`集合的引用，在需要取消观察的时候，在该集合上删除自身，这样在被观察对象该属性发生变更，触发`reactions`的时候不会再引发该`reaction`的调用。
```js
function registerReactionForOperation (reaction, { target, key, type }) {
	if (type === 'iterate') {
		key = ITERATION_KEY
	}
	// 对应上述过程发生的情形 1 ，为可观察对象被访问属性添加观察者`reaction`
	// 获取到目标对象的reaction的map结构
	const reactionsForObj = connectionStore.get(target)
	// 获取目标对象上键为[key]的对应的reactions
	let reactionsForKey = reactionsForObj.get(key)	
	// 如果没有map结构上没有对应该键的reactions集合
	if (!reactionsForKey) {
		// 创建一个空的集合并且赋值在map上
		reactionsForKey = new Set()
		reactionsForObj.set(key, reactionsForKey)
	}
	// save the fact that the key is used by the reaction during its current run
	if (!reactionsForKey.has(reaction)) {	
		// 如果当前reactions集合上没有该reaction,则在集合内添加上该reaction
		reactionsForKey.add(reaction)
		// 针对上述过程的情形 2，保留reactions集合引用，方便在当前reaction取消观察时候清楚该观察
		// 为reaction上的cleaners添加 针对该key的 reaction集合
		reaction.cleaners.push(reactionsForKey)
	}
}
```

### reaction 
利用`observe`创建一个`reaction`，在该`reaction`调用期间，执行的操作：释放原有依赖，重新收集依赖。在收集完依赖之后，当依赖发生变化，该reaction将会被再次调用。
```js
function observe (fn, options = {}) {
	// 如果传入的函数不是一个reaction函数,则用reaction包裹一下
	const reaction = fn[IS_REACTION]
		? fn
		: function reaction () {
			return runAsReaction(reaction, fn, this, arguments)
		}
	reaction.scheduler = options.scheduler
	reaction.debugger = options.debugger
	reaction[IS_REACTION] = true
	// 不是懒运行函数,则直接先运行一次,reaction只有在被运行一次之后才会收集到依赖,所以如果reaction之前没有运行过,observe监听的值不会被监听
	if (!options.lazy) {
		reaction()
	}
	return reaction
}

// 把传入的fn方法以`reaction`的形式运行
function runAsReaction (reaction, fn, context, args) {
	if (reaction.unobserved) {
		// 如果已经被取消观察了,则直接运行返回结果
		return Reflect.apply(fn, context, args)
	}
	// 如果reactionStack中已有该reaction,则不运行
	if (reactionStack.indexOf(reaction) === -1) {
		// 解绑,释放依赖
		releaseReaction(reaction)
		try {
			// 推到reaction栈顶
			reactionStack.push(reaction)
			// 运行fn
			return Reflect.apply(fn, context, args)
		} finally {		
			// 从reaction栈中推出
			reactionStack.pop()
		}
	}
}
```

## 拓展应用
[`react-easy-state`](https://github.com/RisingStack/react-easy-state)在`observable-util`的基础上，增加了对`react`组件响应式处理。
```js
ReactiveComp = (props) => {
	// 用于forceUpdate的state
	const [, setState] = useState();
	// 直接用observe api包裹一个组件，option参数传入scheduler和lazy参数
	const render = useMemo(
		() =>
		observe(originComponent, {
			// 每次originComponent内部依赖更改之后，都会调用scheduler方法，实现强制更新组件，组件更新之后会释放原有依赖，再次收集依赖
			scheduler: () => setState({}),
			// lazy 懒运行函数，仅当页面挂载该组件之后才开始进行收集依赖
			lazy: true,
		}),
		[originComponent],
	);
	// 卸载，解绑，释放依赖
	useEffect(() => {
		return () => unobserve(render);
	}, []);	
	isInsideFunctionComponent = true;
	try {
		return render(props);
	} finally {
		isInsideFunctionComponent = false;
	}
};
```

这与`mobx` 与`mobx-react(-lite)`类似，`mobx`实现要复杂的多，抛开兼容性及性能的差异，`mobx`还有批处理的概念。
`mobx`批处理：如果一个代码块连续修改store中的状态，可以使用`runInAction`函数进行包裹（或者是action/flow函数），在`runInAction`运行期间不会引发`reaction`调用，只有在`runInAction`运行结束之后，才会引发`reaction`。`observable-util`中却没有类似的批处理，当一个点击事件内部更改了多个可观察对象的值，此时每次修改都会引发`reaction`，如果是执行一个比较耗时的同步任务函数（组件渲染），性能将成问题。
`observable-util`和`react-easy-state`实现的思想或许可以对理解`mobx`有所帮助，也有助于理解响应式。
## 整理代码流程图
![[(img)获取值.png]]
![[(img)注册reaction.png]]
![[(img)设值.png]]
![[(img)排队运行reaction.png]]
![[(img)reaction调用.png]]
## 故事续集
hr姐姐再次与我攀谈，告知我：你的底子还不错，我们挺看好你的，我们愿意培养你，现在有几个方案让你选，一是你与我们签不低于五年的长约。二是你在这工作半年没有工资，随后一年无论你在哪工作，需要每个月交给我们一千块的培养费。
原来是个培训机构，甘霖凉。
