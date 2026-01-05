# LLDB高阶调试

iOS开发中经常会碰到问题，解决问题最常见的便是使用LLDB调试，下面讲解部分常用的LLDB高阶调试

## Expression命令

单提`Expression`可能会有点陌生，但是如果是`po`或者`p`大家可能都很熟悉，我们平时开发中接触的最多的命令可能就是这两个了，首先在lldb看下这两个命令

``` python
(lldb) help po
Evaluate an expression on the current thread.  Displays any returned value with
formatting controlled by the type's author.  Expects 'raw' input (see 'help
raw-input'.)

Syntax: po <expr>

Command Options Usage:
  po <expr>


'po' is an abbreviation for 'expression -O  --'

(lldb) help p
Evaluate an expression on the current thread.  Displays any returned value with
LLDB's default formatting.  Expects 'raw' input (see 'help raw-input'.)

Syntax: p <expr>

Command Options Usage:
  p <expr>


'p' is an abbreviation for 'expression --'

```

可见po其实是`expression -O  --`的缩写，p则是`expression --`的缩写，那就看看`expression`命令吧

``` python
(lldb) help expression
Evaluate an expression on the current thread.  Displays any returned value with
LLDB's default formatting.  Expects 'raw' input (see 'help raw-input'.)

Syntax: expression <cmd-options> -- <expr>

......
       -A ( --show-all-children )
            Ignore the upper bound on the number of children to show.

       -D <count> ( --depth <count> )
            Set the max recurse depth when dumping aggregate types (default is
            infinity).

       -F ( --flat )
            Display results in a flat format that uses expression paths for
            each variable or member.
......
```

可见`expression`其实支持很多的`option`，简单看一个例子

``` python
(lldb) expression -P 1 -- self
(ViewController *) $12 = 0x00007f8fd1f090f0 {
  UIViewController = {
    UIResponder = {
      NSObject = {
        isa = ViewController
      }
    }
  }
  _sex = 0x000000010b44d0e8 @"123"
  _block = 0x000000010b44a900
}
```

其实在反编译的时候十分有用

## 断点

### 常规断点

Xcode创建GUI断点很简单，只要在特定的想要调试的代码处添加断点即可，但其实也可以使用命令行进行创建，命令行创建有一个优势，可以把断点打在没有源码的二进制库里面

首先看看创建断点的方法

``` python
(lldb) help b
Set a breakpoint using one of several shorthand formats.  Expects 'raw' input
(see 'help raw-input'.)

Syntax: 
_regexp-break <filename>:<linenum>:<colnum>
              main.c:12:21          // Break at line 12 and column 21 of main.c

_regexp-break <filename>:<linenum>
              main.c:12             // Break at line 12 of main.c

_regexp-break <linenum>
              12                    // Break at line 12 of current file

_regexp-break 0x<address>
              0x1234000             // Break at address 0x1234000

_regexp-break <name>
              main                  // Break in 'main' after the prologue

_regexp-break &<name>
              &main                 // Break at first instruction in 'main'

_regexp-break <module>`<name>
              libc.so`malloc        // Break in 'malloc' from 'libc.so'

_regexp-break /<source-regex>/
              /break here/          // Break on source lines in current file
                                    // containing text 'break here'.


'b' is an abbreviation for '_regexp-break'
```

可见`b`是`_regexp-break`的缩写，可以使用`b main.m:12:21`在main.m的第12行处打下断点，也可以指定某个方法，现在进行测试

```python
(lldb) b -[UIViewController viewDidLoad]
Breakpoint 7: where = UIKitCore`-[UIViewController viewDidLoad], address = 0x00007fff248416d0
```

可见在`UIViewController`的`viewDidLoad`方法处打下了断点

### 正则表达式断点

除了使用`b`进行常规断点，我们还可以使用正则表达式断点

```python
(lldb) help rb
Sets a breakpoint or set of breakpoints in the executable.

Syntax: rbreak <cmd-options>
 ......
'rb' is an abbreviation for 'breakpoint set -r %1'
```

举个简单的例子，比如我们要在`UIViewController`的所有`set`方法上设置断点

```python
(lldb) rb '\-\[UIViewController\ set'
Breakpoint 2: 74 locations.
```

还可以通过各个参数显示断点的库、文件等

### 删除断点

首先看下breakpoint命令

``` python
(lldb) help breakpoint
Commands for operating on breakpoints (see 'help b' for shorthand.)

Syntax: breakpoint <subcommand> [<command-options>]

