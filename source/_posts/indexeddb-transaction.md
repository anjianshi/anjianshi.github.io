---
title: IndexedDB transaction 测试
date: 2020-11-12 17:07:26
tags: 
- JavaScript
- IndexedDB
---

Tips：理解以下测试内容需要先了解：宏任务、微任务、事件循环（Event-Loop）的概念。

# 测试目标
1. transaction 的执行方式
2. transaction 的自动 commit 机制
3. 手动 commit 和 abort 的行为
4. 事务隔离情况
5. 高频率开启 transaction 的性能情况


# 前置代码
```js
// 开启数据库连接
function getConnection() {
  return new Promise<IDBDatabase>((resolve, reject) => {
    const request = window.indexedDB.open('test-db', 2)
    request.onupgradeneeded = function() {
      this.result.createObjectStore('test', { keyPath: 'id' })
      this.result.createObjectStore('test2', { keyPath: 'id' })
    }
    request.onerror = function() { reject(this.error) }
    request.onsuccess = function() { resolve(this.result) }
  })
}

// 执行数据库操作的简化函数
function execute<T>(
  request: IDBRequest<T>,
  onSuccess: (data: T) => void,
  onError?: (this: IDBRequest<T>, ev: Event) => void
) {
  request.onsuccess = () => onSuccess(request.result)
  if (onError) request.onerror = onError
}

// 执行数据库操作并以 Promise 的形式输出结果
function executePromise<T>(request: IDBRequest<T>) {
  return new Promise<T>((resolve, reject) => {
    request.onsuccess = () => resolve(request.result)
    request.onerror = () => reject(request.error)
  })
}

async function run() {
  const db = await getConnection()
  // 这里执行要测试的内容
}
run()
```


# 1. transaction 的执行方式
```js
const objectStore = db.transaction('test', 'readwrite').objectStore('test')
console.log('direct 1')
setTimeout(() => console.log('setTimeout'), 0)
Promise.resolve(1).then(() => console.log('promise 1'))
execute(objectStore.put({ id: 1, value: 'abc' }), result => console.log('put', result))
execute(objectStore.getAll(), result => console.log('getAll', result))
Promise.resolve(2).then(() => console.log('promise 2'))
console.log('direct 2')
```

输出：
```
direct 1
direct 2
promise 1
promise 2
setTimeout
put 1
getAll [{id:1,value:"abc"}]
```

## 结论
transaction 操作无论读还是写，都是异步的，且是 **宏任务** 级别。


# 2. transaction 的自动 commit 机制
```js
const transaction = db.transaction('test', 'readwrite')
transaction.addEventListener('complete', () => console.log('transaction complete'))
const objectStore = transaction.objectStore('test')

console.log('1-1 直接执行 start')
execute(objectStore.getAll(), result => console.log('1-1 直接执行 end', result))

Promise.resolve(1).then(() => {
  console.log('1-2 微任务中执行 start')
  execute(objectStore.getAll(), result => console.log('1-2 微任务中执行 end', result))
})

setTimeout(() => {
  console.log('1-3 宏任务中执行 start')
  execute(objectStore.getAll(), result => console.log('1-3 宏任务中执行 end', result))
})

execute(objectStore.getAll(), () => {
  console.log('2-1 直接执行 -> 直接执行 start')
  execute(objectStore.getAll(), result => console.log('2-1 嵌套直接执行 end', result))
})

execute(objectStore.getAll(), () => {
  Promise.resolve(1).then(() => {
    console.log('2-2 直接执行 -> 微任务中执行 start')
    execute(objectStore.getAll(), result => console.log('2-2 直接执行 -> 微任务中执行 end', result))
  })
})

execute(objectStore.getAll(), () => {
  setTimeout(() => {
    console.log('2-3 直接执行 -> 宏任务中执行 start')
    execute(objectStore.getAll(), result => console.log('2-3 直接执行 -> 微任务中执行 end', result))
  })
})

Promise.resolve(1).then(() => {
  execute(objectStore.getAll(), () => {
    console.log('3-1 微任务中执行 -> 直接执行 start')
    execute(objectStore.getAll(), result => console.log('3-1 微任务中执行 -> 直接执行 end', result))
  })
})

Promise.resolve(1).then(() => {
  execute(objectStore.getAll(), () => {
    Promise.resolve(1).then(() => {
      console.log('3-2 微任务中执行 -> 微任务中执行 start')
      execute(objectStore.getAll(), result => console.log('3-2 微任务中执行 -> 微任务中执行 end', result))
    })
  })
})

console.log('4-1 async-await start')
const res1 = await executePromise(objectStore.getAll())
console.log('4-1 async-await end', res1)

console.log('4-2 async-await start')
const res2 = await executePromise(objectStore.getAll())
console.log('4-2 async-await end', res2)
```

