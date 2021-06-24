# 背景

在平时开发过程中也经常会使用一些私有类进行调试，但是Apple经常会更新自己的动态库，可能原先的使用方法就不适用，下面就以**UIDebuggingInformationOverlay**这个Apple的调试工具来进行举例，碰到这种情况下的常用解决办法。**UIDebuggingInformationOverlay**是一个Apple于iOS9推出的用于调试神器，主要用于对正在运行的应用进行样式的修改和控件的读取，在iOS10及以下调用`prepareDebuggingOverlay`函数即可使用，但是iOS10之后，Apple对其进行了限制，我们普通的设备无法调用。

# 探究prepareDebuggingOverlay方法iOS10之后的变化

首先看下iOS12下`prepareDebuggingOverlay`函数做了什么，输入`disassemble -n "-[UIDebuggingInformationOverlay prepareDebuggingOverlay]"`进行反编译

``` assembly
UIKitCore`+[UIDebuggingInformationOverlay prepareDebuggingOverlay]:
    0x10db748c4 <+0>:   push   rbp
    0x10db748c5 <+1>:   mov    rbp, rsp
    0x10db748c8 <+4>:   push   r15
    0x10db748ca <+6>:   push   r14
    0x10db748cc <+8>:   push   r13
    0x10db748ce <+10>:  push   r12
    0x10db748d0 <+12>:  push   rbx
    0x10db748d1 <+13>:  push   rax
    0x10db748d2 <+14>:  call   0x10db7572f               ; _UIGetDebuggingOverlayEnabled
    0x10db748d7 <+19>:  test   al, al
    0x10db748d9 <+21>:  je     0x10db749e2               ; <+286>
    0x10db748df <+27>:  lea    rax, [rip + 0x9bf222]     ; UIApp
    0x10db748e6 <+34>:  mov    rdi, qword ptr [rax]
    0x10db748e9 <+37>:  mov    rsi, qword ptr [rip + 0x90a918] ; "statusBarWindow"
    0x10db748f0 <+44>:  mov    r13, qword ptr [rip + 0x4e4bb9] ; (void *)0x0000000109ef3640: objc_msgSend
    0x10db748f7 <+51>:  call   r13
    0x10db748fa <+54>:  mov    rdi, rax
    0x10db748fd <+57>:  call   0x10dd5d758               ; symbol stub for: objc_retainAutoreleasedReturnValue
    0x10db74902 <+62>:  mov    qword ptr [rbp - 0x30], rax
    0x10db74906 <+66>:  mov    rdi, qword ptr [rip + 0x94a6a3] ; (void *)0x000000010e4f86c0: UITapGestureRecognizer
    0x10db7490d <+73>:  mov    rsi, qword ptr [rip + 0x90045c] ; "alloc"
    0x10db74914 <+80>:  call   r13
    0x10db74917 <+83>:  mov    r12, rax
    0x10db7491a <+86>:  mov    rdi, qword ptr [rip + 0x94ede7] ; (void *)0x000000010e514780: UIDebuggingInformationOverlayInvokeGestureHandler
    0x10db74921 <+93>:  mov    r15, qword ptr [rip + 0x941408] ; "mainHandler"
    0x10db74928 <+100>: mov    rsi, r15
    0x10db7492b <+103>: call   r13
    0x10db7492e <+106>: mov    rdi, rax
    0x10db74931 <+109>: call   0x10dd5d758               ; symbol stub for: objc_retainAutoreleasedReturnValue
    0x10db74936 <+114>: mov    rbx, rax
    0x10db74939 <+117>: mov    rcx, qword ptr [rip + 0x9413f8] ; "_handleActivationGesture:"
    0x10db74940 <+124>: mov    rsi, qword ptr [rip + 0x9009d1] ; "initWithTarget:action:"
    0x10db74947 <+131>: mov    rdi, r12
    0x10db7494a <+134>: mov    rdx, rax
    0x10db7494d <+137>: call   r13
    0x10db74950 <+140>: mov    r12, rax
    0x10db74953 <+143>: mov    r14, qword ptr [rip + 0x4e4b5e] ; (void *)0x0000000109ef0990: objc_release
    0x10db7495a <+150>: mov    rdi, rbx
    0x10db7495d <+153>: call   r14
    0x10db74960 <+156>: mov    rdi, qword ptr [rip + 0x94eda1] ; (void *)0x000000010e514780: UIDebuggingInformationOverlayInvokeGestureHandler
    0x10db74967 <+163>: mov    rsi, r15
    0x10db7496a <+166>: call   r13
    0x10db7496d <+169>: mov    rdi, rax
    0x10db74970 <+172>: call   0x10dd5d758               ; symbol stub for: objc_retainAutoreleasedReturnValue
    0x10db74975 <+177>: mov    rbx, rax
    0x10db74978 <+180>: mov    rsi, qword ptr [rip + 0x9009a9] ; "setDelegate:"
    0x10db7497f <+187>: mov    rdi, r12
    0x10db74982 <+190>: mov    rdx, rax
    0x10db74985 <+193>: call   r13
    0x10db74988 <+196>: mov    rdi, rbx
    0x10db7498b <+199>: call   r14
    0x10db7498e <+202>: mov    rsi, qword ptr [rip + 0x90712b] ; "setNumberOfTouchesRequired:"
    0x10db74995 <+209>: mov    edx, 0x2
    0x10db7499a <+214>: mov    rdi, r12
    0x10db7499d <+217>: call   r13
    0x10db749a0 <+220>: mov    rsi, qword ptr [rip + 0x907111] ; "setNumberOfTapsRequired:"
    0x10db749a7 <+227>: mov    edx, 0x1
    0x10db749ac <+232>: mov    rdi, r12
    0x10db749af <+235>: call   r13
    0x10db749b2 <+238>: mov    rsi, qword ptr [rip + 0x900977] ; "addGestureRecognizer:"
    0x10db749b9 <+245>: mov    rbx, qword ptr [rbp - 0x30]
    0x10db749bd <+249>: mov    rdi, rbx
    0x10db749c0 <+252>: mov    rdx, r12
    0x10db749c3 <+255>: call   r13
    0x10db749c6 <+258>: mov    rdi, r12
    0x10db749c9 <+261>: call   r14
    0x10db749cc <+264>: mov    rdi, rbx
    0x10db749cf <+267>: mov    rax, r14
    0x10db749d2 <+270>: add    rsp, 0x8
    0x10db749d6 <+274>: pop    rbx
    0x10db749d7 <+275>: pop    r12
    0x10db749d9 <+277>: pop    r13
    0x10db749db <+279>: pop    r14
    0x10db749dd <+281>: pop    r15
    0x10db749df <+283>: pop    rbp
    0x10db749e0 <+284>: jmp    rax
    0x10db749e2 <+286>: add    rsp, 0x8
    0x10db749e6 <+290>: pop    rbx
    0x10db749e7 <+291>: pop    r12
    0x10db749e9 <+293>: pop    r13
    0x10db749eb <+295>: pop    r14
    0x10db749ed <+297>: pop    r15
    0x10db749ef <+299>: pop    rbp
    0x10db749f0 <+300>: ret    
