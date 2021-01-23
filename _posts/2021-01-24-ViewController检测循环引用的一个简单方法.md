---
layout: post
title:  ViewController检测循环引用的一个简单方法
date:   2021-1-24 00:13:27 +0800
categories: jekyll update
---

当检测 View Controller 是否被循环引用，没有被释放掉，通常在 `deinit` 方法中输出一段文字，来检测 VC 是否被释放掉。

```
deinit {
    print("deinit \(self)")
}
```

另一种方法是通过添加断点的方式：使用 Symbolic Breakpoint 检测 VC 有没有释放掉。

1. 在添加全局断点，即 **Breakpoint Navigator** (Menu View > Navigators > Show Breakpoint Navigator or ⌘ - command + 8) 


    ![Breakpoint Navigator](https://d33wubrfki0l68.cloudfront.net/16ddc6db69a9121cf2e5484be796a49c069af5f3/f9834/images/debug-deinit-breakpoint-add.png)


2. 点击 + 选择 **Symbolic Breakpoint**，或者 通过菜单栏 Menu Debug > Breakpoints > Create Symbolic Breakpoint 


    ![New Symbolic Breakpoint](https://d33wubrfki0l68.cloudfront.net/939e37ca4eacdc2f2bed61facaed3e6ae8e4e7f6/2097b/images/debug-deinit-breakpoint-new.png)


3. 设置 `Symbol` 的值为 `-[UIViewController dealloc]` 


    ![](https://d33wubrfki0l68.cloudfront.net/9dfec756012efde4e9edddfd49a819656eeaeba9/3d7be/images/debug-deinit-breakpoint-symbol.png)


4. 点击 **Add Action** ，设置声音（**Sound**） 


     ![](https://d33wubrfki0l68.cloudfront.net/1d4a6297ccb4588c1065e1b8d68983237c17ce2b/2c648/images/debug-deinit-breakpoint-sound.png)


5. 点击 + 添加另一个 Action
6. 另一个 **Action** 选择 **Log Message**，在下面选择 **Log message to console**，设置想输出的信息，例如 `--- dealloc @(id)[$arg1 description]@` 


    ![](https://d33wubrfki0l68.cloudfront.net/65fea51ea9b1f2485cc292b5e7b517b87631a9c8/15552/images/debug-deinit-breakpoint-log.png)


7. 最后勾选上 **Automatically continue after evaluating actions** ，防止在调试的时候因为执行 `deinit` 或者 `dealloc` 方法而暂停。 


    ![](https://d33wubrfki0l68.cloudfront.net/02787f5f5a4704667b633bcc88184c50672878ec/b4396/images/debug-deinit-breakpoint-options.png)