The following subcommands are supported:
......
      delete  -- Delete the specified breakpoint(s).  If no breakpoints are
                 specified, delete them all.
......
      list    -- List some or all breakpoints at configurable levels of detail.
......
For more help on any particular subcommand, type 'help <command> <subcommand>'.
```

着重介绍下`list`以及`delete`命令

先创建部分断点

``` python
(lldb) b -[UIViewController viewDidLoad]
Breakpoint 2: where = UIKitCore`-[UIViewController viewDidLoad], address = 0x00007fff248416d0
(lldb) rb \-\[UIViewController\ set'
Breakpoint 3: 1030 locations.
```

查看所有断点

```
(lldb) breakpoint list
Current breakpoints:
1: file = '/Users/.../Documents/Demo/Demo/Demo/main.m', line = 12, exact_match = 0, locations = 1, resolved = 1, hit count = 1

  1.1: where = Debug_Objective-C`main + 22 at main.m:12:16, address = 0x000000010e828f56, resolved, hit count = 1 

2: name = '-[UIViewController viewDidLoad]', locations = 1, resolved = 1, hit count = 0
  2.1: where = UIKitCore`-[UIViewController viewDidLoad], address = 0x00007fff248416d0, resolved, hit count = 0 

3: regex = '\-\[UIViewController', locations = 1030, resolved = 1030, hit count = 0
  3.1: where = DocumentManager`-[UIViewController(DOCRecentsSupport) doc_appLocationLargeNavigationTitlesForLocation:configuration:], address = 0x00007fff2e914c28, resolved, hit count = 0 
  3.2: where = UIKitCore`-[UIViewController(UIAlertControllerContentViewController) _visualStyleOfContainingAlertController], address = 0x00007fff2439f80c, resolved, hit count = 0 
  3.3: where = UIKitCore`-[UIViewController(UIAlertControllerContentViewController) _containingAlertControllerDidChangeVisualStyle:], address = 0x00007fff2439f87b, resolved, hit count = 0 
  3.4: where = UIKitCore`-[UIViewController(UIResponderChainTraversal) _nextResponderUsingTraversalStrategy:], address = 0x00007fff246e74b0, resolved, hit count = 0 
  3.5: where = UIKitCore`__84-[UIViewController(UIResponderChainTraversal) _nextResponderUsingTraversalStrategy:]_block_invoke, address = 0x00007fff246e766a, resolved, hit count = 0 
  3.6: where = UIKitCore`-[UIViewController(UIResponderChainTraversal) _hasDeepestActionResponder], address = 0x00007fff246e774b, resolved, hit count = 0 
```

可以看到第一个断点很明显是一个GUI断点，打在main.m里面，后面就是我们刚刚打的两个断点，但是由于断点3是使用正则表达式匹配，会出现3.1、3.2等一系列断点

`list`命令也支持查看某个断点，比如`breakpoint list 1`

接下来删除断点

``` python
// 删除某个断点
(lldb) breakpoint delete 1
1 breakpoints deleted; 0 breakpoint locations disabled.

// 删除所有断点
(lldb) breakpoint delete
About to delete all breakpoints, do you want to do that?: [Y/n] 
All breakpoints removed. (2 breakpoints)
```

### 本地化断点

使用命令行断点有个很大的优势在于可以本地化一些常用的断点，然后下次直接读取使用即可

``` python
// 创建部分断点
(lldb) b -[UIViewController viewDidLoad]

// 将断点写入本地
(lldb) breakpoint write -f ~/breakpoint.json

// 读取本地断点
(lldb) breakpoint read -f ~/breakpoint.json
New breakpoints:
Breakpoint 2: where = UIKitCore`-[UIViewController viewDidLoad], address = 0x00007fff248416d0
```

这样即便换了电脑也可以将自己常用的一些复杂断点备份

## 堆栈

堆栈信息也可以在LLDB打印

``` python
(lldb) help thread
Commands for operating on one or more threads in the current process.

Syntax: thread <subcommand> [<subcommand-options>]

The following subcommands are supported:
```

可见`thread`命令可对线程就行操作，比如`thread backtrace`可以打印出当前线程的函数栈

如果想打印某一个具体的栈，可以使用`frame`命令

``` python
(lldb) help frame
Commands for selecting and examing the current thread's stack frames.

Syntax: frame <subcommand> [<subcommand-options>]