输出：
```
1-1 直接执行 start
4-1 async-await start
1-2 微任务中执行 start
1-3 宏任务中执行 start
Uncaught DOMException: Failed to execute 'getAll' on 'IDBObjectStore': The transaction is not active.
1-1 直接执行 end [{…}]
2-1 直接执行 -> 直接执行 start
2-2 直接执行 -> 微任务中执行 start
4-1 async-await end [{…}]
4-2 async-await start
1-2 微任务中执行 end [{…}]
3-1 微任务中执行 -> 直接执行 start
3-2 微任务中执行 -> 微任务中执行 start
2-3 直接执行 -> 宏任务中执行 start
Uncaught DOMException: Failed to execute 'getAll' on 'IDBObjectStore': The transaction is not active.
2-1 嵌套直接执行 end [{…}]
2-2 直接执行 -> 微任务中执行 end [{…}]
4-2 async-await end [{…}]
3-1 微任务中执行 -> 直接执行 end [{…}]
3-2 微任务中执行 -> 微任务中执行 end [{…}]
transaction complete
```

## 现象描述
1. 建立 transaction 后，直接使用或放到一个微任务中使用它都没问题，但放到宏任务里使用则会抛出 `transaction is not active` 的异常。
2. transaction 的一次操作完成后，再次直接执行或在微任务里执行下一次操作也没问题。
3. 尝试在“宏任务”里使用 transaction 时，就算 transaction 仍有操作没有执行完成，也会抛出异常。
  （所以能不能正常使用 transaction 并不完全是根据 transaction 是否“已结束”来判断的，而是根据“transaction 在当前 event-loop 中是否有效 ”来判断）

## 结论
浏览器对 transaction 的检查及自动提交的机制如下：
1. 建立 transaction 的那个 event-loop，以及每一个操作完成触发回调的 event-loop，都会被标记为“transaction 可用”
2. 在拥有标记的 event-loop 里，可以调用 transaction 执行操作，反之则不允许。（会抛出异常，而不是触发 `request.onerror` 回调）
3. 当所有有标记的 event-loop 都运行完成，且每一个 event-loop 里都没有再触发新操作，浏览器就认为事务已完成，触发 commit。

用伪代码来表现这个逻辑：
```js
const db = {
  transaction() {
    return new Transaction()
  }
}

class Transaction {
  currentIsActive = false
  runningCount = 0
  finish = false
  oncomplete = () => {}

  constructor() {
    this._active()
    this.runningCount = 1
  }

  _active() {
    this.currentIsActive = true
    setTimeout(() => {
      this.currentIsActive = false
      this.runningCount -= 1
      if (!this.runningCount) {
        this.finish = true
        this.oncomplete()
      }
    })
  }

  _execute() {
    if (this.finish || !this.currentIsActive) throw new Error('transaction not active')
    this.runningCount += 1

    const request = {
      result: null as null | [],
      error: null,
      onsuccess: () => {},
      onerror: () => {},
    }

    // 两层 setTimeout 以避免和 _active() 的 timeout 同处一个 event-loop
    setTimeout(() => {
      setTimeout(() => {
        this._active()
        request.result = []
        request.onsuccess()
      })
    })

    return request
  }

  objectStore() {
    return {
      getAll: () => this._execute()
    }
  }
}
```

使用和前面完全一样的测试代码，获得输出输出：
```
1-1 直接执行 start
4-1 async-await start
1-2 微任务中执行 start
1-3 宏任务中执行 start
Uncaught Error: transaction not active
1-1 直接执行 end []
2-1 直接执行 -> 直接执行 start
2-2 直接执行 -> 微任务中执行 start
4-1 async-await end []
4-2 async-await start
1-2 微任务中执行 end []
3-1 微任务中执行 -> 直接执行 start
3-2 微任务中执行 -> 微任务中执行 start
2-3 直接执行 -> 宏任务中执行 start
Uncaught Error: transaction not active
2-1 嵌套直接执行 end []
2-2 直接执行 -> 微任务中执行 end []
4-2 async-await end []
3-1 微任务中执行 -> 直接执行 end []
3-2 微任务中执行 -> 微任务中执行 end []
transaction complete
```

