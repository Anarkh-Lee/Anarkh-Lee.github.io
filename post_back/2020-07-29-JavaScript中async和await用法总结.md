---
layout: post
title: 'JavaScript中async和await用法总结'
date: 2020-07-29
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: JavaScript



---

> JavaScript中async/await用法总结

### 0.前言（async function函数与await）

`async function` 用来定义一个返回 `AsyncFunction`对象的异步函数。异步函数是指通过事件循环异步执行的函数，它会通过一个隐式的 `Promise` 返回其结果。如果你在代码中使用了异步函数，就会发现它的语法和结构会更像是标准的同步函数。

#### 0.1语法

```javascript
async function name([param[, param[, ... param]]]) { statements }
```

* 参数
  * `name`：函数名称
  * `param`：传递给函数的参数
  * `statements`：函数体语句
* 返回值
  * 返回的`Promise`对象会运行执行（resolve）异步函数的返回结果，或者运行拒绝（reject）--如果一步函数抛出异常的话。

#### 0.2描述

一个async一步函数可以包含0个或者多个`await`指令，该指令会暂停异步函数的执行，并等待`Promise`执行，然后继续执行异步函数，并返回结果。解析过后的返回值在`Promise`中按照`await`表达式的返回值对待。使用async/await语法能让开发者在异步代码中也按照类似同步代码中一样的方式应用try-catch语句。

`await`关键字只在异步函数内有效。如果在异步函数外使用它，会抛出语法错误。

注意，当异步函数暂停时，它调用的函数会继续执行（收到异步函数返回的隐式Promise）。

`async/await`的目的是简化使用多个promise时的同步行为，并对一组Promises执行某些操作。正如Promises类似于结构化回调，`async/await`更像结合了generators和promises。

`async`函数通常返回一个`Promise`对象。如果`async`函数的返回值不是一个标准意义上的`Promise`对象，那它（该返回）一定是包裹在了一个`Promise`对象里面。

例如：

```javascript
async function foo() {
   return 1
}
```

等价于：

```javascript
function foo() {
   return Promise.resolve(1)
}
```

async函数的函数体可以被认为是由0个或者多个await表达式分割开来。第一行代码到第一个await表达式之间的代码是同步运行的。在这种情况下（函数体中没有await表达式），async函数会同步运行。如果在方法体内有await表达式，async方法将会异步执行。

例如：

```javascript
async function foo() {
   await 1
}
```

等价于：

```javascript
function foo() {
   return Promise.resolve(1).then(() => undefined)
}
```

在await表达式之后的代码可以被认为是存在在链式调用的then回调方法中。

#### 0.4示例

**简单例子**

```javascript
var resolveAfter2Seconds = function() {
  console.log("starting slow promise");
  return new Promise(resolve => {
    setTimeout(function() {
      resolve("slow");
      console.log("slow promise is done");
    }, 2000);
  });
};

var resolveAfter1Second = function() {
  console.log("starting fast promise");
  return new Promise(resolve => {
    setTimeout(function() {
      resolve("fast");
      console.log("fast promise is done");
    }, 1000);
  });
};

var sequentialStart = async function() {
  console.log('==SEQUENTIAL START==');

  // 1. Execution gets here almost instantly
  const slow = await resolveAfter2Seconds();
  console.log(slow); // 2. this runs 2 seconds after 1.

  const fast = await resolveAfter1Second();
  console.log(fast); // 3. this runs 3 seconds after 1.
}

var concurrentStart = async function() {
  console.log('==CONCURRENT START with await==');
  const slow = resolveAfter2Seconds(); // starts timer immediately
  const fast = resolveAfter1Second(); // starts timer immediately

  // 1. Execution gets here almost instantly
  console.log(await slow); // 2. this runs 2 seconds after 1.
  console.log(await fast); // 3. this runs 2 seconds after 1., immediately after 2., since fast is already resolved
}

var concurrentPromise = function() {
  console.log('==CONCURRENT START with Promise.all==');
  return Promise.all([resolveAfter2Seconds(), resolveAfter1Second()]).then((messages) => {
    console.log(messages[0]); // slow
    console.log(messages[1]); // fast
  });
}

var parallel = async function() {
  console.log('==PARALLEL with await Promise.all==');
  
  // Start 2 "jobs" in parallel and wait for both of them to complete
  await Promise.all([
      (async()=>console.log(await resolveAfter2Seconds()))(),
      (async()=>console.log(await resolveAfter1Second()))()
  ]);
}

// This function does not handle errors. See warning below!
var parallelPromise = function() {
  console.log('==PARALLEL with Promise.then==');
  resolveAfter2Seconds().then((message)=>console.log(message));
  resolveAfter1Second().then((message)=>console.log(message));
}

sequentialStart(); // after 2 seconds, logs "slow", then after 1 more second, "fast"

// wait above to finish
setTimeout(concurrentStart, 4000); // after 2 seconds, logs "slow" and then "fast"

// wait again
setTimeout(concurrentPromise, 7000); // same as concurrentStart

// wait again
setTimeout(parallel, 10000); // truly parallel: after 1 second, logs "fast", then after 1 more second, "slow"

// wait again
setTimeout(parallelPromise, 13000); // same as parallel
```