The following subcommands are supported:

      diagnose   -- Try to determine what path path the current stop location
                    used to get to a register or address
      info       -- List information about the current stack frame in the
                    current thread.
      recognizer -- Commands for editing and viewing frame recognizers.
      select     -- Select the current stack frame by index from within the
                    current thread (see 'thread backtrace'.)
      variable   -- Show variables for the current stack frame. Defaults to all
                    arguments and local variables in scope. Names of argument,
                    local, file static and file global variables can be
                    specified. Children of aggregate variables can be specified
                    such as 'var->child.x'.  The -> and [] operators in 'frame
                    variable' do not invoke operator overloads if they exist,
                    but directly access the specified element.  If you want to
                    trigger operator overloads use the expression command to
                    print the variable instead.
                    It is worth noting that except for overloaded operators,
                    when printing local variables 'expr local_var' and 'frame
                    var local_var' produce the same results.  However, 'frame
                    variable' is more efficient, since it uses debug
                    information and memory reads directly, rather than parsing
                    and evaluating an expression, which may even involve JITing
                    and running code in the target program.

For more help on any particular subcommand, type 'help <command> <subcommand>'.
```

比较常用的有`frame info`命令查看当前栈的信息，`frame select`命令选择调用栈，`frame variable`命令查看当前栈的参数（按照msg_send方法来看比如第一个参数为objc、第二个为SEL，后面才是真正传入的）

## image命令

image命令个人在反编译的时候用的比较多，还是首先查看image的介绍

``` python
(lldb) help image
Commands for accessing information for one or more target modules.

Syntax: image

'image' is an abbreviation for 'target modules'
```

从简介可知，image命令是用来访问module的

下面简单介绍几个image常用的命令

``` python
(lldb) help image lookup
Look up information within executable and dependent shared library images.

Syntax: target modules lookup <cmd-options> [<filename> [<filename> [...]]]

Command Options Usage:
  target modules lookup [-Av] -a <address-expression> [-o <offset>] [<filename> [<filename> [...]]]
  target modules lookup [-Arv] -s <symbol> [<filename> [<filename> [...]]]
  target modules lookup [-Aiv] -f <filename> [-l <linenum>] [<filename> [<filename> [...]]]
  target modules lookup [-Airv] -F <function-name> [<filename> [<filename> [...]]]
  target modules lookup [-Airv] -n <function-or-symbol> [<filename> [<filename> [...]]]
  target modules lookup [-Av] -t <name> [<filename> [<filename> [...]]]

       -A ( --all )
            Print all matches, not just the best match, if a best match is
            available.

       -F <function-name> ( --function <function-name> )
            Lookup a function by name in the debug symbols in one or more
            target modules.

       -a <address-expression> ( --address <address-expression> )
            Lookup an address in one or more target modules.

       -f <filename> ( --file <filename> )
            Lookup a file by fullpath or basename in one or more target
            modules.

       -i ( --no-inlines )
            Ignore inline entries (must be used in conjunction with --file or
            --function).

       -l <linenum> ( --line <linenum> )
            Lookup a line number in a file (must be used in conjunction with
            --file).

       -n <function-or-symbol> ( --name <function-or-symbol> )
            Lookup a function or symbol by name in one or more target modules.

       -o <offset> ( --offset <offset> )
            When looking up an address subtract <offset> from any addresses
            before doing the lookup.

       -r ( --regex )
            The <name> argument for name lookups are regular expressions.

       -s <symbol> ( --symbol <symbol> )
            Lookup a symbol by name in the symbol tables in one or more target
            modules.

       -t <name> ( --type <name> )
            Lookup a type by name in the debug symbols in one or more target
            modules.

       -v ( --verbose )
            Enable verbose lookup information.
     
     This command takes options and free-form arguments.  If your arguments
     resemble option specifiers (i.e., they start with a - or --), you must use
     ' -- ' between the end of the command options and the beginning of the
     arguments.

'image' is an abbreviation for 'target modules'
```

`image lookup`命令通常用来在可执行文件以及加载的动态库里面查找一些信息，比如

``` python
(lldb) image lookup -vn "-[UIViewController viewDidLoad]"
1 match found in /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Library/Developer/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/System/Library/PrivateFrameworks/UIKitCore.framework/UIKitCore:
        Address: UIKitCore[0x000000000050d6d0] (UIKitCore.__TEXT.__text + 5288592)
        Summary: UIKitCore`-[UIViewController viewDidLoad]
         Module: file = "/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Library/Developer/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/System/Library/PrivateFrameworks/UIKitCore.framework/UIKitCore", arch = "x86_64"
         Symbol: id = {0x000066cf}, range = [0x00007fff248416d0-0x00007fff248416d6), name="-[UIViewController viewDidLoad]"
```

