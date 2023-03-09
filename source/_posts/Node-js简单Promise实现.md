---
title: Node.js简单Promise实现
date: 2019-06-07 15:17:58
tags:
---
Node.js最常见的callback形式的异步调用API是这个样子的，我们调用fs.readFile，然后在回调函数中分别调用onSuccess和onFail。

```
const fs = require('fs');

function onSuccess (data) {
  console.log(data);
}

function onFail (err) {
  console.log('Error: ', err);
}

fs.readFile('./myPromise.js', (err, data) => {
  if (err) {
    onFail(err);
  } else {
    onSuccess(data);
  }
});
```

我们也可以这么写
```
const fs = require('fs');

const p = {};

p.resolve = function (data) {
  console.log(data);
};

p.reject = function (err) {
  console.log('Error: ', err);
};

fs.readFile('./myPromise.js', (err, data) => {
  if (err) {
    p.reject(err);
  } else {
    p.resolve(data);
  }
});
```
和上面代码的区别是，我们把onSuccess和onFail封装到了一个Object中，分别叫resolve和reject，我们先不讨论这么写的好处。

然后我们发现其实设置p.resolve和p.reject其实可以在fs.readFile之后
```
const fs = require('fs');

const p = {};

fs.readFile('./myPromise.js', (err, data) => {
  if (err) {
    p.reject(err);
  } else {
    p.resolve(data);
  }
});

p.resolve = function (data) {
  console.log(data);
};

p.reject = function (err) {
  console.log('Error: ', err);
};
```

这其实是异步编程的一种特性，而不仅仅是Node.js的特性，用伪代码表示

```
声明一个对象A
开始一个异步操作
修改对象A的成员
```

这里的问题是，异步操作完成以后如何访问修改后的A的成员？在Node.js中，我们可以利用closure来实现。上面的例子中，回调函数内部可以引用到p对象。

更进一步，我们发现，把p对象封装在一个函数中返回，然后设置函数返回的对象的resolve和reject也是可以行的，原因同上。

```
const fs = require('fs');

function readFile (filePath) {
  const p = {};

  function cb (err, data) {
    if (err) {
      p.reject(err);
    } else {
      p.resolve(data);
    }
  }

  fs.readFile(filePath, cb);

  return p;
}

let p = readFile('./myPromise.js');

p.resolve = function (data) {
  console.log(data);
};

p.reject = function (err) {
  console.log('Error: ', err);
};
```

我们现在添加一个MyPromise类来替换p，方便我们扩展p这个对象
```
const fs = require('fs');

function MyPromise () {

}

MyPromise.prototype.then = function (resolve, reject) {
  this.resolve = resolve;
  this.reject = reject;
};

function readFile (filePath) {
  const p = new MyPromise();

  function cb (err, data) {
    if (err) {
      p.reject(err);
    } else {
      p.resolve(data);
    }
  }

  fs.readFile(filePath, cb);

  return p;
}

let p = readFile('./myPromise.js');

function resolve (data) {
  console.log(data);
}

function reject (err) {
  console.log('Error: ', err);
}

p.then(resolve, reject);
```

我们知道Promise被用来避免多层的callback嵌套，例如下面的代码
```
const fs = require('fs');

function MyPromise () {

}

MyPromise.prototype.then = function (resolve, reject) {
  this.resolve = resolve;
  this.reject = reject;
};

function readFile (filePath) {
  const p = new MyPromise();

  function cb (err, data) {
    if (err) {
      p.reject(err);
    } else {
      p.resolve(data);
    }
  }

  fs.readFile(filePath, cb);

  return p;
}

let p = readFile('./myPromise.js');

function resolve (data) {
  console.log(data);
  let p2 = readFile('./myPromise2.js'); // 这里嵌套了一层readFile
  function resolve2 (data) {
    console.log(data);
  }
  function reject2 (err) {
    console.log('Error: ', err);
  }
  p2.then(resolve2, reject2);
}

function reject (err) {
  console.log('Error: ', err);
}

p.then(resolve, reject);
```

p2也是一个MyPromise对象，那么也许可以把p2返回到最外层，然后在最外层调用then。这需要调整MyPromise的内部实现。思路是then方法会返回一个MyPromise对象，我们把这个对象保存在MyPromise的next属性。我们添加了MyPromise.prototype.resolve方法，这个方法调用了this._resolve，而this._resolve会返回一个MyPromise对象，在MyPromise.prototype.resolve方法中，我们调用this._resolve返回的MyPromise对象的then方法。

```
const fs = require('fs');

function MyPromise () {
  
}

MyPromise.prototype.then = function (resolve, reject) {
  this._resolve = resolve;
  this._reject = reject;
  this.next = new MyPromise();
  return this.next;
};

MyPromise.prototype.resolve = function (data) {
  let p = this._resolve(data);
  if (p) p.then(this.next._resolve, this.next._reject);
};

MyPromise.prototype.reject = function (err) {
  this._reject(err);
};

function readFile (filePath) {
  const p = new MyPromise();

  function cb (err, data) {
    if (err) {
      p.reject(err);
    } else {
      p.resolve(data);
    }
  }

  fs.readFile(filePath, {encoding: 'utf-8'}, cb);

  return p;
}

let p = readFile('./a.txt');

function resolve (data) {
  console.log('a.txt', data);
  let p2 = readFile('./b.txt');
  return p2;
}

function reject (err) {
  console.log('Error: ', err);
}

function resolve2 (data) {
  console.log('b.txt', data);
}

function reject2 (err) {
  console.log('Error: ', err);
}

p
.then(resolve, reject)
.then(resolve2, reject2);
```

和上一段代码的最大区别是，不需要在resolve函数中嵌套，p2的resolve和reject函数了。

上面的代码可以写成
```
let p = readFile('./a.txt');

p
.then(function resolve (data) {
  console.log('a.txt', data);
  let p2 = readFile('./b.txt');
  return p2;
}, function reject (err) {
  console.log('Error: ', err);
})
.then(function resolve2 (data) {
  console.log('b.txt', data);
}, function reject2 (err) {
  console.log('Error: ', err);
});
```

如果忽略reject函数，那么这段代码已经很像调用Promise对象的代码。

```
readFile('./a.txt');
.then(function resolve (data) {
  console.log('a.txt', data);
  let p2 = readFile('./b.txt');
  return p2;
})
.then(function resolve2 (data) {
  console.log('b.txt', data);
});
```

这样的代码给人一种顺序调用readFile的感觉，饶了一大圈，我们仅仅是为了把嵌套的代码变成顺序的形式。

这里仅仅实现了嵌套的异步操作看起来像顺序操作，和Promise对象功能还差很多，完整的Promise实现的规范是[Promises/A+](https://promisesaplus.com/)

Promise其实是一种设计模式，就像面向对象编程带来的一堆设计模式一样，在异步编程领域，Promise被大量使用。

如果看一下Promise规范的话，单单一个then函数的实现就有十几条规则，编程的时候总是要考虑这些规则可不是个好的体验，如果你不注意的话，它就会跳出说"Surprise!"。但愿你的客户不会被"Surprise"。
