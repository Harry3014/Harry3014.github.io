---
title: "Promise简单实现"
excerpt: ""
toc: true
toc_sticky: true
date: 2023-04-20
last_modified_at: 2023-04-20
---

## 大致结构

```javascript
const PENDING_STATE = "pending";
const FULFILLED_STATE = "fulfilled";
const REJECTED_STATE = "rejected";

class PromisePolyfill {
  // 属性state和result
  #state = PENDING_STATE;
  #result = undefined;

  // 构造函数参数executor
  constructor(executor) {
    const resolve = this.#resolve.bind(this);
    const reject = this.#reject.bind(this);

    try {
      // executor的参数resolve和reject，注意绑定this
      executor(resolve, reject);
    } catch (error) {
      // 执行executor期间抛出错误reject
      reject(error);
    }
  }

  #resolve(value) {}

  #reject(reason) {}
}
```

## 敲定状态

调用 resolve 或 reject 反应操作结果，此时敲定 promise 的状态，状态一旦敲定就不会再变化。

```javascript
class PromisePolyfill {
  #state = PENDING_STATE;
  #result = undefined;

  constructor(executor) {
    const resolve = this.#resolve.bind(this);
    const reject = this.#reject.bind(this);

    try {
      executor(resolve, reject);
    } catch (error) {
      reject(error);
    }
  }

  #resolve(value) {
    if (this.#state !== PENDING_STATE) {
      return;
    }

    this.#state = FULFILLED_STATE;
    this.#result = value;
  }

  #reject(reason) {
    if (this.#state !== PENDING_STATE) {
      return;
    }

    this.#state = REJECTED_STATE;
    this.#result = reason;
  }
}
```

## 敲定状态后的处理

- 添加回调

  - 通过 then，catch，finally 添加状态变化后的回调，可以在同一个 promise 上多次调用上面的方法添加回调，使用队列保存回调。
  - 添加回调时，原 promise 的状态可能已经敲定，所以需要立即调用回调

- 调用回调：

  - 当原 promise 成功（fulfilled）或者失败（rejected）时，回调函数将被**异步**调用，所以这里使用 queueMicroTask 这个 API

  - 回调一旦被调用就不可能再次被调用，所以完成后清空回调队列，这样即使再次调用 resolve 或者 reject，也不会有任何影响

```javascript
class PromisePolyfill {
  #state = PENDING_STATE;
  #result = undefined;

  #fulfillHandlerQueue = [];
  #rejectHandlerQueue = [];

  constructor(executor) {
    const resolve = this.#resolve.bind(this);
    const reject = this.#reject.bind(this);

    try {
      executor(resolve, reject);
    } catch (error) {
      reject(error);
    }
  }

  #resolve(value) {
    if (this.#state !== PENDING_STATE) {
      return;
    }

    this.#state = FULFILLED_STATE;
    this.#result = value;

    this.#executeAllHandler();
  }

  #reject(reason) {
    if (this.#state !== PENDING_STATE) {
      return;
    }

    this.#state = REJECTED_STATE;
    this.#result = reason;

    this.#executeAllHandler();
  }

  #executeAllHandler() {
    queueMicrotask(() => {
      if (this.#state === FULFILLED_STATE) {
        this.#fulfillHandlerQueue.forEach((handler) => {
          handler(this.#result);
        });

        this.#clearHandlerQueue();
      }

      if (this.#state === REJECTED_STATE) {
        this.#rejectHandlerQueue.forEach((handler) => {
          handler(this.#result);
        });

        this.#clearHandlerQueue();
      }
    });
  }

  #clearHandlerQueue() {
    this.#fulfillHandlerQueue = [];
    this.#rejectHandlerQueue = [];
  }

  then(onFulfilled, onRejected) {
    this.#fulfillHandlerQueue.push(onFulfilled);
    this.#rejectHandlerQueue.push(onRejected);

    this.#executeAllHandler();
  }
}
```

## 支持链式调用

调用原型上的 then，catch，finally 方法，以及静态方法 resolve，reject，all，allSettled，any，race，都返回一个新的 promise，这可以支持链式调用。

### then

```javascript
then(onFulfilled, onRejected) {
  return new PromisePolyfill((resolve, reject) => {
    function createHandler(onSettled) {
      return (originalPromiseResult) => {
        try {
          const returnValue = onSettled(originalPromiseResult);

          // 根据回调函数的返回值做不同的处理
          if (returnValue instanceof PromisePolyfill) {
            // 如果返回promise，那么new promise状态跟随这个promise
            returnValue.then(
              (value) => {
                resolve(value);
              },
              (reason) => {
                reject(reason);
              }
            );
          } else {
            // 否则resolve
            resolve(returnValue);
          }
        } catch (error) {
          // 发生错误reject
          reject(error);
        }
      };
    }

    // 回调函数不一定是合规的函数
    if (typeof onFulfilled !== "function") {
      onFulfilled = (value) => {
        return value;
      };
    }

    this.#fulfillHandlerQueue.push(createHandler(onFulfilled));

    if (typeof onRejected !== "function") {
      onRejected = (reason) => {
        throw reason;
      };
    }

    this.#rejectHandlerQueue.push(createHandler(onRejected));

    this.#executeAllHandler();
  });
}
```