```

代码比较多，先看核心的部分

``` assembly
0x10db748d2 <+14>:  call   0x10db7572f               ; _UIGetDebuggingOverlayEnabled
0x10db748d7 <+19>:  test   al, al
0x10db748d9 <+21>:  je     0x10db749e2               ; <+286>
```

rax为返回值寄存器，调用`_UIGetDebuggingOverlayEnabled`返回到rax，然后确认rax是否有值，如果为1，改变标志寄存器为0，如果为0，标志寄存器为1，则跳转0x10db749e2，可看到0x10db749e2是函数末尾，所以`_UIGetDebuggingOverlayEnabled`是关键。但是这边先不探究`_UIGetDebuggingOverlayEnabled`（后续会解释原因），把这段汇编翻译成伪代码大概是

``` objective-c
+ (void)prepareDebuggingOverlay {
  if(_UIGetDebuggingOverlayEnabled) {	①
    UIView *statusView = [UIApp statusBarWindow];	②
    UIDebuggingInformationOverlayInvokeGestureHandler *hander = [UIDebuggingInformationOverlayInvokeGestureHandler mainHandler]; ③
    UITapGestureRecognizer *ges = [[UITapGestureRecognizer alloc] initWithTarget:hander action:@selector(_handleActivationGesture:)]; ④
    [ges setNumberOfTouchesRequired:2]; ⑤
    [ges setNumberOfTapsRequired:1]; ⑥
    [statusView addGestureRecognizer:ges]; ⑦
  }
}
```

我们来慢慢解析如何得到的

1. ``` assembly
   // 调用_UIGetDebuggingOverlayEnabled，得到返回值存储于rax，使用test指令对rax进行测试，为0标志位设为1，跳转到0x10db749e2，即函数末尾（上述可以查到），为1则继续走下去
   0x10db748d2 <+14>:  call   0x10db7572f               ; _UIGetDebuggingOverlayEnabled	
   0x10db748d7 <+19>:  test   al, al
   0x10db748d9 <+21>:  je     0x10db749e2               ; <+286>
   ```

2. ``` assembly
   // UIApp移入rdi，"statusBarWindow"移入rsi，调用，返回值存储入[rbp - 0x30]（栈基地址-0x30）
   0x10db748df <+27>:  lea    rax, [rip + 0x9bf222]     ; UIApp
   0x10db748e6 <+34>:  mov    rdi, qword ptr [rax]
   0x10db748e9 <+37>:  mov    rsi, qword ptr [rip + 0x90a918] ; "statusBarWindow"
   0x10db748f0 <+44>:  mov    r13, qword ptr [rip + 0x4e4bb9] ; (void *)0x0000000109ef3640: objc_msgSend
   0x10db748f7 <+51>:  call   r13
   0x10db748fa <+54>:  mov    rdi, rax
   0x10db748fd <+57>:  call   0x10dd5d758               ; symbol stub for: objc_retainAutoreleasedReturnValue
   0x10db74902 <+62>:  mov    qword ptr [rbp - 0x30], rax
   ```

3. ``` assembly
   // UIDebuggingInformationOverlayInvokeGestureHandler移入rdi，"mainHandler"移入rsi，OC方法消息传递以一个参数为调用者，第二个为SEL，call，返回值存储在rax，调用retain
   0x10db7491a <+86>:  mov    rdi, qword ptr [rip + 0x94ede7] ; (void *)0x000000010e514780: UIDebuggingInformationOverlayInvokeGestureHandler
   0x10db74921 <+93>:  mov    r15, qword ptr [rip + 0x941408] ; "mainHandler"
   0x10db74928 <+100>: mov    rsi, r15
   0x10db7492b <+103>: call   r13
   0x10db7492e <+106>: mov    rdi, rax
   0x10db74931 <+109>: call   0x10dd5d758               ; symbol stub for: objc_retainAutoreleasedReturnValue
   0x10db74936 <+114>: mov    rbx, rax
   ```

4. 这边比较复杂，需要拆解一下

   ``` assembly
   // 调用[UITapGestureRecognizer alloc]，返回值存储在r12
   0x10db74906 <+66>:  mov    rdi, qword ptr [rip + 0x94a6a3] ; (void *)0x000000010e4f86c0: UITapGestureRecognizer
   0x10db7490d <+73>:  mov    rsi, qword ptr [rip + 0x90045c] ; "alloc"
   0x10db74914 <+80>:  call   r13
   0x10db74917 <+83>:  mov    r12, rax
   ```

   ``` assembly
   // [UITapGestureRecognizer alloc]返回值移入rdi，"initWithTarget:action:"为SEL移入rsi，前面[UIDebuggingInformationOverlayInvokeGestureHandler mainHandler]的返回值存储在rax未动，移入rdi，"_handleActivationGesture:"这个SEL作为第四个参数移入rcx，调用，返回值存储于r12
   0x10db74939 <+117>: mov    rcx, qword ptr [rip + 0x9413f8] ; "_handleActivationGesture:"
   0x10db74940 <+124>: mov    rsi, qword ptr [rip + 0x9009d1] ; "initWithTarget:action:"
   0x10db74947 <+131>: mov    rdi, r12
   0x10db7494a <+134>: mov    rdx, rax
   0x10db7494d <+137>: call   r13
   0x10db74950 <+140>: mov    r12, rax
   ```

5. ``` assembly
   // 5、6比较类似，"setNumberOfTouchesRequired:"作为SEL，r12存储着[[UITapGestureRecognizer alloc] init...]的返回值，用于第一个参数
   0x10db7498e <+202>: mov    rsi, qword ptr [rip + 0x90712b] ; "setNumberOfTouchesRequired:"
   0x10db74995 <+209>: mov    edx, 0x2
   0x10db7499a <+214>: mov    rdi, r12
   0x10db7499d <+217>: call   r13
   ```

6. ``` assembly
   0x10db749a0 <+220>: mov    rsi, qword ptr [rip + 0x907111] ; "setNumberOfTapsRequired:"
   0x10db749a7 <+227>: mov    edx, 0x1
   0x10db749ac <+232>: mov    rdi, r12
   0x10db749af <+235>: call   r13
   ```

7. ``` assembly
   // 前面可以看到，[UIApp statusBarWindow]的返回值移入[rbp - 0x30]，r12存着的UITapGestureRecognizer对象
   0x10db749b2 <+238>: mov    rsi, qword ptr [rip + 0x900977] ; "addGestureRecognizer:"
   0x10db749b9 <+245>: mov    rbx, qword ptr [rbp - 0x30]
   0x10db749bd <+249>: mov    rdi, rbx
   0x10db749c0 <+252>: mov    rdx, r12
   0x10db749c3 <+255>: call   r13
   ```



可以看到核心是先看`UIDebuggingInformationOverlayInvokeGestureHandler`这个类，通过这个类去创建一个手势，手势的调用。



# UIDebuggingInformationOverlayInvokeGestureHandler

前面可以看到`UIDebuggingInformationOverlayInvokeGestureHandler`类的关键方法是`_handleActivationGesture`，同样，进行反编译`disassemble -n "-[UIDebuggingInformationOverlayInvokeGestureHandler _handleActivationGesture:]"`

``` assembly
UIKitCore`-[UIDebuggingInformationOverlayInvokeGestureHandler _handleActivationGesture:]:
    0x10db74537 <+0>:   push   rbp
    0x10db74538 <+1>:   mov    rbp, rsp
    0x10db7453b <+4>:   push   r15
    0x10db7453d <+6>:   push   r14
    0x10db7453f <+8>:   push   r12
    0x10db74541 <+10>:  push   rbx
    0x10db74542 <+11>:  mov    rbx, rdi
    0x10db74545 <+14>:  mov    rsi, qword ptr [rip + 0x900eac] ; "state"
    0x10db7454c <+21>:  mov    rdi, rdx
    0x10db7454f <+24>:  call   qword ptr [rip + 0x4e4f5b] ; (void *)0x0000000109ef3640: objc_msgSend
    0x10db74555 <+30>:  cmp    rax, 0x3
    0x10db74559 <+34>:  jne    0x10db7469d               ; <+358>
    0x10db7455f <+40>:  mov    r15, qword ptr [rip + 0x96d0c2] ; UIDebuggingInformationOverlayInvokeGestureHandler._didCreateTools
    0x10db74566 <+47>:  cmp    byte ptr [rbx + r15], 0x0
    0x10db7456b <+52>:  jne    0x10db7465c               ; <+293>
    0x10db74571 <+58>:  mov    rdi, qword ptr [rip + 0x94f198] ; (void *)0x000000010e514b90: UIDebuggingInformationHierarchyViewController
    0x10db74578 <+65>:  mov    r14, qword ptr [rip + 0x9007a9] ; "class"
    0x10db7457f <+72>:  mov    r12, qword ptr [rip + 0x4e4f2a] ; (void *)0x0000000109ef3640: objc_msgSend
    0x10db74586 <+79>:  mov    rsi, r14
    0x10db74589 <+82>:  call   r12
    0x10db7458c <+85>:  lea    rdi, [rip + 0x564d1d]     ; @"View Hierarchy"
    0x10db74593 <+92>:  mov    rsi, rax
    0x10db74596 <+95>:  call   0x10db746a6               ; UIDebuggingViewControllerAtTopLevel
    0x10db7459b <+100>: mov    rdi, rax
    0x10db7459e <+103>: call   0x10dd5d7a0               ; symbol stub for: objc_unsafeClaimAutoreleasedReturnValue
    0x10db745a3 <+108>: mov    rdi, qword ptr [rip + 0x94f16e] ; (void *)0x000000010e514d20: UIDebuggingInformationVCHierarchyViewController
    0x10db745aa <+115>: mov    rsi, r14
    0x10db745ad <+118>: call   r12
    0x10db745b0 <+121>: lea    rdi, [rip + 0x564d19]     ; @"VC Hierarchy"
    0x10db745b7 <+128>: mov    rsi, rax
    0x10db745ba <+131>: call   0x10db746a6               ; UIDebuggingViewControllerAtTopLevel
    0x10db745bf <+136>: mov    rdi, rax
    0x10db745c2 <+139>: call   0x10dd5d7a0               ; symbol stub for: objc_unsafeClaimAutoreleasedReturnValue
    0x10db745c7 <+144>: mov    rdi, qword ptr [rip + 0x94f152] ; (void *)0x000000010e514dc0: UIDebuggingIvarViewController
    0x10db745ce <+151>: mov    rsi, r14
    0x10db745d1 <+154>: call   r12
    0x10db745d4 <+157>: lea    rdi, [rip + 0x564d15]     ; @"Ivar Explorer"
    0x10db745db <+164>: mov    rsi, rax
    0x10db745de <+167>: call   0x10db746a6               ; UIDebuggingViewControllerAtTopLevel
    0x10db745e3 <+172>: mov    rdi, rax
    0x10db745e6 <+175>: call   0x10dd5d7a0               ; symbol stub for: objc_unsafeClaimAutoreleasedReturnValue
    0x10db745eb <+180>: mov    rdi, qword ptr [rip + 0x94f136] ; (void *)0x000000010e5150e0: UIDebuggingZoomViewController
    0x10db745f2 <+187>: mov    rsi, r14
    0x10db745f5 <+190>: call   r12
    0x10db745f8 <+193>: lea    rdi, [rip + 0x564d11]     ; @"Measure"
    0x10db745ff <+200>: mov    rsi, rax
    0x10db74602 <+203>: call   0x10db746a6               ; UIDebuggingViewControllerAtTopLevel
    0x10db74607 <+208>: mov    rdi, rax
    0x10db7460a <+211>: call   0x10dd5d7a0               ; symbol stub for: objc_unsafeClaimAutoreleasedReturnValue
    0x10db7460f <+216>: mov    rdi, qword ptr [rip + 0x94f11a] ; (void *)0x000000010e514f00: UIDebuggingSpecViewController
    0x10db74616 <+223>: mov    rsi, r14
    0x10db74619 <+226>: call   r12
    0x10db7461c <+229>: lea    rdi, [rip + 0x564d0d]     ; @"Spec Compare"
    0x10db74623 <+236>: mov    rsi, rax
    0x10db74626 <+239>: call   0x10db746a6               ; UIDebuggingViewControllerAtTopLevel
    0x10db7462b <+244>: mov    rdi, rax
    0x10db7462e <+247>: call   0x10dd5d7a0               ; symbol stub for: objc_unsafeClaimAutoreleasedReturnValue
    0x10db74633 <+252>: mov    rdi, qword ptr [rip + 0x94f0fe] ; (void *)0x000000010e514e60: UIDebuggingColorAuditViewController
    0x10db7463a <+259>: mov    rsi, r14
    0x10db7463d <+262>: call   r12
    0x10db74640 <+265>: lea    rdi, [rip + 0x564d09]     ; @"System Color Audit"
    0x10db74647 <+272>: mov    rsi, rax
    0x10db7464a <+275>: call   0x10db746a6               ; UIDebuggingViewControllerAtTopLevel
    0x10db7464f <+280>: mov    rdi, rax
    0x10db74652 <+283>: call   0x10dd5d7a0               ; symbol stub for: objc_unsafeClaimAutoreleasedReturnValue
    0x10db74657 <+288>: mov    byte ptr [rbx + r15], 0x1
    0x10db7465c <+293>: mov    rdi, qword ptr [rip + 0x94df85] ; (void *)0x000000010e5147a8: UIDebuggingInformationOverlay
    0x10db74663 <+300>: mov    rsi, qword ptr [rip + 0x93641e] ; "overlay"
    0x10db7466a <+307>: mov    r14, qword ptr [rip + 0x4e4e3f] ; (void *)0x0000000109ef3640: objc_msgSend
    0x10db74671 <+314>: call   r14
    0x10db74674 <+317>: mov    rdi, rax
    0x10db74677 <+320>: call   0x10dd5d758               ; symbol stub for: objc_retainAutoreleasedReturnValue
    0x10db7467c <+325>: mov    rbx, rax
    0x10db7467f <+328>: mov    rsi, qword ptr [rip + 0x94169a] ; "toggleVisibility"
    0x10db74686 <+335>: mov    rdi, rax
    0x10db74689 <+338>: call   r14
    0x10db7468c <+341>: mov    rdi, rbx
    0x10db7468f <+344>: pop    rbx
    0x10db74690 <+345>: pop    r12
    0x10db74692 <+347>: pop    r14
    0x10db74694 <+349>: pop    r15
    0x10db74696 <+351>: pop    rbp
    0x10db74697 <+352>: jmp    qword ptr [rip + 0x4e4e1b] ; (void *)0x0000000109ef0990: objc_release
    0x10db7469d <+358>: pop    rbx
    0x10db7469e <+359>: pop    r12
    0x10db746a0 <+361>: pop    r14
    0x10db746a2 <+363>: pop    r15
    0x10db746a4 <+365>: pop    rbp
    0x10db746a5 <+366>: ret    
```