**对例子分析**

* await and parallelism（并行）

  在`sequentialStart`中，程序在第一个`await`停留了2秒，然后又在第二个`await`停留了1秒。直到第一个计时器结束后，第二个计时器才被创建。程序需要3秒执行完毕。

  
  在 `concurrentStart`中，两个计时器被同时创建，然后执行`await`。这两个计时器同时运行，这意味着程序完成运行只需要2秒，而不是3秒,即最慢的计时器的时间。

  但是 `await `仍旧是顺序执行的，第二个 `await` 还是得等待第一个执行完。在这个例子中，这使得先运行结束的输出出现在最慢的输出之后。

  如果你希望并行执行两个或更多的任务，你必须像在`parallel`中一样使用`await Promise.all([job1(), job2()])`。

* async/await和Promise#then对比及错误处理

  大多数异步函数也可以使用Promises编写。但是，在错误处理方面，`async`函数更容易捕获异常错误

  上面例子中的`concurrentStart`函数和`concurrentPromise`函数在功能上都是等效的。在`concurrentStart`函数中，如果任一`await`ed调用失败，它将自动捕获异常，异步函数执行中断，并通过隐式返回Promise将错误传递给调用者。

  在Promise例子中这种情况同样会发生，该函数必须负责返回一个捕获函数完成的`Promise`。在`concurrentPromise`函数中，这意味着它从`Promise.all([]).then()`返回一个Promise。事实上，在此示例的先前版本忘记了这样做！

  但是，`async`函数仍有可能然可能错误地忽略错误。
  以`parallel`异步函数为例。 如果它没有等待`await`（或返回）`Promise.all([])`调用的结果，则不会传播任何错误。
  虽然`parallelPromise`函数示例看起来很简单，但它根本不会处理错误！ 这样做需要一个类似于`return ``Promise.all([])`处理方式。

**使用async函数重写promise链**

返回 `Promise`的 API 将会产生一个 promise 链，它将函数肢解成许多部分。例如下面的代码：

```javascript
function getProcessedData(url) {
  return downloadData(url) // 返回一个 promise 对象
    .catch(e => {
      return downloadFallbackData(url)  // 返回一个 promise 对象
    })
    .then(v => {
      return processDataInWorker(v); // 返回一个 promise 对象
    });
}
```

可以重写为单个async函数：

```javascript
async function getProcessedData(url) {
  let v;
  try {
    v = await downloadData(url); 
  } catch (e) {
    v = await downloadFallbackData(url);
  }
  return processDataInWorker(v);
}
```

注意，在上述示例中，`return` 语句中没有 `await` 操作符，因为 `async function` 的返回值将被隐式地传递给 `Promise.resolve`。

**return await promiseValue与retrun promiseValue的比较**

返回值`隐式的传递给``Promise.resolve`，并不意味着`return await promiseValue;和return promiseValue;`在功能上相同。

看下下面重写的上面代码，在`processDataInWorker`抛出异常时返回了null：

```javascript
async function getProcessedData(url) {
  let v;
  try {
    v = await downloadData(url);
  } catch(e) {
    v = await downloadFallbackData(url);
  }
  try {
    return await processDataInWorker(v); // 注意 `return await` 和单独 `return` 的比较
  } catch (e) {
    return null;
  }
}
```

简单地写上`return processDataInworker(v);将导致在processDataInWorker(v)`出错时function返回值为`Promise`而不是返回null。`return foo;`和`return await foo;`，有一些细微的差异:`return foo;`不管`foo`是promise还是rejects都将会直接返回`foo`。相反地，如果`foo`是一个`Promise`，`return await foo;`将等待`foo`执行(resolve)或拒绝(reject)，如果是拒绝，将会在返回前抛出异常。

### 1.async和await作用

async 是“异步”的简写,而 await 可以认为是 async wait 的简写。

asyns用于申明一个function是异步的，而await用于等待一个异步方法执行完成。

await只能出现在async函数中。

#### 1.1async的作用

