## js 实现 promise 串行执行

#### 题目

假设一个数组tasks: Promise[] (每一项都是一个Promise的数组) 
实现一个方法 , 将该tasks内的任务依次执行, 并返回一个结果为数组的Promise, 该数组为任务执行结果(已执行顺序排序)

#### 思考
demo的核心思想是: ```Promise.prototype.then```返回的是一个```Promise```对象

#### 解法

测试用例
```
const tasks = [
	new Promise((resolve) => {
		setTimeout(() => {
			resolve(3)
		}, 3000)
	}),
	new Promise((resolve) => {
		setTimeout(() => {
			resolve(2)
		}, 2000)
	}),
	new Promise((resolve) => {
		setTimeout(() => {
			resolve(1)
		}, 1000)
	})
]
```
1. 普通循环
```
function exectue(tasks){
 let p = Promise.resolve(tasks.shift())
 const result = []
 tasks.forEach(v => {
 	p = p.then((data) => {
 		result.push(data)
 		 return v
 	})

 })
 return p.then(data => {
 	result.push(data)
 	return result
 })
}
exectue(tasks).then(data=>{
  console.log(data) //[3,2,1]
})
```
2. async/await
```
async function exectue(tasks){
	const result = []
	for (let i = 0; i < tasks.length; i++) {
		const res = await tasks[i]
		console.log(res)
		result.push(res)
		if(result.length === tasks.length) return Promise.resolve(result)	
	}
}
exectue(tasks).then(data=>{
  console.log(data) //[3,2,1]
})
```
3. while

```
async function exectue(tasks){
	const result = []
	while(tasks.length) {
		let item = tasks.shift()
		result.push(await item)
	}
	return Promise.resolve(result)
}
exectue(tasks).then(data=>{
  console.log(data) //[3,2,1]
})
```