看到这个DWARF信息就知道成功了一大半了，其中包含的@"View Hierarchy"等就是我们想要的视图信息

这边需要重点关注的

``` assembly
0x10db7465c <+293>: mov    rdi, qword ptr [rip + 0x94df85] ; (void *)0x000000010e5147a8: UIDebuggingInformationOverlay
0x10db74663 <+300>: mov    rsi, qword ptr [rip + 0x93641e] ; "overlay"
0x10db7466a <+307>: mov    r14, qword ptr [rip + 0x4e4e3f] ; (void *)0x0000000109ef3640: objc_msgSend
0x10db74671 <+314>: call   r14
0x10db74674 <+317>: mov    rdi, rax
0x10db74677 <+320>: call   0x10dd5d758               ; symbol stub for: objc_retainAutoreleasedReturnValue
0x10db7467c <+325>: mov    rbx, rax
0x10db7467f <+328>: mov    rsi, qword ptr [rip + 0x94169a] ; "toggleVisibility"
0x10db74686 <+335>: mov    rdi, rax
0x10db74689 <+338>: call   r14
```

可以推测这边是使用UIDebuggingInformationOverlay的toggleVisibility进行展示，同样的，对UIDebuggingInformationOverlay的overlay以及toggleVisibility进行反编译

> 注：
>
> ``` assembly
> 0x10db74545 <+14>:  mov    rsi, qword ptr [rip + 0x900eac] ; "state"
> 0x10db7454c <+21>:  mov    rdi, rdx
> 0x10db7454f <+24>:  call   qword ptr [rip + 0x4e4f5b] ; (void *)0x0000000109ef3640: objc_msgSend
> 0x10db74555 <+30>:  cmp    rax, 0x3
> ```
>
> 这边有个点要注意，验证点击状态的值位0x3才会继续走下去，根据官方文档状态为UIGestureRecognizerStateEnded

