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
