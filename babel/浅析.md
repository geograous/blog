在分析 [babel](https://github.com/babel/babel) 之前，肯定得介绍一下 [ecmascript 语言规范](http://www.ecma-international.org/ecma-262/9.0/#sec-intro) 。这个文档包含了 es9 的各个方面，包括代码编码、语法规范、内置库的功能逻辑、语句和表达式的算法以及内存分配等内容。

babel 和其他编译器一样，工作流程分为了词法分析、语法分析、中间代码优化、生成中间代码这些过程。不过 babel 最终生成的代码是较低版本的 javascript 而不是机器码，更多时候中间代码优化这步的工作是将浏览器暂不支持的高版本的语法、处于提案阶段的语法转化为低版本的语法。

babel 的语法树使用的是 [estree 规范](https://github.com/estree/estree)。这个规范定义了一个完整的 ecmascript 语法树，jsx、flow、typescript 在 estree 的规范下做了一些特有的扩充。

词法分析和语法分析是密不可分的，词法分析是语法分析的基础；语法分析又是词法分析的动力和后继。比如解析 `var answer = 6 * 7;` 时，当词法分析解析出 `var` 后，babel parser 就判断到了这是一个 VarStatement ，使用 VarStatement 的解析器去解析。接下去，调用词法分析器去读下一个单词，读取到的结果是 `answer` ，这个节点作为等号左侧的值放入到语法树中，再接着读取 `6 * 7` 这个表达式，解析出来后作为等式的右侧值放入到语法树中。事实上，babel parser 中 Parser（整个解析器）继承了 StatementParser，StatementParser 继承了 ExpressionParser, 中间省略, 最后继承了 Tokenizer。在解析的过程中相互调用。上述的过程中依次读取到的 token 有：
```
[
    {
        "type": "Keyword",
        "value": "var"
    },
    {
        "type": "Identifier",
        "value": "answer"
    },
    {
        "type": "Punctuator",
        "value": "="
    },
    {
        "type": "Numeric",
        "value": "6"
    },
    {
        "type": "Punctuator",
        "value": "*"
    },
    {
        "type": "Numeric",
        "value": "7"
    },
    {
        "type": "Punctuator",
        "value": ";"
    }
]
```
最后生成的语法树结构是：
```
{
    "type": "Program",
    "body": [
        {
            "type": "VariableDeclaration",
            "declarations": [
                {
                    "type": "VariableDeclarator",
                    "id": {
                        "type": "Identifier",
                        "name": "answer"
                    },
                    "init": {
                        "type": "BinaryExpression",
                        "operator": "*",
                        "left": {
                            "type": "Literal",
                            "value": 6,
                            "raw": "6"
                        },
                        "right": {
                            "type": "Literal",
                            "value": 7,
                            "raw": "7"
                        }
                    }
                }
            ],
            "kind": "var"
        }
    ],
    "sourceType": "script"
}
```

parser 的解析过程吸收了图灵机的思想，有个内部的状态的概念，对应源码中的 state 和 context。state 描述了在词法分析过程的中间状态。其中有一个操作符 `/` 的解析较为复杂，既可以解析为 “除号”，又可以解析为“正则表达式”的开头，要判定是哪种情况就需要结合具体的语义来分析，这些情况都包含在了 context 中。

babel 有个语法树遍历的工具 ———— babel-traverse 。语法转换的过程都借助于这个工具。babel-template 能够减少一些重复的工作量。

babel-generator 的过程正好是 parser 反过来，不赘述。生成代码的过程中可以配置是否生成 sourceMap ，souceMap 的作用是建立压缩后代码和原始代码之间的映射，使得在调试过程中更加容易直观。具体详见 [阮一峰老师的博客](http://www.ruanyifeng.com/blog/2013/01/javascript_source_map.html)