# overlay & toggleVisibility

同样`disassemble -n "+[UIDebuggingInformationOverlay overlay]"`

输出

``` assembly
UIKitCore`+[UIDebuggingInformationOverlay overlay]:
    0x10db74a34 <+0>:  push   rbp
    0x10db74a35 <+1>:  mov    rbp, rsp
    0x10db74a38 <+4>:  sub    rsp, 0x30
    0x10db74a3c <+8>:  mov    rax, qword ptr [rip + 0x4e40d5] ; (void *)0x000000010c3b7070: _NSConcreteStackBlock
    0x10db74a43 <+15>: mov    qword ptr [rbp - 0x28], rax
    0x10db74a47 <+19>: mov    eax, 0xc2000000
    0x10db74a4c <+24>: mov    qword ptr [rbp - 0x20], rax
    0x10db74a50 <+28>: lea    rax, [rip + 0x40]         ; __40+[UIDebuggingInformationOverlay overlay]_block_invoke
    0x10db74a57 <+35>: mov    qword ptr [rbp - 0x18], rax
    0x10db74a5b <+39>: lea    rax, [rip + 0x5063de]     ; __block_descriptor_40_e8__e5_v8@?0l
    0x10db74a62 <+46>: mov    qword ptr [rbp - 0x10], rax
    0x10db74a66 <+50>: mov    qword ptr [rbp - 0x8], rdi
    0x10db74a6a <+54>: cmp    qword ptr [rip + 0x9bdc86], -0x1 ; overlay.__overlay + 7
    0x10db74a72 <+62>: jne    0x10db74a85               ; <+81>
    0x10db74a74 <+64>: mov    rdi, qword ptr [rip + 0x9bdc75] ; overlay.__overlay
    0x10db74a7b <+71>: add    rsp, 0x30
    0x10db74a7f <+75>: pop    rbp
    0x10db74a80 <+76>: jmp    0x10dd5d752               ; symbol stub for: objc_retainAutoreleaseReturnValue
    0x10db74a85 <+81>: lea    rdi, [rip + 0x9bdc6c]     ; overlay.onceToken
    0x10db74a8c <+88>: lea    rsi, [rbp - 0x28]
    0x10db74a90 <+92>: call   0x10dd5d3ec               ; symbol stub for: dispatch_once
    0x10db74a95 <+97>: jmp    0x10db74a74               ; <+64>