## 注意事项
因为 transaction 是以“宏任务”方式运行的，而一个 transaction 在新开的宏任务中是未激活状态。
所以同时使用多个 transaction 时需要注意：
```js
const t1 = db.transaction('test', 'readwrite')
const t2 = db.transaction('test', 'readwrite')

execute(t1.objectStore('test').put({ id: 1, value: 'abc' }), () => {
  // 这里已处于一个新开的宏任务中
  t1.objectStore('test').getAll()   // 这句能正常运行，因为当前的宏任务（event-loop）是 t1 自己开启的
  t2.objectStore('test').getAll()   // 这里会报错，在 t1 开启的宏任务中，t2 是未激活状态
})

// 在 async function 中更要小心
async function asyncTest() {
  const t1 = db.transaction('test', 'readwrite')
  const t2 = db.transaction('test', 'readwrite')
  
  // 这里虽然表面上 transaction 操作被封装成了 promise，但本质上还是新开了一个宏任务。
  // 因此每次 await 后的下一句代码都是在一个新的宏任务（event-loop）里运行的。
  await executePromise(t1.objectStore('test').put({ id: 1, value: 'abc' }))
  await executePromise(t1.objectStore('test').getAll())     // 这句能正常运行，因为就是 t1 自己新开了这个宏任务
  await executePromise(t2.objectStore('test').getAll())     // 这句就会报错
}
asyncTest()
```


# 3.1 commit 的行为
```js
const transaction = db.transaction('test', 'readwrite')
execute(transaction.objectStore('test').put({ id: 1, value: 'commit-test' }), result => console.log('put result', result))
console.log('query-1 start')
execute(transaction.objectStore('test').getAll(), result => console.log('query-1 result', result))
transaction.commit()
console.log('query-2 start')
execute(transaction.objectStore('test').getAll(), result => console.log('query-2 result', result))
```

输出：
```
query-1 start
query-2 start
DOMException: Failed to execute 'objectStore' on 'IDBTransaction': The transaction has finished.
put result 1
query-1 result [{id:1,value:"commit-test"}]
```

结论：
- 调用 `commit()` 前执行的 transaction 操作都能正常完成（即使调用 `commit()` 当时尚未完成）
- 调用 `commit()` 后执行 transaction 操作会抛出异常（而不是触发 `request.onerror` 回调）


# 3.2 abort 的行为
```js
const transaction = db.transaction('test', 'readwrite')
await executePromise(transaction.objectStore('test').put({ id: 1, value: 'abort-test-1' }))
console.log('put-1 finish')

execute(transaction.objectStore('test').put({ id: 2, value: 'abort-test-2' }), () => console.log('put-2 finish'), e => console.log('put-2 error', e))
console.log('query-1 start')
execute(transaction.objectStore('test').getAll(), result => console.log('query-1 result', result), e => console.log('query-1 error', e))
transaction.abort()
console.log('query-2 start')
try {
execute(transaction.objectStore('test').getAll(), result => console.log('query-2 result', result), e => console.log('query-2 error', e))
} catch (e) { console.error(e) }

const transaction2 = db.transaction('test', 'readwrite')
execute(transaction2.objectStore('test').getAll(), result => console.log('transaction-2 query result', result))
```

输出：
```
put-1 finish
query-1 start
query-2 start
DOMException: Failed to execute 'objectStore' on 'IDBTransaction': The transaction has finished.
put-2 error Event {isTrusted: true, type: "error", target: IDBRequest, currentTarget: IDBRequest, eventPhase: 2, …}
query-1 error Event {isTrusted: true, type: "error", target: IDBRequest, currentTarget: IDBRequest, eventPhase: 2, …}
transaction-2 query result []
```

结论：
- 调用 `abort()` 时若有尚未完成的操作，这些操作会失败，触发 `request.onerror` 回调。
- 调用 `abort()` 后尝试执行 transaction 操作会抛出异常。
- `abort()` 会撤销当前 transaction 里已执行的操作。