可见`UIViewController`的`viewDidLoad`方法在UIKitCore的__text section偏移5288592处，在虚拟内存中的位置为0x00007fff248416d0，使用image lookup -a进行验证

``` python
(lldb) image lookup -a 0x00007fff248416d0
      Address: UIKitCore[0x000000000050d6d0] (UIKitCore.__TEXT.__text + 5288592)
      Summary: UIKitCore`-[UIViewController viewDidLoad]
```

同样的，image命令也支持正则表达式，可见上面的`-r` option ，使用正则表达式可以搜索一些很有意思的系统私有函数

## Watchpoints

watchpoint，简单的来说，就是可以监控指定的内存，当对这块内存进行读取以及写入的时候暂停执行

设置watchpoint同样有两种方法

### 通过GUI设置watchpoint

![watchpoint_gui](https://user-images.githubusercontent.com/22512175/149658687-10c565d4-15fd-4512-a945-dd1858e077a4.png)

如图，在debug界面右键点击想要观察的属性，然后点击watch"xxx"即可

### 命令行设置watchpoint

同样的，也可以使用命令行设置watchpoint

``` python
(lldb) help watchpoint
Commands for operating on watchpoints.

Syntax: watchpoint <subcommand> [<command-options>]

The following subcommands are supported:

      command -- Commands for adding, removing and examining LLDB commands
                 executed when the watchpoint is hit (watchpoint 'commands').
      delete  -- Delete the specified watchpoint(s).  If no watchpoints are
                 specified, delete them all.
      disable -- Disable the specified watchpoint(s) without removing it/them. 
                 If no watchpoints are specified, disable them all.
      enable  -- Enable the specified disabled watchpoint(s). If no watchpoints
                 are specified, enable all of them.
      ignore  -- Set ignore count on the specified watchpoint(s).  If no
                 watchpoints are specified, set them all.
      list    -- List all watchpoints at configurable levels of detail.
      modify  -- Modify the options on a watchpoint or set of watchpoints in
                 the executable.  If no watchpoint is specified, act on the
                 last created watchpoint.  Passing an empty argument clears the
                 modification.
      set     -- Commands for setting a watchpoint.

For more help on any particular subcommand, type 'help <command> <subcommand>'.
```

可以看到watchpoint其实和普通的断点十分相像，都存在delete、disable等，接下来看一个示例

``` python
// 注意：0x7f8ad4707d30为对象指针的地址，举个栗子比如当前类有个属性sex，是NSString类型，地址为0x000000010609b0c8，但是这个地址是存储这个String的地址，不是对象属性的地址，所以需要对sex取地址
(lldb) watchpoint set expression -- 0x7f93c1309da0
```

默认为写入进入watchpoint时触发，也可以设置为读写触发，详情可看`set` option

## LLDB Script

最后算是一个重头戏，自定义LLDB脚本，LLDB启动的时候会加载 **~/.lldbinit** 里的内容，在这里面可以添加一些常用的命令，并使用 **alias** 设置别名方便使用，当然还有更方便的使用方法，使用 **regex** 正则表达式，可以有更大的发挥空间，比如

``` python
command regex ivars 's/(.+)/expression -lobjc -O -- [%1 _ivarDescription]/'
```

以上命令的作用是打印出当前对象的所有ivars，输出

``` python
(lldb) ivars self.sex
<__NSCFConstantString: 0x10609b148>:
in __NSCFConstantString:
in __NSCFString:
in NSMutableString:
in NSString:
in NSObject:
	isa (Class): __NSCFConstantString (isa, 0x7fff862da668)
```

当然，看这个表达式就能知道这其实调用了一个系统方法

我们也可以编写自己的LLDB脚本，使用python编写，官方文档（），脚本学习曲线较抖，个人也属于初学者，这边就不在赘序了，感兴趣的同学可以自行翻阅文档以及网上资料，并且有很多大神也已经完成了大部分常用的脚本，可以参考学习

脚本编写完成之后，可以在~/lldbinit文件中通过`command script import`脚本加入，当然网上也有更方便的方法（其实是写了一个python脚本实现了自动化）

> 最后: LLDB其实是我们日常工作中最常用工具之一了，至此这些用法其实也是对自己的一个总结，因为里面一些命令其实没那么常用，导致自己也经常翻阅以前的笔记&文档，这也算一个记录