* 首先，需要了解async函数是如何处理它的返回值的。

  * 简单的使用return语句返回值（如果使用return返回，那么await就没有任何意义了）

    ```javascript
    async function testAsync() {
        return "hello async";
    }
    
    const result = testAsync();
    console.log(result);
    ```

  * 其输出的是一个Promise对象。

    ```
    c:\var\test> node --harmony_async_await .
    Promise { 'hello async' }
    ```

  * 因此，async函数（包含函数语句、函数表达式、Lambda表达式）会返回一个Promise对象，如果在函数中return一个直接量，async会把这个直接量通过Promise.resolve()封装成Promise对象。（注：Promise.resolve(x)可以看作是new Promise(resolve => resolve(x))的简写，可以用于快速封装字面量对象或其他对象，将其封装成Promise实例。）

  * async函数返回的是一个Promise对象，所以在最外层不能用await获取其返回值的情况下，需要使用原来的方式：then()链来处理这个Promise对象：

    ```javascript
    testAsync().then(v => {
        console.log(v);    // 输出 hello async
    });
    ```

  * 如果async函数没有返回值，那么它会返回Promise.resolve(undefined)。

  * 因为Promise的特点就是“无等待“，所以在没有await的情况下执行async函数，它会立即执行，返回一个Promise对象，并且，绝不会阻塞后面的语句。这和普通返回Promise对象的函数一样。


#### 1.2await在等待什么

一般来说，都认为await是在等待一个async函数完成。不过按语法说明，await等待的是一个表达式，这个表达式的计算结果是Promise对象或者其他值（没有特殊限定）。

因为 async 函数返回一个 Promise 对象，所以 await 可以用于等待一个 async 函数的返回值——这也可以说是 await 在等 async 函数，但要清楚，它等的实际是一个返回值。注意到 await 不仅仅用于等 Promise 对象，它可以等任意表达式的结果，所以，await 后面实际是可以接普通函数调用或者直接量的。所以下面这个示例完全可以正确运行。

```javascript
function getSomething() {
    return "something";
}

async function testAsync() {
    return Promise.resolve("hello async");
}

async function test() {
    const v1 = await getSomething();
    const v2 = await testAsync();
    console.log(v1, v2);
}

test();
```

#### 1.3await等到了要等的接下来做什么

await 是个运算符，用于组成表达式，await 表达式的运算结果取决于它等的东西。

如果它等到的不是一个 Promise 对象，那 await 表达式的运算结果就是它等到的东西。

如果它等到的是一个 Promise 对象，await 就忙起来了，它会阻塞后面的代码，等着 Promise 对象 resolve，然后得到 resolve 的值，作为 await 表达式的运算结果。

await为什么必须使用在async函数中？是因为async 函数调用不会造成阻塞，它内部所有的阻塞都被封装在一个 Promise 对象中异步执行。

### 2.async/await具有什么优势

#### 2.1简单的比较

async会将其后的函数（函数表达式或Lambda）的返回值封装成一个Promise对象，而await会等待这个Promise完成，并将其resolve的结果返回出来。

举例说明，用setTimeout模拟耗时的异步操作，首先不用async/await的写法：

```javascript
function takeLongTime() {
    return new Promise(resolve => {
        setTimeout(() => resolve("long_time_value"), 1000);
    });
}

takeLongTime().then(v => {
    console.log("got", v);
});
```

如果改用async/await写法如下：

```javascript
function takeLongTime() {
    return new Promise(resolve => {
        setTimeout(() => resolve("long_time_value"), 1000);
    });
}

async function test() {
    const v = await takeLongTime();
    console.log(v);
}

test();
```

可以看到takeLongTime()没有申明为async。实际上，takeLongTime()本身就是返回的Promise对象，加不加async结果都一样。

上面的两端代码，两种方式对异步调用的处理（实际就是对Promise对象的处理）差别并不明显，甚至使用async/await还需要多写一些代码，那么async/await的优势在哪里？

#### 2.2async/await的优势在于处理then链

单一的Promise链并不能体现出async/await的优势，但是，如果需要处理由多个Promise组成的then链的时候，优势就能体现出来了（可以理解为，Promise通过then链来解决多层回调的问题，现在又用async/await来进一步优化它）。

假设一个业务，分多个步骤完成，每个步骤都是异步的，而且依赖于上一个步骤的结果。我们仍用setTimeout来模拟异步操作：

```javascript
/**
 * 传入参数 n，表示这个函数执行的时间（毫秒）
 * 执行的结果是 n + 200，这个值将用于下一步骤
 */
function takeLongTime(n) {
    return new Promise(resolve => {
        setTimeout(() => resolve(n + 200), n);
    });
}

function step1(n) {
    console.log(`step1 with ${n}`);
    return takeLongTime(n);
}

function step2(n) {
    console.log(`step2 with ${n}`);
    return takeLongTime(n);
}

function step3(n) {
    console.log(`step3 with ${n}`);
    return takeLongTime(n);
}
```

现在用Promise方式来实现这三个步骤的处理：