### catch

```javascript
catch(onRejected) {
  return this.then(undefined, onRejected);
}
```

### finally

```javascript
finally(onFinally) {
  function createHandler(isResolved) {
    return (originalPromiseResult) => {
      if (typeof onFinally === "function") {
        const returnValue = onFinally();
        // 如果onFinally回调返回一个拒绝的promise，那么新的promise也跟随这个promise
        if (
          returnValue instanceof PromisePolyfill &&
          returnValue.#state === REJECTED_STATE
        ) {
          return returnValue;
        }
      }

      // 否则新的promise一定跟随原promise的状态，而不是像then返回的promise根据回调函数的返回值判断
      if (isResolved) {
        return originalPromiseResult;
      }

      throw originalPromiseResult;
    };
  }

  // onFinally无论哪种状态都要执行
  return this.then(createHandler(true), createHandler(false));
}
```

### resolve

```javascript
static resolve(value) {
  // 如果参数是promise，那么直接返回
  if (
    value instanceof PromisePolyfill &&
    value.constructor === PromisePolyfill
  ) {
    return value;
  }

  return new PromisePolyfill((resolve) => {
    resolve(value);
  });
}
```

### reject

与 resolve 不同，reject 一定以参数为原因拒绝

```javascript
static reject(reason) {
  return new PromisePolyfill((_, reject) => {
    reject(reason);
  });
}
```

### all

```javascript
// 参数其实是iterable对象，这里以数组为例
// 参数中可以也包含非promise，这里也省略了判断
static all(promises) {
  return new PromisePolyfill((resolve, reject) => {
    const result = [];

    const promiseSize = promises.length;

    if (promiseSize === 0) {
      resolve(result);
    }

    for (const promise of promises) {
      promise.then(
        (value) => {
          result.push(value);
          if (result.length === promiseSize) {
            resolve(result);
          }
        },
        (reason) => {
          reject(reason);
        }
      );
    }
  });
}
```

### allSettled

```javascript
static allSettled(promises) {
  return new PromisePolyfill((resolve) => {
    const result = [];

    const promiseSize = promises.length;

    if (promiseSize === 0) {
      resolve(result);
    }

    for (const promise of promises) {
      promise
        .then(
          (value) => {
            result.push({
              status: FULFILLED_STATE,
              value,
            });
          },
          (reason) => {
            result.push({
              status: REJECTED_STATE,
              reason,
            });
          }
        )
        .finally(() => {
          if (result.length === promiseSize) {
            resolve(result);
          }
        });
    }
  });
}
```

### any

如果所有的 promise 都没有成功，那么返回一个 AggregateError 的实例。

```javascript
static any(promises) {
  return new PromisePolyfill((resolve, reject) => {
    const promiseSize = promises.length;

    const reasons = [];

    if (promiseSize === 0) {
      reject(new AggregateError(reasons, "All promises were rejected"));
    }

    for (const promise of promises) {
      promise.then(
        (value) => {
          resolve(value);
        },
        (reason) => {
          reasons.push(reason);

          if (reasons.length === promiseSize) {
            reject(new AggregateError(reasons, "All promises were rejected"));
          }
        }
      );
    }
  });
}
```

### race

```javascript
static race(promises) {
  return new PromisePolyfill((resolve, reject) => {
    for (const promise of promises) {
      promise.then(resolve, reject);
    }
  });
}
```

## 完整版本

