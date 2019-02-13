ejs是一款历史悠久的模版，具有简单、性能好、使用广泛的特点，可以被用到很多的地方。比如cms系统、ssr等。这里会介绍ejs项目的[源码](https://github.com/mde/ejs)。使用方法详见[项目](https://github.com/mde/ejs)的readme，或者[这里](http://ejs.co)。

**哲学**
ejs的思想是`模版 + 数据 => 结果`。`模版`是字符串的格式，包含普通的字符串和占位符，占位符最终被`数据`替换。使用`include`方法还可以引用其他模版。

**一些概念**
+ `template`：即模版，比如这个栗子👇。
```html
<% if (user) { %>
  <h2><%= user.name %></h2>
<% } %>
```
+ `data`: 模版对应的数据，具有模版中使用的所有变量，比如上面这个栗子中必须具有`user.name`这个数据。
+ `option`：模版配置项。有这些👇：
  - `cache`                 Compiled functions are cached, requires `filename`
  - `filename`              The name of the file being rendered. Not required if you
    are using `renderFile()`. Used by `cache` to key caches, and for includes.
  - `root`                  Set project root for includes with an absolute path (/file.ejs).
  - `context`               Function execution context
  - `compileDebug`          When `false` no debug instrumentation is compiled
  - `client`                When `true`, compiles a function that can be rendered
    in the browser without needing to load the EJS Runtime
    ([ejs.min.js](https://github.com/mde/ejs/releases/latest)).
  - `delimiter`             Character to use with angle brackets for open/close
  - `debug`                 Output generated function body
  - `strict`                When set to `true`, generated function is in strict mode
  - `_with`                 Whether or not to use `with() {}` constructs. If `false`
    then the locals will be stored in the `locals` object. Set to `false` in strict mode.
  - `localsName`            Name to use for the object storing local variables when not using
    `with` Defaults to `locals`
  - `rmWhitespace`          Remove all safe-to-remove whitespace, including leading
    and trailing whitespace. It also enables a safer version of `-%>` line
    slurping for all scriptlet tags (it does not strip new lines of tags in
    the middle of a line).
  - `escape`                The escaping function used with `<%=` construct. It is
    used in rendering and is `.toString()`ed in the generation of client functions.
    (By default escapes XML).
  - `outputFunctionName`    Set to a string (e.g., 'echo' or 'print') for a function to print
    output inside scriptlet tags.
  - `async`                 When `true`, EJS will use an async function for rendering. (Depends
    on async/await support in the JS runtime.
+ `compile`：编译函数，把template和option转化为一个函数，往这个函数中注入数据，生成最终的字符串，不一定是html哦，还可以是各种形式的字符串。
+ `render`：渲染函数，直接把template、data和option转化为最终的字符串。

**主流程**
ejs引擎的实现思路是把配置的模版转化为渲染的函数，再通过的数据生成字符串。把模版转化为渲染函数的这个过程就是`compile`。它的主要工作就是通生成函数输入和函数体的字符串，再通过Function这个类来生成函数。执行流程分别为：

 1. 根据正则表达式切割模版，比如`{ key1 = <%= key1 %>, 2key1 = <%= key1+key1 %> }`会被切割成`[ '{ key1 = ', '<%=', ' key1 ', '%>', ', 2key1 = ', '<%=', ' key1+key1 ', '%>', ' }' ]`
 2. 根据切割后的数据，生成渲染函数中的执行步骤。上面这个栗子中，执行步骤为
```
'    ; __append("{ key1 = ")\n    ; __append(escapeFn( key1 ))\n    ; __append(", 2key1 = ")\n    ; __append(escapeFn( key1+key1 ))\n    ; __append(" }")\n'
```
 3. 组装函数，通过prepend+执行步骤+append的模式生成函数体的字符串，最后生成这样的函数头、函数体，以及渲染函数。
```
opts.localsName + ', escapeFn, include, rethrow'
```
```
var __output = [], __append = __output.push.bind(__output);
  with (locals || {}) {
     ; __append("{ key1 = ")
    ; __append(escapeFn( key1 ))
    ; __append(", 2key1 = ")
    ; __append(escapeFn( key1+key1 ))
    ; __append(" }")
   }
  return __output.join("");
```
```
function (data) {
  var include = function (path, includeData) {
    var d = utils.shallowCopy({}, data);
    if (includeData) {
      d = utils.shallowCopy(d, includeData);
    }
    return includeFile(path, opts)(d);
  };
  return fn.apply(opts.context, [data || {}, escapeFn, include, rethrow]);
}
```
通过渲染函数和数据生成结果的方式和react比较像哈。但ejs比较简单明了，在不直接支持嵌套，而是通过include方法调用子模版的渲染函数。

**一些细节**
渲染函数的4个参数：`data`、`escapeFn`、`include`、`rethrow`。

 - `data`: 传入的数据。
 - `escapeFn`: 转义函数。
 - `include`: 引入子模版函数。主要的逻辑是根据路径获取模版，并且编译生成渲染函数进行缓存，最后进行渲染。
```
var include = function (path, includeData) {
    var d = utils.shallowCopy({}, data);
    if (includeData) {
      d = utils.shallowCopy(d, includeData);
    }
    return includeFile(path, opts)(d);
  };
```
 - `rethrow`: 抛出异常函数。

生成渲染函数的执行步骤。
这一步是在模版被切割之后进行的，首先模版遇到ejs的标签时就会被切割，切割后的字符串中标签是成对出现的，引用一下上面的栗子。
```
[ '{ key1 = ', '<%=', ' key1 ', '%>', ', 2key1 = ', '<%=', ' key1+key1 ', '%>', ' }' ]
```
ejs会根据不同的标签生成不同的执行步骤。执行过程中会遍历整个数组。由于标签不能嵌套，而且成对出现，正好可以利用全局的变量来保存当前标签的类型，执行到夹在一对标签中的内容时，可以获取到外层标签信息。当执行到闭合标签时，重置标签信息。

**关于路径**
先了解一下`include`方法，ejs的语法不支持嵌套，只能通过这个方法来复用模版。下面是一个使用的栗子。
```
<ul>
  <% users.forEach(function(user){ %>
    <%- include('user/show', {user: user}) %>
  <% }); %>
</ul>
```
在使用`include`方法时，需要传入复用`template`的路径和`data`。路径的逻辑先会看是否是绝对路径，然后会拼接传入的路径参数和`options.filename`，如果不存在这个文件最后看`views`的目录下是否存在这个文件，代码请看👇
```
function getIncludePath(path, options) {
  var includePath;
  var filePath;
  var views = options.views;

  // Abs path
  if (path.charAt(0) == '/') {
    includePath = exports.resolveInclude(path.replace(/^\/*/,''), options.root || '/', true);
  }
  // Relative paths
  else {
    // Look relative to a passed filename first
    if (options.filename) {
      filePath = exports.resolveInclude(path, options.filename);
      if (fs.existsSync(filePath)) {
        includePath = filePath;
      }
    }
    // Then look in any views directories
    if (!includePath) {
      if (Array.isArray(views) && views.some(function (v) {
        filePath = exports.resolveInclude(path, v, true);
        return fs.existsSync(filePath);
      })) {
        includePath = filePath;
      }
    }
    if (!includePath) {
      throw new Error('Could not find the include file "' +
          options.escapeFunction(path) + '"');
    }
  }
  return includePath;
}
```
这就意味着在使用`include`的时候，子`template`文件只能在`views`目录下，后缀为`ejs`的文件。或者设置`options.filename`变量，文件分布在不同的目录下。这个就比较坑了，使用起来很不方便。当嵌套层次比较高时，怎么复用模版？貌似只能通过绝对路径的方式了。