```javascript
function doIt() {
    console.time("doIt");
    const time1 = 300;
    step1(time1)
        .then(time2 => step2(time2))
        .then(time3 => step3(time3))
        .then(result => {
            console.log(`result is ${result}`);
            console.timeEnd("doIt");
        });
}

doIt();

// c:\var\test>node --harmony_async_await .
// step1 with 300
// step2 with 500
// step3 with 700
// result is 900
// doIt: 1507.251ms
```

输出结果 result是step3()的参数700 + 200=900。doIt()顺序执行了三个步骤，一共用了300 + 500 + 700 = 1500毫秒，和console.time()/console.timeEnd()计算的结果一致。

如果使用async/await来实现，如下：

```javascript
async function doIt() {
    console.time("doIt");
    const time1 = 300;
    const time2 = await step1(time1);
    const time3 = await step2(time2);
    const result = await step3(time3);
    console.log(`result is ${result}`);
    console.timeEnd("doIt");
}

doIt();
```

结果和之前的Promise实现时一样的，但是这个代码看起来清晰很多，几乎和同步代码一样简洁。

#### 2.3async/await用于每个步骤都余姚之前步骤返回结果的时候代码更简洁

现在把业务要求修改一下，仍然是三个步骤，但是每个步骤都需要之前每个步骤的结果。

```javascript
function step1(n) {
    console.log(`step1 with ${n}`);
    return takeLongTime(n);
}

function step2(m, n) {
    console.log(`step2 with ${m} and ${n}`);
    return takeLongTime(m + n);
}

function step3(k, m, n) {
    console.log(`step3 with ${k}, ${m} and ${n}`);
    return takeLongTime(k + m + n);
}
```

使用async/await写法如下：

```javascript
async function doIt() {
    console.time("doIt");
    const time1 = 300;
    const time2 = await step1(time1);
    const time3 = await step2(time1, time2);
    const result = await step3(time1, time2, time3);
    console.log(`result is ${result}`);
    console.timeEnd("doIt");
}

doIt();

// c:\var\test>node --harmony_async_await .
// step1 with 300
// step2 with 800 = 300 + 500
// step3 with 1800 = 300 + 500 + 1000
// result is 2000
// doIt: 2907.387ms
```

使用Promise的then链方式实现如下：

```javascript
function doIt() {
    console.time("doIt");
    const time1 = 300;
    step1(time1)
        .then(time2 => {
            return step2(time1, time2)
                .then(time3 => [time1, time2, time3]);
        })
        .then(times => {
            const [time1, time2, time3] = times;
            return step3(time1, time2, time3);
        })
        .then(result => {
            console.log(`result is ${result}`);
            console.timeEnd("doIt");
        });
}

doIt();
```

对比以上两段代码可以发现，使用Promise方案，需要处理一堆参数，参数残敌太麻烦，写法复杂不简洁。

### 3.实战中的代码例子

做手机app时，点击下一页时，需要先对业务进行校验，完成之后返回结果判断是否跳转下一页或者弹出校验提示。

```javascript
//await使用在如下这种async函数中。
$scope.nextPage = async function () {
		var pass = false;
		var shix = $scope.shixiang;
		var list =[];
		var sx = {
			isShow: true,
			isValid: true,
			isCurrent: false,
			sxName: 'base'
		};
		list.push(sx);
		for (let key of Object.keys(shix)) {
			let select = shix[key];
			if(select){
				var sx = {
					isShow: true,
					isValid: true,
					isCurrent: false,
					isOver: false,
					sxName: key
				};
				if(key == 'bysbd'){
					var res = await initGraduate();
                    //在这里，如果initGraduate()返回的resolve是false就弹出提示并且不能跳转下一页
					if (!res) {
						return;
					}
				}

				list.push(sx);
				pass = true;
			}
		}

		var linkUrl =  global_config.ahrefUrl+"packSubmit/views/packSubmitApplication.html";
		var param = {
			shixiang: list,
			choose: $scope.shixiang,
			temportary: $scope.temportary
		};
		var obj = {
			pageId: "packSubmitApplication",
			url: linkUrl,
			param: param
		};
		J2C.createNewWebPage(obj);
	}
```

```javascript
function initGraduate () {
		var promise = new Promise(function (resolve,reject) {
			var data = {
				"aac002": $scope.idNumber
			};
			var headers = {
				"Authorization": $scope.token,
				"Content-Type": "application/json"
			};
			businessUtilService.getQuery(data, headers).then(function (response) {
				if (response.data.gridList !== null) {
					if (response.data.gridList[0].bbd012 === '1') {
						dialogMsg("提示不符合");
						resolve(false)
					} else {
						resolve(true)
					}
				} else {
					dialogMsg("提示不符合");
					resolve(false)
				}
			})
		})
		return promise;
	};
```



<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>