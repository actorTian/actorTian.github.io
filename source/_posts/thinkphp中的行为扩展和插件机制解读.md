---
title: thinkphp中的行为扩展和插件机制解读
date: 2016-06-08 15:58:11
tags: [thinkphp,php]
category: 技术
---

在立即行为扩展和插件机制之前必须先来了解另外一个知识点。
### 什么是钩子？
就像我们的房间，柜子上或者门上总有那么一个钩子，它的作用很明显，能够挂东西。我们可以挂上一个包包，或者一件衣服，或者一条耳机线。当你不想要某个挂件、衣服时，取下来即可。并不会破坏原有的袋子或者衣服的功能。
你挂与不挂，钩子就在那里。
回归到我们的程序，里面的钩子也是同样的理解。我们在程序中放了一个钩子，可以挂上某些轻量级单一的功能。例如加个统计，或者发个邮件，或者挂上一个编辑器。再回归正题，献上代码。

<!-- more -->

### ThinkPHP中的行为
加载标签与类之间的对应关系
``` php
// 加载模式行为定义
if(isset($mode['tags'])) {
      Hook::import(is_array($mode['tags'])?$mode['tags']:include $mode['tags']);
}
```
在ThinkPHP/Mode/common.php中tags标签中定义
``` php
'tags'  =>  array(
        'app_begin'     =>  array(
            'Behavior\ReadHtmlCache', // 读取静态缓存
        ),
        'app_end'       =>  array(
            'Behavior\ShowPageTrace', // 页面Trace显示
        ),
        'view_parse'    =>  array(
            'Behavior\ParseTemplate', // 模板解析 支持PHP、内置模板引擎和第三方模板引擎
        ),
        'template_filter'=> array(
            'Behavior\ContentReplace', // 模板输出替换
        ),
        'view_filter'   =>  array(
            'Behavior\WriteHtmlCache', // 写入静态缓存
        ),
    )
```
重头戏就在这里了，先说一下tp行为的执行入口是 run()方法，触发钩子时会自动执行行为类里的run()方法。
``` php
  /**App.class.php中程序执行调用
     * 运行应用实例 入口文件使用的快捷方法
     * @access public
     * @return void
     */
    static public function run() {
        // 应用初始化标签
        Hook::listen('app_init');
        App::init();
        // 应用开始标签
        Hook::listen('app_begin'); //我们就分析这个了，Hook类（负责实现行为和插件的排头兵）监听了app_begein行为。通过common.php中的tags我们确定了是'Behavior\ReadHtmlCache'类，程序找到它之后就会执行run()方法。
        // Session初始化
        if(!IS_CLI){
            session(C('SESSION_OPTIONS'));
        }
        // 记录应用初始化时间
        G('initTime');
        App::exec();
        // 应用结束标签
        Hook::listen('app_end');
        return ;
    }
```
通过钩子+行为，我们发现要扩展程序的行为很简单，只需要扩展相应的行为，和修改tags配置就好了。

### 那什么又是插件？
插件是用于扩展系统的功能的一些独立“组件”。相比于行为（只有一个run()方法入口），插件类中可有多个入口。例如可以附上安装，卸载，启用，禁用等方法入口。
就像我们在文章编辑中放了个编辑器钩子，我们可以通过这个这个钩子可以挂上ueditor,kindeditor。