# 4. 事务隔离
代码：
```js
setTimeout(() => {
  console.log('case-1: objectStore 范围完全一致的两个 transaction')

  const transaction = db.transaction('test', 'readwrite')
  const transaction2 = db.transaction('test', 'readwrite')

  console.log('case1: query other transaction start')
  execute(transaction2.objectStore('test').getAll(), result => console.log('case1: query other transaction result', result))

  console.log('case1: put start')
  execute(transaction.objectStore('test').put({ id: 1, value: 'case-1' }), () => console.log('case1: put finish'))
  console.log('case1: query start')
  execute(transaction.objectStore('test').getAll(), result => console.log('case1: query result', result))
})

setTimeout(() => {
  console.log('------------------------------------------------------')
  console.log('case-2: objectStore 有交集的两个 transaction')

  const transaction = db.transaction('test', 'readwrite')
  const transaction2 = db.transaction(['test', 'test2'], 'readwrite')

  console.log('case2: query other transaction start')
  // 注意这里查询的是第一个 transaction 没有包含的 test2 objectStore
  execute(transaction2.objectStore('test2').getAll(), result => console.log('case2: query other transaction result', result))

  console.log('case2: put start')
  execute(transaction.objectStore('test').put({ id: 1, value: 'case-2' }), () => console.log('case2: put finish'))
  console.log('case2: query start')
  execute(transaction.objectStore('test').getAll(), result => console.log('case2: query result', result))
}, 1000)

setTimeout(() => {
  console.log('------------------------------------------------------')
  console.log('case-3: objectStore 没有交集的两个 transaction')

  const transaction = db.transaction('test', 'readwrite')
  const transaction2 = db.transaction(['test2'], 'readwrite')

  console.log('case3: query other transaction start')
  execute(transaction2.objectStore('test2').getAll(), result => console.log('case3: query other transaction result', result))

  console.log('case3: put start')
  execute(transaction.objectStore('test').put({ id: 1, value: 'case-3' }), () => console.log('case3: put finish'))
  console.log('case3: query start')
  execute(transaction.objectStore('test').getAll(), result => console.log('case3: query result', result))
}, 2000)

setTimeout(() => {
  console.log('------------------------------------------------------')
  console.log('case-4: objectStore 一致，但前一个 transaction 是 readonly 的情况')

  const transaction = db.transaction('test', 'readonly')
  const transaction2 = db.transaction('test', 'readwrite')

  console.log('case4: query other transaction start')
  execute(transaction2.objectStore('test').getAll(), result => console.log('case4: query other transaction result', result))

  console.log('case4: query start')
  execute(transaction.objectStore('test').getAll(), result => console.log('case4: query result', result))
}, 3000)
```

输出：
```
case-1: objectStore 范围完全一致的两个 transaction
case1: query other transaction start
case1: put start
case1: query start
case1: put finish
case1: query result [{…}]
case1: query other transaction result [{…}]
------------------------------------------------------
case-2: objectStore 有交集的两个 transaction
case2: query other transaction start
case2: put start
case2: query start
case2: put finish
case2: query result [{…}]
case2: query other transaction result []
------------------------------------------------------
case-3: objectStore 没有交集的两个 transaction
case3: query other transaction start
case3: put start
case3: query start
case3: query other transaction result []
case3: put finish
case3: query result [{…}]
------------------------------------------------------
case-4: objectStore 一致，但前一个 transaction 是 readonly 的情况
case4: query other transaction start
case4: query start
case4: query result [{…}]
case4: query other transaction result [{…}]
```

现象分析：
- objectStore 有交集的两个 transaction，会等前一个 transaction 的所有读写操作结束后，才执行后一个 transaction 的操作。
  （无论后一个 transaction 实际读写的是不是前一个 transaction 读写的 objectStore，也无论前一个 transaction 是 readwrite 还是 readonly）
- objectStore 没有交集的两个 transaction 执行操作不会产生等待。

结论：
- 浏览器通过让后面的 transaction 等待前面的 transaction 来实现事务间的隔离。
  这样可避免传统 RDBMS 里事务 commit 时产生冲突的情况。
- 只要 transaction 的 objectStore 有交集，即使某个具体操作不涉及另一个 transaction 的 objectStore，也会发生等待。
  因为这个 objectStore 里的内容可能是基于另一个 objectStore 计算出来的。
  这样看也就能理解为什么建立 transaction 时必须制定 objectStore 范围了。


# 5. 高频率开启 transaction 的性能情况
代码：
```js
const cases = [[100, 1], [500, 5], [1000, 10], [5000, 20], [10000, 100]]
function testCase() {
  const [total, currentCase] = cases.shift()!
  console.log(`读写 ${total} 次，每 ${currentCase} 次插入开一个 transaction`)
  const start = now()
  let i = 0
  while (i < total) {
    const transaction = db.transaction('test', 'readwrite')
    const end = i + currentCase
    while (i < end) {
      const index = ++i
      execute(transaction.objectStore('test').put({ id: index, content: 'abc' }), () => {
        execute(transaction.objectStore('test').get(index), () => {
          if (index === total) {
            const time = now() - start
            const per = time / total
            console.log(`take ${time}ms, 平均每次读写 ${per}ms`)
            if (cases.length) testCase()
          }
        })
      })
    }
  }
}
testCase()
```

输出：
```
读写 100 次，每 1 次插入开一个 transaction
take 1874ms, 平均每次读写 18.74ms
读写 500 次，每 5 次插入开一个 transaction
take 1960ms, 平均每次读写 3.92ms
读写 1000 次，每 10 次插入开一个 transaction
take 2112ms, 平均每次读写 2.112ms
读写 5000 次，每 20 次插入开一个 transaction
take 6314ms, 平均每次读写 1.2628ms
读写 10000 次，每 100 次插入开一个 transaction
```

结论：
- 将多次读写操作合并到一个 transaction 里，可以显著提高运行效率。
- 但不要无意义地扩大一个 transaction 的 objectStore 范围，不然因为 transaction 间的等待机制，反而会影响效率。

