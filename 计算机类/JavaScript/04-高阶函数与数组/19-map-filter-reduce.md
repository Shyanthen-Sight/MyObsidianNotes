---
title: "19 · map、filter 与 reduce"
aliases:
  - map, filter & reduce
  - map filter and reduce
tags:
  - JavaScript
  - Namaste-JavaScript
  - 数组方法
episode: 19
source: https://github.com/alok722/namaste-javascript-notes
---

# 第 19 讲：map、filter 与 reduce（map, filter & reduce）

> [!abstract] 本讲要点
> - `map` 用来把数组的每个元素按给定逻辑做变换，返回一个由变换结果组成的新数组。
> - `filter` 只保留回调函数返回真值的元素。
> - `reduce` 把数组的所有值归约成一个输出，可用于求和、求最大值、统计次数等。
> - 三者都是高阶函数，并且可以链式组合使用。

> [!note] 前提
> `map`、`filter` 与 `reduce` 都是高阶函数（Higher-Order Function）。

## 一、map 函数

它主要用来对数组做变换。`map()` 方法会为数组的每个元素调用一次函数，并用这些调用的结果组成一个新数组。

```js
const output = arr.map(_function_) // 这个 _function_ 告诉 map：我想对数组的每个元素做什么变换
```

```js
const arr = [5, 1, 3, 2, 6];
// 任务 1：把数组元素翻倍：[10, 2, 6, 4, 12]
function double(x) {
  return x * 2;
}
const doubleArr = arr.map(double); // map 内部会对数组的每个元素执行 double 函数，创建一个新数组并返回它。
console.log(doubleArr); // [10, 2, 6, 4, 12]
```

```js
// 任务 2：把数组元素变为原来的三倍
const arr = [5, 1, 3, 2, 6];
// 变换逻辑
function triple(x) {
  return x * 3;
}
const tripleArr = arr.map(triple);
console.log(tripleArr); // [15, 3, 9, 6, 18]
```

```js
// 任务 3：把数组元素转换成二进制
const arr = [5, 1, 3, 2, 6];
// 变换逻辑：
function binary(x) {
	return x.toString(2);
}
const binaryArr = arr.map(binary);

// 上面的代码可以改写成：
const binaryArr = arr.map(function binary(x) {
	return x.toString(2);
}

// 或者使用箭头函数
const binaryArr = arr.map((x) => x.toString(2));
```

所以，`map` 函数本质上就是把每一个值都映射一遍，并按照给定的条件对它做变换。

## 二、filter 函数

`filter` 函数主要用来过滤数组中的值。`arr.filter()` 方法会基于给定数组创建一个新数组，新数组只包含原数组中满足参数函数所设定条件的那些元素。

```js
const array = [5, 1, 3, 2, 6];
// 过滤出奇数
function isOdd(x) {
  return x % 2;
}
const oddArr = array.filter(isOdd); // [5,1,3]

// 上面写法的另一种写法：
const oddArr = arr.filter((x) => x % 2);
```

`filter` 函数会创建一个数组，只保存求值为 `true` 的那些值。

> [!warning] 注意
> `filter` 判断的依据是回调函数返回值的真假（truthy / falsy），而不是元素本身的值。

## 三、reduce 函数

它接收数组的所有值，并给出一个单一的输出，也就是把数组「归约（reduce）」成一个结果。

```js
const array = [5, 1, 3, 2, 6];
// 计算数组元素之和 —— 非函数式编程写法
function findSum(arr) {
  let sum = 0;
  for (let i = 0; i < arr.length; i++) {
    sum = sum + arr[i];
  }
  return sum;
}
console.log(findSum(array)); // 17

// 使用 reduce 的写法
const sumOfElem = arr.reduce(function (accumulator, current) {
  // current 表示数组当前元素的值
  // accumulator 用来累积数组成员的计算结果。
  // 对比上一段代码，sum 变量相当于 accumulator，arr[i] 相当于 current
  accumulator = accumulator + current;
  return accumulator;
}, 0); // 在上面的例子中 sum 被初始化为 0，所以这里的 accumulator 也需要初始化，reduce 函数的第二个参数就表示这个初始值。
console.log(sumOfElem); // 17
```