```

可以看到`0x10db74a6a <+54>: cmp    qword ptr [rip + 0x9bdc86], -0x1 ; overlay.__overlay + 7`以及`call   0x10dd5d3ec               ; symbol stub for: dispatch_once`，明显这是一个dispatch_once，之后进入dispatch_once调用，打断点调试`b +[UIDebuggingInformationOverlay overlay]`，中间过程就不赘序了，最后会走到

``` assembly
UIKitCore`-[UIDebuggingInformationOverlay init]:
    0x10db747f0 <+0>:   push   rbp
    0x10db747f1 <+1>:   mov    rbp, rsp
    0x10db747f4 <+4>:   push   r14
    0x10db747f6 <+6>:   push   rbx
    0x10db747f7 <+7>:   sub    rsp, 0x10
    0x10db747fb <+11>:  mov    rbx, rdi
    0x10db747fe <+14>:  cmp    qword ptr [rip + 0x9bdee2], -0x1 ; UIDebuggingOverlayIsEnabled.__overlayIsEnabled + 7
    0x10db74806 <+22>:  jne    0x10db74872               ; <+130>
    0x10db74808 <+24>:  cmp    byte ptr [rip + 0x9bded1], 0x0 ; mainHandler.onceToken + 7
    0x10db7480f <+31>:  je     0x10db7485a               ; <+106>
    0x10db74811 <+33>:  lea    rdi, [rbp - 0x20]
    0x10db74815 <+37>:  mov    qword ptr [rdi], rbx
    0x10db74818 <+40>:  mov    rax, qword ptr [rip + 0x953131] ; (void *)0x000000010e5147a8: UIDebuggingInformationOverlay
    0x10db7481f <+47>:  mov    qword ptr [rdi + 0x8], rax
    0x10db74823 <+51>:  mov    rsi, qword ptr [rip + 0x900576] ; "init"
    0x10db7482a <+58>:  call   0x10dd5d734               ; symbol stub for: objc_msgSendSuper2
    0x10db7482f <+63>:  mov    rbx, rax
    0x10db74832 <+66>:  test   rax, rax
    0x10db74835 <+69>:  je     0x10db74849               ; <+89>
    0x10db74837 <+71>:  mov    rsi, qword ptr [rip + 0x91dad2] ; "_setWindowControlsStatusBarOrientation:"
    0x10db7483e <+78>:  xor    edx, edx
    0x10db74840 <+80>:  mov    rdi, rbx
    0x10db74843 <+83>:  call   qword ptr [rip + 0x4e4c67] ; (void *)0x0000000109ef3640: objc_msgSend
    0x10db74849 <+89>:  mov    rdi, rbx
    0x10db7484c <+92>:  call   qword ptr [rip + 0x4e4c6e] ; (void *)0x0000000109ef0920: objc_retain
    0x10db74852 <+98>:  mov    rbx, rax
    0x10db74855 <+101>: mov    r14, rax
    0x10db74858 <+104>: jmp    0x10db7485d               ; <+109>
    0x10db7485a <+106>: xor    r14d, r14d
    0x10db7485d <+109>: mov    rdi, rbx
    0x10db74860 <+112>: call   qword ptr [rip + 0x4e4c52] ; (void *)0x0000000109ef0990: objc_release
    0x10db74866 <+118>: mov    rax, r14
    0x10db74869 <+121>: add    rsp, 0x10
    0x10db7486d <+125>: pop    rbx
    0x10db7486e <+126>: pop    r14
    0x10db74870 <+128>: pop    rbp
    0x10db74871 <+129>: ret    
    0x10db74872 <+130>: lea    rdi, [rip + 0x9bde6f]     ; UIDebuggingOverlayIsEnabled.onceToken
    0x10db74879 <+137>: lea    rsi, [rip + 0x50f9e0]     ; __block_literal_global.104
    0x10db74880 <+144>: call   0x10dd5d3ec               ; symbol stub for: dispatch_once
    0x10db74885 <+149>: jmp    0x10db74808               ; <+24>
```