```javascript
class MyPromise {
  #state;
  #result;

  #resolvedCallbacks;
  #rejectedCallbacks;

  constructor(executor) {
    this.#state = "pending";
    this.#result = undefined;

    this.#resolvedCallbacks = [];
    this.#rejectedCallbacks = [];

    const resolve = this.#resolve.bind(this);
    const reject = this.#reject.bind(this);

    try {
      executor(resolve, reject);
    } catch (error) {
      reject(error);
    }
  }

  #resolve(value) {
    if (this.#state !== "pending") {
      return;
    }

    if (value instanceof MyPromise) {
      value.then(
        (result) => {
          this.#resolve(result);
        },
        (error) => {
          this.#reject(error);
        }
      );
      return;
    }

    this.#state = "fulfilled";
    this.#result = value;

    this.#scheduleCallback();
  }

  #reject(error) {
    if (this.#state !== "pending") {
      return;
    }

    this.#state = "rejected";
    this.#result = error;

    this.#scheduleCallback();
  }

  #scheduleCallback() {
    queueMicrotask(() => {
      const callbacks = this.#state === "fulfilled" ? this.#resolvedCallbacks : this.#rejectedCallbacks;

      callbacks.forEach((callback) => {
        callback();
      });

      this.#resolvedCallbacks = [];
      this.#rejectedCallbacks = [];
    });
  }

  then(onFulfilled, onRejected) {
    if (typeof onFulfilled !== "function") {
      onFulfilled = (value) => value;
    }

    if (typeof onRejected !== "function") {
      onRejected = (error) => {
        throw error;
      };
    }

    const createCallback = (realCallback, resolve, reject) => {
      return () => {
        let callbackResult;

        try {
          callbackResult = realCallback(this.#result);

          if (callbackResult instanceof MyPromise) {
            callbackResult.then(resolve, reject);
          } else {
            resolve(callbackResult);
          }
        } catch (error) {
          reject(error);
        }
      };
    };

    return new MyPromise((resolve, reject) => {
      this.#resolvedCallbacks.push(createCallback(onFulfilled, resolve, reject));

      this.#rejectedCallbacks.push(createCallback(onRejected, resolve, reject));

      if (this.#state !== "pending") {
        this.#scheduleCallback();
      }
    });
  }

  catch(onRejected) {
    return this.then(undefined, onRejected);
  }

  finally(onFinally) {
    return new MyPromise((resolve, reject) => {
      this.#resolvedCallbacks.push(() => {
        onFinally();
        resolve(this.#result);
      });

      this.#rejectedCallbacks.push(() => {
        onFinally();
        reject(this.#result);
      });
    });
  }

  static resolve(value) {
    if (value instanceof MyPromise) {
      return value;
    }

    return new MyPromise((resolve, reject) => {
      resolve(value);
    });
  }

  static reject(error) {
    return new MyPromise((_, reject) => {
      reject(error);
    });
  }

  static all(promises) {
    return new MyPromise((resolve, reject) => {
      const result = [];

      if (promises.length === 0) {
        resolve(result);
        return;
      }

      for (const promise of promises) {
        promise.then(
          (value) => {
            result.push(value);
            if (result.length === promises.length) {
              resolve(result);
            }
          },
          (error) => {
            reject(error);
          }
        );
      }
    });
  }

  static allSettled(promises) {
    return new MyPromise((resolve) => {
      const result = [];

      if (promises.length === 0) {
        resolve(result);
        return;
      }

      for (const promise of promises) {
        promise.then(
          (value) => {
            result.push({ status: "fulfilled", value });
            if (result.length === promises.length) {
              resolve(result);
            }
          },
          (reason) => {
            result.push({ status: "rejected", reason });
            if (result.length === promises.length) {
              resolve(result);
            }
          }
        );
      }
    });
  }

  static any(promises) {
    return new MyPromise((resolve, reject) => {
      const errors = [];

      if (promises.length === 0) {
        reject(new AggregateError(errors));
      }

      for (const promise of promises) {
        promise.then(
          (value) => {
            resolve(value);
          },
          (error) => {
            errors.push(error);

            if (errors.length === promises.length) {
              reject(new AggregateError(errors));
            }
          }
        );
      }
    });
  }

  static race(promises) {
    return new MyPromise((resolve) => {
      for (const promise of promises) {
        promise.then((value) => {
          resolve(value);
        });
      }
    });
  }
}
```

## 实现 Promise.map

```js
Promise.map = (array, mapper, limit) => {
  const results = [];
  let activeCount = 0;
  let currentIndex = 0;

  return new Promise((resolve, reject) => {
    const processNext = () => {
      if (results.length === array.length) {
        resolve(results);
        return;
      }

      const promise = Promise.resolve(mapper(array[currentIndex]));

      activeCount++;
      currentIndex++;

      promise
        .then((value) => {
          results.push(value);
        })
        .catch((error) => {
          reject(error);
        })
        .finally(() => {
          activeCount--;
          processNext();
        });

      if (activeCount < limit) {
        processNext();
      }
    };

    processNext();
  });
};
```

## 实现 Promise.retry

```js
Promise.retry = function (fn, retryTimes, delay) {
  return new Promise((resolve, reject) => {
    fn()
      .then(resolve)
      .catch(() => {
        if (retryTimes === 0) {
          reject("all rejected");
        } else {
          return new Promise((resolve) => {
            setTimeout(resolve, delay);
          }).then(() => {
            return Promise.retry(fn, --retryTimes, delay);
          });
        }
      });
  });
};
```
