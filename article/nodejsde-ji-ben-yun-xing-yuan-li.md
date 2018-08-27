\#\#\#\# nodeJs文件目录

    .

    ├── ...

    ├── deps

    │   ├── ...

    │   ├── v8

    │   ├── ...

    ├── ...

    ├── lib

    │   ├── ...

    │   ├── buffer.js

    │   ├── child\_process.js

    │   ├── console.js

    │   ├── ...

    ├── node -&gt; out/Release/node

    ├── ...

    ├── out

    │   ├── ...

    │   ├── Release

    \|         ├── node

    \|         ├── node.d

    \|         ├── obj

    \|             └── gen

    \|                 ├── ...

    \|                 ├── node\_natives.h

    \|                 ├── ...

    │   ├── ...

    ├── src

    │   ├── ...

    │   ├── debug-agent.cc

    │   ├── debug-agent.h

    │   ├── env-inl.h

    │   ├── env.cc

    │   ├── ...

    ├── 

    ...

    

需要关注的几个目录和文件：



==/deps/v8==：这里是V8源码所在文件夹，你会发现里面的目录结构跟V8源码十分相似。NodeJS除了移植V8源码，还在增添了一些内容。



==/src==：由C/C++编写的核心模块所在文件夹，由C/C++编写的这部分模块被称为「Builtin Module」



==/lib==：由JavaScript编写的核心模块所在文件夹，这部分被称为「Native Code」，在编译Node源码的时候，会采用V8附带的js2c.py工具，把所有内置的JavaScript代码转换成C++里面的数组，生成out/Release/obj/gen/node\_natives.h文件。有些 Native Module 需要借助于 Builtin Module 实现背后的功能。



==/out==：该目录是Node源码编译\(即命令行运行make\)后生成的目录，里面包含了Node的可执行文件。当在命令行中键入node xxx.js，实际就是运行了out/Release/node文件。



\`\`\`

graph TD

A\[lib/\*.js\] --&gt; \| js2c.py\|B\(node\_native.h\)

B --&gt; \|g++ complie\| D\(Node\)

C\[src/\*.cc + src/\*.h\] --&gt; \|g++ complie\|D

D --&gt; E\(memory\)

F\[other Js module\] --&gt; J\[require调用\]

J--&gt; K\(V8 parser\)

K --&gt; E



\`\`\`

    Node在启动的时候，就已经把 Native Module，Builtin Module 加载到内存里面了，这样可以供全局使用。后来的 JavaScript 代码，就需要通过 V8 进行动态编译解析运行。

    

    V8 作为一个 JavaScript 引擎，最初是服役于 Google Chrome 浏览器的。它随着 Chrome 的第一版发布而发布以及开源。现在它除了 Chrome 浏览器，已经有很多其他的使用者了。诸如 NodeJS、MongoDB、CouchDB 等。

    

    JavaScript 作为 Prototype-Based Language , 基于它使用 Prototype 继承的特征，V8 使用了直译的方式，即把 JavaScript 代码直接编译成机器码\( Machine Code, 有些地方也叫 Native Code \)，然后直接交由硬件执行。

    与传统的「编译-解析-执行」的流程不同，V8 处理 JavaScript，并没有二进制码或其他的中间码。

    

    简单来说，V8主要工作就是：「把 JavaScript 直译成机器码，然后运行」

    但这中间，往往是一个复杂的过程，它需要处理很多的难题，诸如：

    

        1.编译优化

        2.内存管理

        3.垃圾回收



\#\#\#\# C/C++编写的核心模块，调用方法基本运行原理

    \#include "v8.h"  

    \#include &lt;string.h&gt;  

    \#include &lt;stdio.h&gt;  

      

    using namespace v8;  

    using namespace std;  

      

    Handle&lt;Value&gt; Hi\(const Arguments& args\) {  

        HandleScope  handle\_scope;  

        char buffer\[4096\];  

          

        memset\(buffer, 0, sizeof\(buffer\)\);  

        Handle&lt;String&gt; str = args\[0\]-&gt;ToString\(\);  

        str-&gt;WriteAscii\(buffer\);  

        printf\("Yell: %s\n", buffer\);  

      

        return Undefined\(\);  

    }  

      

    int main\(int argc, char\*\* argv\) {  

        HandleScope handle\_scope;  

      

        //定义一个FunctionTempte并与C++函数绑定

        Handle&lt;FunctionTemplate&gt; fun = FunctionTemplate::New\(Hi\);  

        //定义一个ObectTemplate，并向该对象注册一个FunctionTemplate

        Handle&lt;ObjectTemplate&gt; global = ObjectTemplate::New\(\);  

        global-&gt;Set\(String::New\("Hi"\), fun\);  

        //将该对象注册到JS的global中去 

        Persistent&lt;Context&gt; cxt = Context::New\(NULL, global\);  

      

        Context::Scope context\_scope\(cxt\);  

        Handle&lt;String&gt; source = String::New\("Hi\('Hi V8!'\)"\);  

        Handle&lt;Script&gt; script = Script::Compile\(source\);  

        Handle&lt;Value&gt; result = script-&gt;Run\(\);  

      

        cxt.Dispose\(\);  

    }  





\`\`\`

using namespace v8;  

  

int main\(int argc, char\* argv\[\]\) {  

  // Get the default Isolate created at startup.  

  Isolate\* isolate = Isolate::GetCurrent\(\);  

  

  // Create a stack-allocated handle scope.  

  HandleScope handle\_scope\(isolate\);  

  

  // Create a new context.  

  Handle&lt;Context&gt; context = Context::New\(isolate\);  

  

  // Here's how you could create a Persistent handle to the context, if needed.  

  Persistent&lt;Context&gt; persistent\_context\(isolate, context\);  

    

  // Enter the created context for compiling and  

  // running the hello world script.   

  Context::Scope context\_scope\(context\);  

  

  // 创建一个js的字符串的文件  

  Handle&lt;String&gt; source = String::New\("'Hello' + ', World!'"\);  

  

  // 编译原文件 

  Handle&lt;Script&gt; script = Script::Compile\(source\);  

    

  // 运行Js得到返回结果.  

  Handle&lt;Value&gt; result = script-&gt;Run\(\);  

    

  // The persistent handle needs to be eventually disposed.  

  persistent\_context.Dispose\(\);  

  

  // Convert the result to an ASCII string and print it.  

  String::AsciiValue ascii\(result\);  

  printf\("%s\n", \*ascii\);  

  return 0;  

\`\`\`