这块的伪代码出来了

``` objective-c
+ (instancetype)overlay {
    static id instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[UIDebuggingInformationOverlay alloc] init];
    });
    return instance;
}
```

然后是init，首先看`0x10db747fe <+14>:  cmp    qword ptr [rip + 0x9bdee2], -0x1 ; UIDebuggingOverlayIsEnabled.__overlayIsEnabled + 7`，将`[rip + 0x9bdee2]`与-1进行比较，如果不相等则跳转到0x10db74872（`jne    0x10db74872`），0x10db74872处是

``` assembly
0x10db74872 <+130>: lea    rdi, [rip + 0x9bde6f]     ; UIDebuggingOverlayIsEnabled.onceToken
0x10db74879 <+137>: lea    rsi, [rip + 0x50f9e0]     ; __block_literal_global.104
0x10db74880 <+144>: call   0x10dd5d3ec               ; symbol stub for: dispatch_once
```

同样是一个` dispatch_once`，然后是

``` assembly
0x10db74808 <+24>:  cmp    byte ptr [rip + 0x9bded1], 0x0 ; mainHandler.onceToken + 7
0x10db7480f <+31>:  je     0x10db7485a
```

查看mainHandler.onceToken + 7处是什么属性

```
image lookup -a 0x10E5326E0	// [rip + 0x9bded1]，rip为指令寄存器，此时的址为0x10db7480f
```

