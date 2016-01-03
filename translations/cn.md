<!-- author:shi.pengyan 2015-12-23 20:13:11 -->
# 目录

- [简介](#introduction)
- [基础](#basics)
  - [抽象语法树(AST)](#asts)
  - [Babel各阶段](#stages-of-babel)
    - [解析](#parse)
      - [词法分析](#lexical-analysis)
      - [语法分析](#syntactic-analysis)
    - [转换](#transform)
    - [生成](#generate)
  - [遍历](#traversal)
    - [访问者](#visitors)
    - [路径](#paths)
      - [访问者中的路径](#paths-in-visitors)
    - [状态](#state)
    - [作用域](#scopes)
      - [绑定](#bindings)
- [API](#api)
  - [巴比伦](#babylon)
  - [babel-traverse](#babel-traverse)
  - [babel-types](#babel-types)
    - [Definitions](#definitions)
    - [Builders](#builders)
    - [Validators](#validators)
    - [Converters](#converters)
  - [babel-generator](#babel-generator)
  - [babel-template](#babel-template)
- [编写你的第一个Babel插件](#writing-your-first-babel-plugin)
- [Transformation Operations](#transformation-operations)
  - [Visiting](#visiting)
    - [Check if a node is a certain type](#check-if-a-node-is-a-certain-type)
    - [Check if an identifier is referenced](#check-if-an-identifier-is-referenced)
  - [Manipulation](#manipulation)
    - [Replacing a node](#replacing-a-node)
    - [Replacing a node with multiple nodes](#replacing-a-node-with-multiple-nodes)
    - [Replacing a node with a source string](#replacing-a-node-with-a-source-string)
    - [Inserting a sibling node](#inserting-a-sibling-node)
    - [Removing a node](#removing-a-node)
    - [Replacing a parent](#replacing-a-parent)
    - [Removing a parent](#removing-a-parent)
  - [Scope](#scope)
    - [Checking if a local variable is bound](#checking-if-a-local-variable-is-bound)
    - [Generating a UID](#generating-a-uid)
    - [Pushing a variable declaration to a parent scope](#pushing-a-variable-declaration-to-a-parent-scope)
    - [Rename a binding and its references](#rename-a-binding-and-its-references)
- [Building Nodes](#building-nodes)
- [Best Practices](#best-practices)
  - [Avoid traversing the AST as much as possible](#avoid-traversing-the-ast-as-much-as-possible)
    - [Merge visitors whenever possible](#merge-visitors-whenever-possible)
    - [Do not traverse when manual lookup will do](#do-not-traverse-when-manual-lookup-will-do)
  - [Optimizing nested visitors](#optimizing-nested-visitors)
  - [Being aware of nested structures](#being-aware-of-nested-structures)

# 简介

`Babel`是一款`JavaScript`通用多功能编译器。此外它还是用于多种不同形式的静态分析的模块集合。

> 静态分析是分析代码不执行代码的过程。（执行时期的代码分析被称为动态分析） 静态代码分析会产生巨大变化。它可用于代码质量检查、编译、代码高亮、代码转换、优化，压缩等等


你可以用`Babel`来构建很多不同的工具，来提高效率，编写更好的程序。

# 基础

Babel是一个JavaScript编译器，尤其是一个源码到源码的编译器，通常称为`转换器`。这意味着你用Babel构建一些JavaScript代码，Babel修改这些代码，然后生成新的代码。

## 抽象语法树(AST)

这里的每一步都涉及到[抽象语法树](https://en.wikipedia.org/wiki/Abstract_syntax_tree)（或AST）的创建和协作。

> Babel使用基于[ESTree](https://github.com/estree/estree)改写的抽象语法树,核心规范在[这里](https://github.com/babel/babel/blob/master/doc/ast/spec.md).

```js
function square(n) {
  return n * n;
}
```

> 打开[AST资源管理器](http://astexplorer.net/) ，可以更好的理解AST节点。 [这里](http://astexplorer.net/#/Z1exs6BWMq) 是上面的一代码示例。

这段程序就像下面这个列表一样：:

```md
- FunctionDeclaration:
  - id:
    - Identifier:
      - name: square
  - params [1]
    - Identifier
      - name: n
  - body:
    - BlockStatement
      - body [1]
        - ReturnStatement
          - argument
            - BinaryExpression
              - operator: *
              - left
                - Identifier
                  - name: n
              - right
                - Identifier
                  - name: n
```

或者像这样的JavaScript对象:

```js
{
  type: "FunctionDeclaration",
  id: {
    type: "Identifier",
    name: "square"
  },
  params: [{
    type: "Identifier",
    name: "n"
  }],
  body: {
    type: "BlockStatement",
    body: [{
      type: "ReturnStatement",
      argument: {
        type: "BinaryExpression",
        operator: "*",
        left: {
          type: "Identifier",
          name: "n"
        },
        right: {
          type: "Identifier",
          name: "n"
        }
      }
    }]
  }
}
```

你可能注意到了AST每层有如下相似结构:

```js
{
  type: "FunctionDeclaration",
  id: {...},
  params: [...],
  body: {...}
}
```

```js
{
  type: "Identifier",
  name: ...
}
```

```js
{
  type: "BinaryExpression",
  operator: ...,
  left: {...},
  right: {...}
}
```

> 注意: 为了方便删除了一些属性。

这里每一项都叫做**节点**. 一个AST可以由一个或百十个甚至上千个节点组成。组织在一起就可以用于静态分析程序的语法。

每个节点都有这样的接口:

```typescript
interface Node {
  type: string;
}
```

`type`字段代表了节点类型的字符串（例如
`"FunctionDeclaration"`, `"Identifier"`, 或者 `"BinaryExpression"`）。节点的每个类型都定义了一系列用于描述特殊节点类型的属性。

Babel在每个节点上都有额外的属性，用来描述节点在原始代码中的位置。

```js
{
  type: ...,
  start: 0,
  end: 38,
  loc: {
    start: {
      line: 1,
      column: 0
    },
    end: {
      line: 3,
      column: 1
    }
  },
  ...
}
```

`start`, `end`, `loc`这些属性出现在每个单节点中。 

## Babel各阶段

Babel三大阶段： **解析**, **转换**, **生成**。

### 解析

**解析** 阶段, 输入代码输出AST。Babel解析包括两部分：
[**词法分析**](https://en.wikipedia.org/wiki/Lexical_analysis) 和
[**语法**](https://en.wikipedia.org/wiki/Parsing).

#### 词法分析

词法分析输入代码串，将其转换成**符号**流。

你可以把符号想象成语法片段的扁平化数组

```js
n * n;
```

```js
[
  { type: { ... }, value: "n", start: 0, end: 1, loc: { ... } },
  { type: { ... }, value: "*", start: 2, end: 3, loc: { ... } },
  { type: { ... }, value: "n", start: 4, end: 5, loc: { ... } },
  ...
]
```

这里的每个`type`有一系列描述符号的属性

```js
{
  type: {
    label: 'name',
    keyword: undefined,
    beforeExpr: false,
    startsExpr: true,
    rightAssociative: false,
    isLoop: false,
    isAssign: false,
    prefix: false,
    postfix: false,
    binop: null,
    updateContext: null
  },
  ...
}
```

类似AST节点他们也有`start`, `end`, 和 `loc`字段。

#### 语法分析

语法分析输入符号流，将它转换成AST展现形式。使用符号中的信息，这一阶段将会用一种更易于处理的方式重新格式化为可展示代码结构的AST。

### 转换
[转换](https://en.wikipedia.org/wiki/Program_transformation)阶段，输入一个AST，遍历它，随着逐渐深入，更新，移除节点。到目前为止，这是Babel或是任何编译器最复杂的部分。这就是插件运行的所在地，因此它将是本手册绝大部分的主题。所以这里我们不会立刻深入。

### 生成

[代码生成](https://en.wikipedia.org/wiki/Code_generation_(compiler))阶段，输入最终的AST，转换代码串，同时创建[代码映射](http://www.html5rocks.com/en/tutorials/developertools/sourcemaps/).

代码生成是相当简单的：你可以遍历AST第一层，构建一个转换过的代码字符串

## 遍历

当你想转换一个AST，你就必须递归[遍历树](https://en.wikipedia.org/wiki/Tree_traversal) 

假如我们有一个`FunctionDeclaration`类型节点。它有一些属性`id`,`params`,`body`。他们中每个嵌套子节点

```js
{
  type: "FunctionDeclaration",
  id: {
    type: "Identifier",
    name: "square"
  },
  params: [{
    type: "Identifier",
    name: "n"
  }],
  body: {
    type: "BlockStatement",
    body: [{
      type: "ReturnStatement",
      argument: {
        type: "BinaryExpression",
        operator: "*",
        left: {
          type: "Identifier",
          name: "n"
        },
        right: {
          type: "Identifier",
          name: "n"
        }
      }
    }]
  }
}
```

因此，从 `FunctionDeclaration`开始，我们了解它内部的属性，因此我们依次访问它们及子节点每项。

下一步，到`id`，它是一个`Identifier`. `Identifier`没有任何子节点，我们继续。

接下来是`params`，它是一个节点数组，所以我们要访问它们每项。在这个示例中，它是同样是个`Identifier`的单个节点。我们继续

然后，到`body`，它是`BloackStatement`类型，它的`body`属性有多个子节点的，因此我们遍历它们。

这里仅有的一项`ReturnStatement`节点，他有一个 `argument`属性, 我们到`argument`，然后找到`BinaryExpression`.

`BinaryExpression`有`operator`, `left`, `right`. `operator`不是一个节点，仅仅是一个值，所以不需要深入，相反，继续看`left`和`right`.

遍历的过程发生在Babel转换阶段。

### 访问者

当我们说继续深入一个节点，实际上是**访问**它们。我们用这个术语的原因，就是因为有一个[访问者](https://en.wikipedia.org/wiki/Visitor_pattern)概念

访问者是在AST遍历中使用一个跨语言的模式。简单来说，把它们当做一个可接收树中特定节点类型的方法的对象。这有点抽象，让我们来看个示例。

```js
const MyVisitor = {
  Identifier() {
    console.log("Called!");
  }
};
```

> **注意:** `Identifier() { ... }` 是 `Identifier: { enter() { ... } }`的简写。

这是一个基础的访问者，当在遍历期间使用它时，将会为树中每个`Identifier` 调用`Identifier()`。

因此用这个 `Identifier()`方法的代码，在每个`Identifier` (including `square`)中调用四次。

```js
function square(n) {
  return n * n;
}
```
```js
Called!
Called!
Called!
Called!
```

这些调用都是发生在所有节点**进入**时期。然而，同样可以在**退出**时调用访问方法。

假想我们有这样树形结构

```js
- FunctionDeclaration
  - Identifier (id)
  - Identifier (params[0])
  - BlockStatement (body)
    - ReturnStatement (body)
      - BinaryExpression (argument)
        - Identifier (left)
        - Identifier (right)
```

当我们向下遍历树中的每个分支，最终到了头，就需要回溯到下一个节点。然后继续遍历树。我们**进入**每个节点，然后回溯，**退出**每个节点。

让我们来 _走_ 个过场，看看上面的树流程看起来像上面。

- 进入 `FunctionDeclaration`
  - 进入 `Identifier (id)`
    - 结束
  - 退出 `Identifier (id)`
  - 进入 `Identifier (params[0])`
    - 结束
  - 退出 `Identifier (params[0])`
  - 进入 `BlockStatement (body)`
    - 进入 `ReturnStatement (body)`
      - 进入 `BinaryExpression (argument)`
        - 进入 `Identifier (left)`
          - 结束
        - 退出 `Identifier (left)`
        - 进入 `Identifier (right)`
          - 结束
        - 退出 `Identifier (right)`
      - 退出 `BinaryExpression (argument)`
    - 退出 `ReturnStatement (body)`
  - 退出 `BlockStatement (body)`
- 退出 `FunctionDeclaration`

因此，当创建一个访问者时，你有两种方式访问一个节点。

```js
const MyVisitor = {
  Identifier: {
    enter() {
      console.log("Entered!");
    },
    exit() {
      console.log("Exited!");
    }
  }
};
```

### 路径

通常一个AST有很多节点，但是节点如何和其他节点进行关联的？我们可以用巨大的可变对象来维护和访问，我们简称**路径**


**Path**是两个节点链接的对象展示。

例如，假设我们有如下节点和子节点：

```js
{
  type: "FunctionDeclaration",
  id: {
    type: "Identifier",
    name: "square"
  },
  ...
}
```

将子节点`Identifier`作为一个路径，它看上去就像这样：

```js
{
  "parent": {
    "type": "FunctionDeclaration",
    "id": {...},
    ....
  },
  "node": {
    "type": "Identifier",
    "name": "square"
  }
}
```

同样，它包含额外的关于路径的元数据

```js
{
  "parent": {...},
  "node": {...},
  "hub": {...},
  "contexts": [],
  "data": {},
  "shouldSkip": false,
  "shouldStop": false,
  "removed": false,
  "state": null,
  "opts": null,
  "skipKeys": null,
  "parentPath": null,
  "context": null,
  "container": null,
  "listKey": null,
  "inList": false,
  "parentKey": null,
  "key": null,
  "scope": null,
  "type": null,
  "typeAnnotation": null
}
```

同样，大量的方法涉及到节点添加，更新，移动和删除，稍后我们来看看它们。

某种意义上说，路径是树中节点位置的**间接**展示以及节点的所有信息。无论何时调用修改树的方法，它们的信息就会更新。Babel为了你更方便的与Node使用，尽可能的无状态地管理着所有这些。

#### 访问者中的路径

当你有一个含`Identifier()`的访问者时，实际上呢访问的是路径而不是节点。绝大部分情况下你是与节点的表现形式进行打交道，而不是节点本身。

```js
const MyVisitor = {
  Identifier(path) {
    console.log("Visiting: " + path.node.name);
  }
};
```

```js
a + b + c;
```

```js
Visiting: a
Visiting: b
Visiting: c
```

### 状态

状态是AST转换的**敌人**。状态会一而再再而三地困扰你，你对状态的假设绝大部分总是被证明是错的，因为有些语法你未曾考虑到。

考虑如下代码:

```js
function square(n) {
  return n * n;
}
```

让我们写一个简单的访问者将`n`改成`x`:

```js
let paramName;

const MyVisitor = {
  FunctionDeclaration(path) {
    const param = path.node.params[0];
    paramName = param.name;
    param.name = "x";
  },

  Identifier(path) {
    if (path.node.name === paramName) {
      path.node.name = "x";
    }
  }
};
```

上面的代码可以运行，但是我们通过下面代码很容易污染它：

```js
function square(n) {
  return n * n;
}
n;
```

更好的处理方式就是递归处理。因此让我们像克里斯托弗·诺兰的电影里一样，访问者中增加一个访问者。

```js
const updateParamNameVisitor = {
  Identifier(path) {
    if (path.node.name === this.paramName) {
      path.node.name = "x";
    }
  }
};

const MyVisitor = {
  FunctionDeclaration(path) {
    const param = path.node.params[0];
    const paramName = param.name;
    param.name = "x";

    path.traverse(updateParamNameVisitor, { paramName });
  }
};
```

当然，这是一个人为的示例，但演示了在访问者中如何消除全局状态。

### 作用域

下面让我们介绍一下[**作用域**](https://en.wikipedia.org/wiki/Scope_(computer_science))概念。JavaScript有 [词法作用域](https://en.wikipedia.org/wiki/Scope_(computer_science)#Lexical_scoping_vs._dynamic_scoping)，它是一个树形结构，树中创建新作用域。

```js
// global scope

function scopeOne() {
  // scope 1

  function scopeTwo() {
    // scope 2
  }
}
```

JavaScript中无论何时通过变量、函数、类、参数、import、label等等创建一个引用，它都属于当前作用域。

```js
var global = "I am in the global scope";

function scopeOne() {
  var one = "I am in the scope created by `scopeOne()`";

  function scopeTwo() {
    var two = "I am in the scope created by `scopeTwo()`";
  }
}
```

在一个很深的作用域中的代码可以使用一个更远的作用域的引用。 

```js
function scopeOne() {
  var one = "I am in the scope created by `scopeOne()`";

  function scopeTwo() {
    one = "I am updating the reference in `scopeOne` inside `scopeTwo`";
  }
}
```

一个低层次作用域同样可以创建一个同名而不修改原值的引用。

```js
function scopeOne() {
  var one = "I am in the scope created by `scopeOne()`";

  function scopeTwo() {
    var one = "I am creating a new `one` but leaving reference in `scopeOne()` alone.";
  }
}
```

当编写一个转换时，我们需要关注作用域。当修改不同部分时，我们需要确保没有污染已存在的代码。 

我们可能需要增加新的引用，确保他们和已存在的没有冲突。或者希望找到这变量是在哪里被引用的。我们想能够在指定的作用域中跟踪这些引用。 

一个作用域可以展示如下：

```js
{
  path: path,
  block: path.node,
  parentBlock: path.parent,
  parent: parentScope,
  bindings: [...]
}
```

通过指定一个路径和父作用域，你就可以创建一个新的作用域。然后，在遍历的过程中，搜集作用域内的所有引用（绑定）

一旦完成，你就可以使用作用域上所有的方法。我们稍后来讨论它。

#### 绑定

所有的引用都属于一个特定的作用域。这种关系称为**绑定**.

```js
function scopeOnce() {
  var ref = "This is a binding";

  ref; // This is a reference to a binding

  function scopeTwo() {
    ref; // This is a reference to a binding from a lower scope
  }
}
```

单个绑定就像这样:

```js
{
  identifier: node,
  scope: scope,
  path: path,
  kind: 'var',

  referenced: true,
  references: 3,
  referencePaths: [path, path, path],

  constant: false,
  constantViolations: [path]
}
```

通过这些信息，你可以找到绑定的所有引用，可以看出它是什么类型的绑定（参数，声明等等），搜索出属于什么样的作用域，或者获得一份拷贝。你甚至可以知道它是否是常量，找到哪条路径导致它变化的。

能够了解到一个绑定是否不变，对很多场景很有用处，最大用处就是压缩。

```js
function scopeOne() {
  var ref1 = "This is a constant binding";

  becauseNothingEverChangesTheValueOf(ref1);

  function scopeTwo() {
    var ref2 = "This is *not* a constant binding";
    ref2 = "Because this changes the value";
  }
}
```

----

# API

Babel实际上是模块的集合。在本节，我们看下主要模块，解释下他们做什么以及如何使用他们。

> 注意: 这不是详细API文档的替代者，详细API文档短期内仍是可用的。

## [`巴比伦`](https://github.com/babel/babel/tree/master/packages/babylon)

Babylon是Babel的解析器。从Acorn中fork而来，它很快，易于使用，基于插件的架构便于非标准特性（同样适用于标准属性）。

首先，先安装它。

```sh
$ npm install --save babylon
```

Let's start by simply parsing a string of code:

```js
import * as babylon from "babylon";

const code = `function square(n) {
  return n * n;
}`;

babylon.parse(code);
// Node {
//   type: "File",
//   start: 0,
//   end: 38,
//   loc: SourceLocation {...},
//   program: Node {...},
//   comments: [],
//   tokens: [...]
// }
```

我们也可以像这样传递参数给 `parse()` :

```js
babylon.parse(code, {
  sourceType: "module", // default: "script"
  plugins: ["jsx"] // default: []
});
```

`sourceType` 可以是 `"module"` 或者 `"script"` ，代表了巴比伦解析的模式。
`"module"` 会在严格模式下进行解析，允许模块申明，而`"script"` 不会.

> **注意:** `sourceType` 默认值是`"script"` ，当它发现有`import` 或者 `export`就会报错。传`sourceType: "module"`就消除错误了.

由于巴比伦是基于插件架构构建的，因此有一个`plugins`选项来启用内部插件。注意巴比伦还未给外部插件开放API，在不久将来就会这样做的。

为了查看所有插件，参考
[巴比伦说明](https://github.com/babel/babel/blob/master/packages/babylon/README.md#plugins).

## [`babel-traverse`](https://github.com/babel/babel/tree/master/packages/babel-traverse)

Babel遍历模块维护着整个树的状态，它的职责是替换、移除、添加节点。

运行以下命令来安装:

```sh
$ npm install --save babel-traverse
```

我们可以单独使用巴比伦来遍历和更新节点：

```js
import * as babylon from "babylon";
import traverse from "babel-traverse";

const code = `function square(n) {
  return n * n;
}`;

const ast = babylon.parse(code);

traverse(ast, {
  enter(path) {
    if (
      path.node.type === "Identifier" &&
      path.node.name === "n"
    ) {
      path.node.name = "x";
    }
  }
});
```

## [`babel-types`](https://github.com/babel/babel/tree/master/packages/babel-types)

Babel类型是为AST节点提供的一种类Loadsh工具类库。它包含构建、校验、转换AST节点的方法。对于清除AST逻辑以及生成工具方法十分有用。

运行以下命令进行安装：

```sh
$ npm install --save babel-types
```

然后开始使用它:

```js
import traverse from "babel-traverse";
import * as t from "babel-types";

//...

traverse(ast, {
  enter(path) {
    if (t.isIdentifier(path.node, { name: "n" })) {
      path.node.name = "x";
    }
  }
});
```

### 定义

Babel类型为每个节点类型都做了定义，这些信息中包含什么属性属于哪里，什么值是合法的，如何构建这些节点，应该如何遍历节点，以及节点的别名。

一个单一节点类型定义就像这样：

```js
defineType("BinaryExpression", {
  builder: ["operator", "left", "right"],
  fields: {
    operator: {
      validate: assertValueType("string")
    },
    left: {
      validate: assertNodeType("Expression")
    },
    right: {
      validate: assertNodeType("Expression")
    }
  },
  visitor: ["left", "right"],
  aliases: ["Binary", "Expression"]
});
```

### 建造器

你可能已经注意到了上面的定义中 `BinaryExpression` 有一个叫`builder`的字段。

```js
builder: ["operator", "left", "right"]
```

这是因为每个节点类型都有builder方法，它用起来就像这样： 

```js
t.binaryExpression("*", t.identifier("a"), t.identifier("b"));
```

它创建的AST就像这样:

```js
{
  type: "BinaryExpression",
  operator: "*",
  left: {
    type: "Identifier",
    name: "a"
  },
  right: {
    type: "Identifier",
    name: "b"
  }
}
```

输出的就像这样：

```js
a * b
```

构建器同时会校验它们创建的节点，如果错误的使用就会抛出描述性错误。这样会使其陷于方法的下一个类型。

### 校验器

`BinaryExpression`的定义同样包含节点的`fields`上的信息，以及如何校验它们。

```js
fields: {
  operator: {
    validate: assertValueType("string")
  },
  left: {
    validate: assertNodeType("Expression")
  },
  right: {
    validate: assertNodeType("Expression")
  }
}
```

上面是用于创建两种类型的校验器方法。第一个是判断是否是某类型`isX`.

```js
t.isBinaryExpression(maybeBinaryExpressionNode);
```

这个测试用于确保节点时一个二进制表达式，但是你也可以传递第二个参数，来确保节点包含指定的属性和值。

```js
t.isBinaryExpression(maybeBinaryExpressionNode, { operator: "*" });
```

同样，这些方法有更详细的断言，它们会抛出错误，而不是返回`true`或`false`。

```js
t.assertBinaryExpression(maybeBinaryExpressionNode);
t.assertBinaryExpression(maybeBinaryExpressionNode, { operator: "*" });
// Error: Expected type "BinaryExpression" with option { "operator": "*" }
```

### 转换器
<!-- work in progress -->
> [WIP]

## [`babel-generator`](https://github.com/babel/babel/tree/master/packages/babel-generator)

Babel生成器是为Babel而生的代码生成器。它输入AST，将其转换成还有源码映射的代码。

运行如下代码来安装：

```sh
$ npm install --save babel-generator
```

然后来使用它：

```js
import * as babylon from "babylon";
import generate from "babel-generator";

const code = `function square(n) {
  return n * n;
}`;

const ast = babylon.parse(code);

generate(ast, null, code);
// {
//   code: "...",
//   map: "..."
// }
```

你同样可以给 `generate()`传递参数

```js
generate(ast, {
  retainLines: false,
  compact: "auto",
  concise: false,
  quotes: "double",
  // ...
}, code);
```

## [`babel-template`](https://github.com/babel/babel/tree/master/packages/babel-template)

Babel Template 是另外一个微小又十分有用的模块。它允许你编写带有占位符的代码串，你可以取代人工构建大量的AST。

```sh
$ npm install --save babel-template
```

```js
import * as babylon from 'babylon';
import * as t from 'babel-types';
import template from 'babel-template';
import generate from 'babel-generator';


const buildRequire = template(`
    var IMPORT_NAME = require(SOURCE);
`);

const ast = buildRequire({
    IMPORT_NAME: t.keyIdentifier('myModule'),
    SOURCE: t.stringLiteral('my-module')
});
```

```js
var myModule = require("my-module");
```

# 编写你的第一个Babel插件

现在你熟悉了所有的Babel基础，让我们试着把这些插件API整合在一起。

从 `function` 开始，传递当前的`babel`对象

```js
export default function(babel) {
  // plugin contents
}
```

由于以后你会经常和它打交道，你可能需要使用像`babel.types`之类的 

```js
export default function({ types: t }) {
  // plugin contents
}
```

然后，你需要返回一个带有`visitor`属性的对象，它是插件的访问者

```js
export default function({ types: t }) {
  return {
    visitor: {
      // visitor contents
    }
  };
};
```

让我们写一个简单的插件来看看它是如何运行的。这是我们的源码。

```js
foo === bar;
```

或者AST形式:

```js
{
  type: "BinaryExpression",
  operator: "===",
  left: {
    type: "Identifier",
    name: "foo"
  },
  right: {
    type: "Identifier",
    name: "bar"
  }
}
```

我们通过添加`BinaryExpression`访问者方法开始

```js
export default function({ types: t }) {
  return {
    visitor: {
      BinaryExpression(path) {
        // ...
      }
    }
  };
}
```

然后，让我们缩小范围到只有使用`===`操作符的`BinaryExpression`，

```js
visitor: {
  BinaryExpression(path) {
    if (path.node.operator !== "===") {
      return;
    }

    // ...
  }
}
```

现在，让我们用一个新的运算符替换`left`属性

```js
BinaryExpression(path) {
  if (path.node.operator !== "===") {
    return;
  }

  path.node.left = t.identifier("sebmck");
  // ...
}
```

好了，如果我们运行这个插件，将会得到：

```js
sebmck === bar;
```

现在，让我们替换`right`属性


```js
BinaryExpression(path) {
  if (path.node.operator !== "===") {
    return;
  }

  path.node.left = t.identifier("sebmck");
  path.node.right = t.identifier("dork");
}
```

最终的结果就像这样：

```js
sebmck === dork;
```

太棒了! 我们第一个Babel插件好了.

----

# 转换操作

## 访问

### 检查节点是否是指定的类型

如果你想检查一个节点是什么类型，首选的方式就是这样：

```js
BinaryExpression(path) {
  if (t.isIdentifier(path.node.left)) {
    // ...
  }
}
```

你也可以在节点上做属性的浅检查：

```js
BinaryExpression(path) {
  if (t.isIdentifier(path.node.left, { name: "n" })) {
    // ...
  }
}
```

功能上等于:

```js
BinaryExpression(path) {
  if (
    path.node.left != null &&
    path.node.left.type === "Identifier" &&
    path.node.left.name === "n"
  ) {
    // ...
  }
}
```

### 检查一个标识符是否被引用

```js
Identifier(path) {
  if (path.isReferencedIdentifier()) {
    // ...
  }
}
```

或者:

```js
Identifier(path) {
  if (t.isReferenced(path.node, path.parent)) {
    // ...
  }
}
```

## 操作

### 替换节点

```js
BinaryExpression(path) {
  path.replaceWith(
    t.binaryExpression("**", path.node.left, t.numberLiteral(2))
  );
}
```

```diff
  function square(n) {
-   return n * n;
+   return n ** 2;
  }
```

### 多个节点替换一个节点

```js
ReturnStatement(path) {
  path.replaceWithMultiple([
    t.expressionStatement(t.stringLiteral("Is this the real life?")),
    t.expressionStatement(t.stringLiteral("Is this just fantasy?")),
    t.expressionStatement(t.stringLiteral("(Enjoy singing the rest of the song in your head)")),
  ]);
}
```

```diff
  function square(n) {
-   return n * n;
+   "Is this the real life?";
+   "Is this just fantasy?";
+   "(Enjoy singing the rest of the song in your head)";
  }
```

> **注意:** 当用多个节点替换一个表达式时，它们必须是语句。这是因为在替换节点时Babel使用广泛地使用了试探法，这意味你可以做一些疯狂的转换，否则会极其冗长。
>When replacing an expression with multiple nodes, they must be
> statements. This is because Babel uses heuristics extensively when replacing
> nodes which means that you can do some pretty crazy transformations that would
> be extremely verbose otherwise.

### 用字符串替换节点

```js
FunctionDeclaration(path) {
  path.replaceWithSourceString(`function add(a, b) {
    return a + b;
  }`);
}
```

```diff
- function square(n) {
-   return n * n;
+ function add(a, b) {
+   return a + b;
  }
```

> **注意:** 不推荐使用这个API，除非你处理动态源字符串，另外，在解析访问者之外的代码效率更好。
>
### 插入兄弟节点

```js
FunctionDeclaration(path) {
  path.insertBefore(t.expressionStatement(t.stringLiteral("Because I'm easy come, easy go.")));
  path.insertAfter(t.expressionStatement(t.stringLiteral("A little high, little low.")));
}
```

```diff
+ "Because I'm easy come, easy go.";
  function square(n) {
    return n * n;
  }
+ "A little high, little low.";
```

> **注意:** 这里应该是语句或一组语句，使用同样的试探法[多个节点替换一个节点](#replacing-a-node-with-multiple-nodes).

### 移除节点

```js
FunctionDeclaration(path) {
  path.remove();
}
```

```diff
- function square(n) {
-   return n * n;
- }
```

### 替换父节点

```js
BinaryExpression(path) {
  path.parentPath.replaceWith(
    t.expressionStatement(t.stringLiteral("Anyway the wind blows, doesn't really matter to me, to me."))
  );
}
```

```diff
  function square(n) {
-   return n * n;
+   "Anyway the wind blows, doesn't really matter to me, to me.";
  }
```

### 移除父节点

```js
BinaryExpression(path) {
  path.parentPath.remove();
}
```

```diff
  function square(n) {
-   return n * n;
  }
```

## 作用域

### 检查本地变量是否可用

```js
FunctionDeclaration(path) {
  if (path.scope.hasBinding("n")) {
    // ...
  }
}
```

这会遍历作用域树，检查指定的绑定。

你也可以检查一个作用域是否有自己的 **拥有（own）** 绑定 

```js
FunctionDeclaration(path) {
  if (path.scope.hasOwnBinding("n")) {
    // ...
  }
}
```

### 生成UID

生成一个不和本地已定义的变量冲突的标识符

```js
FunctionDeclaration(path) {
  path.scope.generateUidIdentifier("uid");
  // Node { type: "Identifier", name: "_uid" }
  path.scope.generateUidIdentifier("uid");
  // Node { type: "Identifier", name: "_uid2" }
}
```

### 推送一个变量申明到父作用域

有事你可能增加一个变量声明`VariableDeclaration`，这样可以给它赋值。

```js
FunctionDeclaration(path) {
  const id = path.scope.generateUidIdentifierBasedOnNode(path.node.id);
  path.remove();
  path.scope.parent.push({ id, init: path.node });
}
```

```diff
- function square(n) {
+ var _square = function square(n) {
    return n * n;
- }
+ };
```

### 重命名绑定及其引用

```js
FunctionDeclaration(path) {
  path.scope.rename("n", "x");
}
```

```diff
- function square(n) {
-   return n * n;
+ function square(x) {
+   return x * x;
  }
```

或者，你可以重命名一个绑定到生成的唯一标识：

```js
FunctionDeclaration(path) {
  path.scope.rename("n");
}
```

```diff
- function square(n) {
-   return n * n;
+ function square(_n) {
+   return _n * _n;
  }
```

----

# 构建节点

当编写转换器时，你经常想构建一些待插入到AST中的节点。正如之前提到的，可以通过使用[`babel-types`](#babel-types)包中的[构建器](#builder)方法来实现。


构建器的方法名通常是待构建节点类型的名字，需要用小写字母。例如，如果你想构建`MemberExpression`，你就这样使用`t.memberExpression(...)`。

构建器的参数是由节点定义决定的。在定义上生成易于阅读的文档还有很多事要做，但是现在，可以在[这里](https://github.com/babel/babel/tree/master/packages/babel-types/src/definitions)找到他们。


一个模块定义就像下面这样：

```js
defineType("MemberExpression", {
  builder: ["object", "property", "computed"],
  visitor: ["object", "property"],
  aliases: ["Expression", "LVal"],
  fields: {
    object: {
      validate: assertNodeType("Expression")
    },
    property: {
      validate(node, key, val) {
        let expectedType = node.computed ? "Expression" : "Identifier";
        assertNodeType(expectedType)(node, key, val);
      }
    },
    computed: {
      default: false
    }
  }
});
```

这里你可以看到这个节点的所有信息，包括如何构建、遍历、校验它。 

通过观察 `builder` 属性，你看到将要被构建器方法（`t.memberExpression`）调用的三个参数。

```js
builder: ["object", "property", "computed"],
```

> 注意，有时你需要定义更多的属性，而不是这里的`builder`数组包含的。这是为了防止构建器有太多的参数。在这种情况下，你需要手动设置属性。有里有一个示例
> [`ClassMethod`](https://github.com/babel/babel/blob/bbd14f88c4eea88fa584dd877759dd6b900bf35e/packages/babel-types/src/definitions/es2015.js#L238-L276).

你可以看到构建器参数上`fields`对象的校验器

```js
fields: {
  object: {
    validate: assertNodeType("Expression")
  },
  property: {
    validate(node, key, val) {
      let expectedType = node.computed ? "Expression" : "Identifier";
      assertNodeType(expectedType)(node, key, val);
    }
  },
  computed: {
    default: false
  }
}
```

你可以看到 `object` 需是一个`Expression`，`property`也需要是一个`Expression`或`Identifier`，这依赖于成员表达式是否有 `computed`, 而且`computed`是一个简单布尔值，默认值是`false`。

因此我们能通过如下方式构造一个`MemberExpression`：

```js
t.memberExpression(
  t.identifier('object'),
  t.identifier('property')
  // `computed` is optional
);
```

其结果是:

```js
object.property
```

然而，我们说的 `Object` 需是一个 `Expression` ，那为什么 `Identifier` 是合法的？

如果我们仔细地看下 `Indentifier` 的定义，我们就能看到它有一个`alias`属性，它声明了它也是一个表达式。

```js
aliases: ["Expression", "LVal"],
```

那么，由于 `MemberExpression`是`Expression`的一种，因此我们能够设置作为另外一种`MemberExpression`中的`object`。

```js
t.memberExpression(
  t.memberExpression(
    t.identifier('member'),
    t.identifier('expression')
  ),
  t.identifier('property')
)
```

其结果就是:

```js
member.expression.property
```

这个和你以前记忆的每个节点类型的构建器方法签名很不一样。因此你需要花点时间，理解它们是如何从节点定义中生成出来的。

你可以找到所有的实际[定义](https://github.com/babel/babel/tree/master/packages/babel-types/src/definitions)，以及[文档](https://github.com/babel/babel/blob/master/doc/ast/spec.md)

----

# 最佳实践

> 在接下来的几周里，做这个章节。

## 尽可能地避免遍历AST

遍历AST是昂贵的，很容易一不小心地在非必要情形下遍历AST。这可能是几千甚至几万个额外操作。

### 尽可能合并访问者

当编写访问者时，它会在逻辑上需要的多个地方尝试调用`path.traverse`。

```js
path.traverse({
  Identifier(path) {
    // ...
  }
});

path.traverse({
  BinaryExpression(path) {
    // ...
  }
});
```

然而，最好把这些编写成一个访问者，这样的话就运行一次。否则的话，你将无缘无故地多次遍历相同的树。

```js
path.traverse({
  Identifier(path) {
    // ...
  },
  BinaryExpression(path) {
    // ...
  }
});
```

### 当手动查找做时就不要遍历

当查找一个指定类型的节点，就会尝试调用 `path.traverse`。

```js
const visitorOne = {
  Identifier(path) {
    // ...
  }
};

const MyVisitor = {
  FunctionDeclaration(path) {
    path.get('params').traverse(visitorOne);
  }
};
```

然而，如果你正在查找一些指定的或潜在的节点，你可以手动查找你需要的节点，不需要执行昂贵的遍历。 

```js
const MyVisitor = {
  FunctionDeclaration(path) {
    path.node.params.forEach(function() {
      // ...
    });
  }
};
```

## 优化嵌套的访问者

当你在嵌套访问者是，确保在你的代码中编写嵌套是有意义的。

```js
const MyVisitor = {
  FunctionDeclaration(path) {
    path.traverse({
      Identifier(path) {
        // ...
      }
    });
  }
};
```

然而，这里每当`FunctionDeclaration()`调用时就创建了一个新的访问者对象，这样Babel就需要每次暴露和校验。这个过程是昂贵的，因此更好的方式是把访问者提取到上面去。 

```js
const visitorOne = {
  Identifier(path) {
    // ...
  }
};

const MyVisitor = {
  FunctionDeclaration(path) {
    path.traverse(visitorOne);
  }
};
```

如果你嵌套的访问中的一些状态，可能就像这样：

```js
const MyVisitor = {
  FunctionDeclaration(path) {
    var exampleState = path.node.params[0].name;

    path.traverse({
      Identifier(path) {
        if (path.node.name === exampleState) {
          // ...
        }
      }
    });
  }
};
```

你可以把它传递给`traverse()`方法，在访问者中可以通过`this`来访问

```js
const visitorOne = {
  Identifier(path) {
    if (path.node.name === this.exampleState) {
      // ...
    }
  }
};

const MyVisitor = {
  FunctionDeclaration(path) {
    var exampleState = path.node.params[0].name;
    path.traverse(visitorOne, { exampleState });
  }
};
```

## 注意嵌套结构

有时当正在考虑一个指定的转换时，你可能忘记了指定的结构可能会嵌套。

例如，加上我们想从`Foo` `ClassDeclaration`中查找 `constructor`  `ClassMethod`。

```js
class Foo {
  constructor() {
    // ...
  }
}
```

```js
const constructorVisitor = {
  ClassMethod(path) {
    if (path.node.name === 'constructor') {
      // ...
    }
  }
}

const MyVisitor = {
  ClassDeclaration(path) {
    if (path.node.id.name === 'Foo') {
      path.traverse(constructorVisitor);
    }
  }
}
```

我们忽略了类可以嵌套的情形，使用上面的遍历，我们同样会遇到嵌套的`constructor`:

```js
class Foo {
  constructor() {
    class Bar {
      constructor() {
        // ...
      }
    }
  }
}
```