```js
// 求数组中的最大值：非函数式编程写法：
const array = [5, 1, 3, 2, 6];
function findMax(arr) {
    let max = 0;
    for(let i = 0; i < arr.length; i++ {
        if (arr[i] > max) {
            max = arr[i]
        }
    }
    return max;
}
console.log(findMax(array)); // 6

// 使用 reduce
const output = arr.reduce((acc, current) => {
	if (current > acc ) {
		acc = current;
	}
	return acc;
}, 0);
console.log(output); // 6

// acc 只是一个标签，表示目前为止累积的值，
// 所以在这个场景里也可以把它命名为 max
const output = arr.reduce((max, current) => {
	if (current > max) {
		max= current;
	}
	return max;
}, 0);
console.log(output); // 6
```

> [!tip] accumulator 的初始值
> `reduce` 的第二个参数是累积值的初始值。上面求和的例子中 `sum` 被初始化为 0，所以这里也要给 `accumulator` 一个初始值 0。

## 四、棘手的 map 用法（Tricky MAP）

```js
const users = [
	{ firstName: "Alok", lastName: "Raj", age: 23 },
	{ firstName: "Ashish", lastName: "Kumar", age: 29 },
	{ firstName: "Ankit", lastName: "Roy", age: 29 },
	{ firstName: "Pranav", lastName: "Mukherjee", age: 50 },
];
// 获取全名数组：["Alok Raj", "Ashish Kumar", ...]
const fullNameArr = users.map((user) => user.firstName + " " + user.lastName);
console.log(fullNameArr); // ["Alok Raj", "Ashish Kumar", ...]

----------------------------------------------------------

// 统计有多少个不同年龄的人，输出形如：
// {29 : 2, 75 : 1, 50 : 1}
// 这里应该用 reduce，为什么？因为我们要从数组中推导出一些信息，
// 本质上是想得到单个对象作为输出
const report = users.reduce((acc, curr) => {
	if(acc[curr.age]) {
		acc[curr.age] = ++ acc[curr.age] ;
	} else {
		acc[curr.age] = 1;
	}

	return acc;  // 每次都返回更新后的对象
}, {})
console.log(report) // {29 : 2, 75 : 1, 50 : 1}
```

## 五、函数链式调用（Function Chaining）

```js
// 获取所有年龄小于 30 岁的人的名字
const users = [
  { firstName: "Alok", lastName: "Raj", age: 23 },
  { firstName: "Ashish", lastName: "Kumar", age: 29 },
  { firstName: "Ankit", lastName: "Roy", age: 29 },
  { firstName: "Pranav", lastName: "Mukherjee", age: 50 },
];

// 函数链式调用
const output = users
  .filter((user) => user.age < 30)
  .map((user) => user.firstName);
console.log(output); // ["Alok", "Ashish", "Ankit"]

// 课后挑战：用 reduce 实现同样的逻辑
const output = users.reduce((acc, curr) => {
  if (curr.age < 30) {
    acc.push(curr.firstName);
  }
  return acc;
}, []);
console.log(output); // ["Alok", "Ashish", "Ankit"]
```

> [!question] 课后挑战
> 不用链式调用，只用 `reduce` 实现与上面相同的逻辑。

> [!abstract] 本讲小结
> - `map`：逐个变换元素，返回新数组。
> - `filter`：按条件筛选元素，返回新数组。
> - `reduce`：把整个数组归约成单个输出。
> - `filter` 与 `map` 可以链式组合，先筛选再变换。

---

> [!info] 🎬 配套视频
> [在 YouTube 观看本讲](https://www.youtube.com/watch?v=zdp0zrpKzIE&list=PLlasXeu85E9cQ32gLCvAvr9vNaUccPVNP)
