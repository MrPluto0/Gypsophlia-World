# 写在前面

模拟`JS`解释器，根据`acorn`模块生成的`AST`树来实现`eval`功能，已支持`ES5`语法，`Async`语法，`Generator`语法，且模拟实现了作用域。特作此文以分享收获与想法。

# 编译原理

> **高级语言**为我们常用的语言，如`C`/`CPP`/`JS`等，是在底层接口与特性之上封装而生，符合人们习惯，但往往效率低；而**低级语言**更接近机械码，如汇编语言，编写难度大，但效率非常高。

编辑原理是一种对高级程序语言进行翻译的一种技术，主要有如下步骤：

- 词法分析
  - 扫描文件中所有字符，构造状态机进行状态转移。
  - 字符所组成的符号表包括**标识符**/**关键字**/**操作符**/**界符**等。
- 语法分析
  - 根据词法分析的各个词法单元进行分析，常用以**语法树**来构造结构。
  - 分析过程可以通过**自上而下的 LL 文法**或**自下而上的 LR 文法**。
- 语义分析
  - 语义分析器借助用语法树和符号表中的信息来检查源程序是否和语言定义的语义一致。
  - 主要包括`类型检查`，`词法结构`等。
- 中间代码生成
  - 中间代码是处于源代码和目标代码的中间结构，容易由其生成目标代码。
  - 这是一种结构简单、含义明确的记号系统。
- 代码优化
  - 改进中间代码（或目的代码），以使得空间占用减少，运行时间缩短。
- 目的代码生成
  - 根据中间代码生成的二进制的机器指令或汇编语言。

## JIT 和 AOT

根据所使用语言的编译时机，可以将分为**编译型语言**和**解释型语言**。

**编译型语言**是程序运行前便将代码编译为计算机能够理解的语言，即汇编码或机械码。典型的如`C`，`C++`等。

**解释型语言**是程序运行时边编译边运行，因此需要用到解释器才可执行代码，且执行效率天生比编译型语言低。典型的便是`JS`。

因此引申出两个概念`AOT`(Ahead-of-Time)提前编译，和`JIT`(Just-in-Time)即时编译。

# JS 执行过程

JS 作为`解释型语言`，其执行步骤基本有如下四步（非完全的步骤）：

1. 语义分析，即进行分词

```js
[
  { type: "Keyword", value: "var" },
  { type: "Identifier", value: "a" },
  { type: "Punctuator", value: "=" },
  { type: "Number", value: "1" },
];
```

2. 语法分析，即生成`AST`树

```js
{
  "type": "VariableDeclaration",
  "declarations": [
    {
      "type": "VariableDeclarator",
      "id": {
        "type": "Identifier",
        "name": "a"
      },
      "init": {
        "type": "Literal",
        "value": 1,
      }
    }
  ],
  "kind": "var"
}
```

3. 预解析，即对某些变量与函数进行声明提前的处理等。

该步骤可在运行中处理，执行某一作用域前，先对该作用域进行预解析，即预解析在运行之中，本项目便是采用此种做法。

4. 运行，即根据`AST`树，或其它中间表示形式来执行`JS`代码。

运行和预解析是本文中的主要内容，由于分词与解析为`AST`的过程过于复杂，便不再具体实现，如对分词与解析感兴趣，可参考我以下两个项目：

- [MrPluto0/Lexical-Analyzer (github.com)](https://github.com/MrPluto0/Lexical-Analyzer) 该项目中仅实现了对`c`语言的分词功能，项目报告里记录了更多细节的内容。
- [MrPluto0/dslBot (github.com)](https://github.com/MrPluto0/dslBot) 其中定义了简单的 dsl 语言，并对该语言实现`tokenize`，`parser`，`runner`三个模块，即对应以上除了预解析外的其它模块。

# JS 解释器

本文中主要借助`acorn`模块来生成`JS`代码的语法树`AST`，我们将从此树入手来递归实现`JS`代码的实现——`JS解释器`。

## 相关工具

1. 一个在线网址可以根据代码在线生成`AST`树，以在完成过程中进行比对：[AST explorer](https://astexplorer.net/)
2. 关于`es5`语法生成`AST`树各种节点的类型说明文档：[estree/es5.md · estree/estree (github.com)](https://github.com/estree/estree/blob/master/es5.md)

## 核心：递归实现

> 注：此处的初步的封装，根据后面的步骤，该递归函数会传入多个参数，函数类型变成`generator`类型，函数调用方式也会发生变化。

实现一个类似于`eval`的功能，给它取名为`evaluate`，传入的参数即为`node`节点对象。在其中通过`switch/case`来判断节点类型，进行转换。

比如最简单的`1+2`的式子的计算，转换出来的`AST`树如下：

```js
{
  "type": "Program",
  "start": 0,
  "end": 3,
  "body": [
    {
      "type": "ExpressionStatement",
      "start": 0,
      "end": 3,
      "expression": {
        "type": "BinaryExpression",
        "start": 0,
        "end": 3,
        "left": {
          "type": "Literal",
          "start": 0,
          "end": 1,
          "value": 1,
          "raw": "1"
        },
        "operator": "+",
        "right": {
          "type": "Literal",
          "start": 2,
          "end": 3,
          "value": 2,
          "raw": "2"
        }
      }
    }
  ],
  "sourceType": "module"
}
```

那么`switch`进行状态转换的关键就是节点的`type`属性，该属于在`es5`的文档中均有详细描述：

```js
function evaluate(node) {
  let res;

  switch (node.type) {
    case "Program": {
      for (const stat of node.body) {
        res = evaluate(node.body);
      }
      return res;
    }
    case "Literal":
      return node.value;
    case "ExpressionStatement":
      return evaluate(node.expressiopn);
    case "BinaryExpression": {
      switch (node.operator) {
        case "+":
          return evaluate(node.left) + evaluate(node.right);
        case "-":
          return evaluate(node.left) - evaluate(node.right);
        // ...
      }
    }
  }
}
```

于是，只要把`AST`生成的节点传入函数中，并执行即可得到最终结果`3`，即`eval("1+2") === 3`。

## 作用域 Scope

在解释执行过程中，必定会存在着变量，那么变量该如何存储呢？考虑这个问题的话，就需要考虑变量的产生环境——`作用域`。

`作用域`用于保存一个区块的变量，常量，函数等的定义信息和赋值信息；与此经常同时提起的一个名字叫做`执行上下文`，上下文的产生是在代码执行过程中产生，其中包含变量信息与`this`信息等，二者不能等同，但是可认为`执行上下文`中包含着`作用域`。

这里我们进行简化，只考虑执行过程中的作用域，同时作用域包含着`作用域链`，若在当前域中找不到变量信息，可继续沿着链向上寻找，类比原型链很好理解。

![DFA.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/90a086988eea45969efbb2b947b48e04~tplv-k3u1fbpfcp-watermark.image?)

### 分类

`JS`中作用域主要分为三类：

- 全局作用域
  - 全局作用域中仅存在一处，即为最上级的环境。
- 函数作用域
  - 顾名思义，函数存在并执行时，内部存储函数作用域。
- 块作用域
  - 每隔 block 块`{}`都可产生作用域，如`if` `for` `while`等。

本解释器中，作用域中存储相关的变量信息，包含`declare`，`get`，`set`等部分。其中针对`var` `const` `let`三类变量进行存储，各自具备各自的声明与使用特点。

> 注：下面的 Scope 结构框架如此，实际使用还需增加其它元素，如变量的类型，如父作用域的处理，如错误处理等。具体见项目[js-interpreter/scope.js at master · MrPluto0/js-interpreter (github.com)](https://github.com/MrPluto0/js-interpreter/blob/master/src/scope.js)。

```js
class Scope {
  constructor(type, parent = null) {
    this.type = type; // Global Or Function Or Block
    this.parent = parent;
    this.variables = {};
  }
  declare(name) {
    this.variables[name] = undefined;
  }
  get(name) {
    if (this.variables[name]) return this.variables[name];
    else return this.parent.get(name);
  }
  set(name, value) {
    if (this.variables[name]) this.variables[name] = value;
    else this.parent.set(name, value);
  }
}
```

### var

- 允许重复声明
- 声明提前至**函数作用域**或**全局作用域**
- **全局作用域**下变量直接赋值，则当作`var`处理
  - 如：`name = 1`，会自动将为其进行`var`声明
- **全局作用域**下会成为全局环境变量的属性
  - `window.name`
  - `global.name`

### const

- 不允许重复声明
- 不允许重新赋值
- 不能声明提前

### let

- 不允许重复声明
- 允许重新赋值
- 不能声明提前

## 上下文 This

`This`即为**执行上下文**中较为关键的一个属性，但是在递归中如何绑定`This`？有两种做法。

第一种，可以在每次递归调用`evaluate`的时候直接传入`this`，即`evaluate.call(this, node)`，这样便不需要额外处理`this`，只需在`ThisExpression`时，返回当前的`this`即可。

第二种，可以在每次产生新的上下文的时候，进行人工判断，在代码中此处为`scope`定义一个`'this'`变量来存储该环境的`this`，只需在`ThisExpression`时，通过`scope.get('this')`，来返回当前的`this`。

## 条件语法

条件语法这里主要涉及`IfStatement`和`ConditionalExpression`，前者不多言，后者即为三元表达式。

这两种表达式生成的`AST`都有共同点：

```js
{
  "type": "IfStatement", // type: "ConditionalExpression"
  "test": {},
  "consequent": {},
  "alternate": {}
}
```

根据`test`执行的返回值是否正确，决定执行`consequent`语法，还是`alternate`语法。

## 循环 Loop

循环语法主要设置`while` `do while` `for` `for of`，以`ForStatement`为例：

```js
{
  "type": "ForStatement",
  "init": {},
  "test": {},
  "update": {}
}
```

这里通过`init`执行一步操作，多为创建一个变量，注意此时创建的变量应该存储`for`的作用域中，需要新建作用域：`const child = new Scope("Block", scope);`

然后进行循环检测，该循环可以借用 js 自带的语法`for或while`，循环体内容执行完毕后执行`update`，最后判断`test`的条件是否正确，若正确则继续，否则退出。

### `break`和`continue`

考虑这么一种情况，在遇到`break`和`continue`时，此时是一次新的递归，碰到了`BreakStatement`和`ContinueStatement`，那么直接调用`break`或`continue`是行不通的，这是便需要将其返回值进行封装，以便在原来循环的执行栈中进行操作。

> 这里你会发现，有个`label`字段，这是一种平时比较少使用的语句`LabeledStatement`，那具体如何处理呢，想一想。

```js
case "WhileStatement": {
  let res;
  const child = new Scope("Block", scope);
  while (evaluate.call(this, node.test, child)) {
    res = evaluate.call(this, node.body, child);
    // 记住下面这条语句，处理函数返回。
    if (res && res.type === "return") return res;
    // 处理break和continue
    if (res && res.type === "break") break;
    if (res && res.type === "continue") continue;
  }
  return res?.type ? res.value : res;
}
case "ContinueStatement":
  return { type: "continue", label: node.label?.name };
case "BreakStatement":
  return { type: "break", label: node.label?.name };
```

## 函数 Function

本解释器中将处理以下四类函数：

- 普通函数 `function() {}`
- 箭头函数 `() => {}`
- 异步函数 `async function() {}`
- 生成器函数 `funtion* (){}`

对于函数的处理，需创建新的函数返回，在新的函数中对原函数进行调用。其中涉及到`CallExpression`，`FunctionExpress`，`FunctionDeclaration`，`ArrowFunctionExpression`。

这里先演示普通函数的实现与调用过程（至于异步与生成器作为下面另一类内容）。

```js
function f(a, b) {}
f(1, 2);
```

对于上面的函数声明与调用，生成如下的简化的`AST`树，主要分为两部分：

```js
{
  "type": "FunctionDeclaration",
  "id": {
    "type": "Identifier",
    "name": "f"
  },
  "params": [
    {
      "type": "Identifier",
      "name": "a"
    },
    {
      "type": "Identifier",
      "name": "b"
    }
  ],
  "body": {
    "type": "BlockStatement",
    "body": []
  }
},
{
  "type": "ExpressionStatement",
  "expression": {
    "type": "CallExpression",
    "callee": {
      "type": "Identifier",
      "name": "f"
    },
    "arguments": [
      {
        "type": "Literal",
        "value": 1,
        "raw": "1"
      },
      {
        "type": "Literal",
        "value": 2,
        "raw": "2"
      }
    ],
  }
}
```

### 函数声明

对于函数声明的封装，应该自己封装一个函数，并存储到 scope 信息中。

> 注：该声明方式会默认将函数名设置为`func`，函数参数长度设置为`0`，但是`function.name`和`function.length`属于不可写属性，此处可通过`Object.defineProperty`来修改。

```js
case "FunctionExpression":
    let func = function (...args) {
      // 创建函数作用域
      const child = new Scope("Function", scope);
      // 遍历参数，将函数参数传入作用域中
      node.params.forEach((param, _i) => {
        child.declare("let", param.name);
        child.set(param.name, args[_i]);
      })
      // 执行函数体，并返回
      let res = evaluate(node.body, child);
      if (res?.type === "return") return res.value;
    };

    // 将函数本身加入当前作用域中，并返回
    scope.declare("let", name);
    scope.set(name, func);
    return func;
```

### 函数调用

当函数执行时，则同步参数后，即可执行：

> 注：此处的执行将上下文直接定为`this`，但是需要考虑函数作为对象的属性执行时的情况。此时 this 应该指向该对象。

```js
case "CallExpression": {
    let callee = evaluate(node.callee, scope)

    return callee.apply(
        this,
        node.arguments.map(
          (subNode) => evaluate.call(this, subNode, scope).next().value
        )
    );
}
```

### 函数返回

考虑这么一种情况，在递归过程中，如果遇到了`return`关键字，那直接`return`的话，返回的是该`evaluate`的值，此时需要进行处理加工，其处理方式类似于前面的`break`和`continue`，但是有所不同的时，在前提的过程中，每个语句执行结果若为`{type:"return", value:"xxx"}`都应该直接`return`，直到碰到了函数体。

可查看上面函数声明的代码中对`return`的判断。

## 生成器 Generator

生成器的特点在于**中断**和**恢复**，即中断当前执行环境，恢复上一次的执行环境。

### `yield`

`gen.next()`返回值为如下对象：

```js
{
  value: "xx",
  done: false // true
}
```

正由于生成器的特点，`yield`不能像`return`一样前提，否则会结束当前执行内容，那么就必须在出现`yield`的位置，实实在在的执行`yield`，使得程序在此处中断。因此，递归函数就封装为 `generator`函数，即为`function* evaluate()`，否则将无法执行`yield`。

接下来碰到的问题就是，若将`evaluate`封装为生成器，那么其返回值就成为一个对象，因此需要对各处调用`evaluate.call(this, node, scope)`的地方进行加工：

一个比较偷懒的办法是将其写为`evaluate.call(this, node, scope).next().value`，`next()`将会执行到下一次`yield`时，或者`return`时，由于在此之前的步骤中均为一次`return`，故可将暂时这么做，一次`next().value`获取的就是返回值了。

但是倘若此前步骤中包含`yield`的话，那么用该办法将会不能得到正确的结果，正确的办法该怎么办呢？核心代码如下：

> 注：该处理方案无法进行封装，因此涉及`yield`层叠的缘故，只能在需要`evaluate`的时候，进行如下的解析。

```js
function* sample() {
  // 表示根据 yield 输入和输出的值
  let input, output;
  let gen = evaluate.call(this, node, scope); // generator

  while (true) {
    // sample的`yield`输入值作为gen的下一个`yield`的输入值
    // gen的输出值，若已经return，则可结束sample函数，并返回
    output = gen.next(input);
    if (!output.done) {
      input = yield output.value;
    } else {
      return output.value; // break;
    }
  }
}
```

个人项目中仅针对测试用例，完成生成器的模拟，也即大部分地方仍使用上面所介绍的偷懒的方式，部分地方改用为以上的正规的处理方法。（要是所有都替换，代码量暴增，好不优雅 www）

## 异步 Async

异步函数将基于 `Generator` 来实现，处理`async`和`await`字段。

### async 和 await

**问**：一个函数如果是异步函数，那么它会如何运行？函数的内部是异步还是同步执行？

**答**：标记了`async`后，函数成为异步函数，其内部仍然是同步执行（允许异步操作）；只有在异步函数中才可以进行`await`操作，存在`await`时，函数内可以进行阻塞，直到获取`await`的返回值。

**问**：调用异步函数会产生什么返回值？

**答**：`async`和`await`实际上是一种语法糖，异步函数的调用默认会返回一个`Promise`，若异步执行该函数需要`promise.then()`获取返回值，但若通过`await`调用，则会阻塞且直接返回返回值，如下。

```js
async funtion test() {
    async funtion f() {
        return 1;
    }

    f().then((res) => {
        console.log(res); // 1
    })

    console.log(await f()); // 1
}

```

> 注：异步函数改成的生成器函数中，实际上以`yield`来替代`await`，主要是实现`await`的阻塞功能。

鉴于以上的特性，可以使用`generator`来实现，将异步函数改造为生成器函数，通过包装，使得其不断地`gen.next()`执行，直到获取`done`为`true`的情况，将对应的`value`作为异步函数的返回值返回即可。这里所谈到的包装即为`AsyncToGenerator`函数，将将生成器函数按照异步函数的逻辑执行即可。

通过`asyncToGenerator`函数来同步执行异步函数，可见项目文档[js-interpreter/libs.js at master · MrPluto0/js-interpreter (github.com)](https://github.com/MrPluto0/js-interpreter/blob/master/src/libs.js)。

### 异步函数执行

> 关于异步的判断，可以观察异步函数的语法树中的属性`async`，不妨研究一下语法树的结构。

```js
// 这里的封装函数属于关键
func = asyncToGenerator(function* (...args) {
  // 创建新的作用域
  const child = new Scope("Function", scope);
  // 参数传递
  node.params.forEach((param, _i) => {
    child.declare("let", param.name);
    child.set(param.name, args[_i]);
  });

  // 存储返回值
  let res;
  // 存储生成器对象
  let gen = evaluate.call(this, node.body, child);

  // 执行生成器的内容
  while (true) {
    res = gen.next(res);
    if (!res.done) {
      res = yield res.value;
    } else {
      res = res.value;
      return res?.type ? res.value : res;
    }
  }
});
```

# 总结

这篇文章对于 JS 解释器的完成来说，也仅仅是以点盖面，并没有把所有的表达式和语句类型的处理方法均给出，也是考虑到其中很多小的问题大家可以自己思考，比如**对象的计算属性如何处理**，**赋值操作中如何处理对象的深层属性**，**LabeledExpression 如何实现跳转**等待。

这些小问题，在个人的项目中已经得到解决，但并不完备，非常欢迎大家指点与学习，也希望大家通过自己的思考能够有自己的解决方案。

我的项目：[MrPluto0/js-interpreter (github.com)](https://github.com/MrPluto0/js-interpreter)

若本文有出错的地方，希望大家指出~

# 参考文章

- [编译原理——一个编译器的各个步骤的介绍\_没有标题-CSDN 博客](https://blog.csdn.net/zoweiccc/article/details/82556601)
- [理解 Javascript 执行过程 - 龙恩 0707 - 博客园](https://www.cnblogs.com/tugenhua0707/p/11980566.html)
- [什么是作用域和执行上下文 - 简书](https://www.jianshu.com/p/dffdbfdfd09b)
- [es5 语法树节点文档 · estree/estree (github.com)](https://github.com/estree/estree/blob/master/es5.md)
- [9k 字 | Promise/async/Generator 实现原理解析 - 掘金](https://juejin.cn/post/6844904096525189128)

# 相关项目

- [MrPluto0/Lexical-Analyzer C 词法处理器](https://github.com/MrPluto0/Lexical-Analyzer)
- [MrPluto0/dslBot DSL 语言程序](https://github.com/MrPluto0/dslBot)
- [MrPluto0/js-interpreter JS 解释器](https://github.com/MrPluto0/js-interpreter)