输出

```
Address: UIKitCore[0x00000000016876e0] (UIKitCore.__DATA.__bss + 28384)
Summary: UIKitCore`UIDebuggingOverlayIsEnabled.__overlayIsEnabled
```

到这里就很明朗了，init方法里面调用了__overlayIsEnabled，如果为No，跳转到0x10db7485a（`0x10db7485a <+106>: xor  r14d, r14d`），而下面可见0x10db74866 <+118>: mov  rax, r14，等于把返回值置为nil

伪代码

``` objective-c
- (void)init {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        __overlayIsEnabled = UIDebuggingOverlayIsEnabled.getEnable;
    });
  	
    if (!__overlayIsEnabled) {
      return nil;
    }
  	
  	if(self = [super init]) {
        [self _setWindowControlsStatusBarOrientation:No]	// 0x10db7483e <+78>:  xor    edx, edx  把rdx寄存器低32位置0
    }
      
    return self;
}
```

也就是说我们在代码中hook init函数，然后把我们自己的手势传递给UIDebuggingInformationOverlayInvokeGestureHandler的_setWindowControlsStatusBarOrientation:进行调用就可。

效果：
![simulator_screenshot_6900311D-6B1B-4D9A-AAB2-C9C39DBFA5D9](https://user-images.githubusercontent.com/22512175/123213995-47b63200-d4f9-11eb-902a-c62f54ac4190.png)

> 注：上述说先不关心`_UIGetDebuggingOverlayEnabled`，是因为在iOS14下进行反编译的时候发现`prepareDebuggingOverlay`函数几乎不做事了，在iOS12下通过`mem write`命令直接改写内存可以实现效果，即调用prepareDebuggingOverlay就行，iOS14不行。

> 以上汇编在Mac x86_64架构上进行

# demo

  
