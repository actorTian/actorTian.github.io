---
title: yii2的DI容器解读
date: 2017-01-11 21:13:24
tags: [php,yii2]
category: 技术
---

为了降低代码耦合程度，提高项目的可维护性，Yii采用多许多当下最流行又相对成熟的设计模式，包括了依赖注入（Denpdency Injection, DI）和服务定位器（Service Locator）两种模式。

<!-- more -->

### 依赖注入
首先来讲讲DI，在Web应用中，很常见的是使用各种第三方Web Service实现特定的功能，比如发送邮件、推送微博等。 假设要实现当访客在博客上发表评论后，向博文的作者发送Email的功能。那么这里的发送邮件可能会用到各种各样的邮件发送组件，所以要灵活设计。依赖注入就是为了解决这个问题而生的，当然，DI也不是唯一解决问题的办法，毕竟条条大路通罗马。 
在Yii中使用DI解耦，有2种注入方式：构造函数注入、属性注入。
#### 构造函数注入示例
``` php
class Comment extend yii\db\ActiveRecord
{
    // 用于引用发送邮件的库
    private $_eMailSender;

    // 构造函数注入
    public function __construct($emailSender)
    {
        ...
        $this->_eMailSender = $emailSender;
        ...
    }

    // 当有新的评价，即 save() 方法被调用之后中，会触发以下方法
    public function afterInsert()
    {
        ...
        //
        $this->_eMailSender->send(...);
        ...
    }
}

// 实例化两种不同的邮件服务，当然，他们都实现了EmailSenderInterface
sender1 = new GmailSender();
sender2 = new MyEmailSender();

// 用构造函数将GmailSender注入
$comment1 = new Comment(sender1);
// 使用Gmail发送邮件
$comment1.save();

// 用构造函数将MyEmailSender注入
$comment2 = new Comment(sender2);
// 使用MyEmailSender发送邮件
$comment2.save();
```
#### 问题
依赖注入是不是很灵活？那么问题来了，一个Web应用的某一组件会依赖于若干单元，这些单元又有可能依赖于更低层级的单元， 从而形成依赖嵌套的情形。而且，这些依赖单元的实例化、注入过程的代码可能会比较长，前后关系也需要特别地注意，必须将被依赖的放在需要注入依赖的前面进行实例化。 这是高智商人群的天敌！！！如何改进？

#### 改进：DI容器
DI容器被设计出来了。他知道如何对对象及对象的所有依赖，和这些依赖的依赖，进行实例化和配置。

#### DI容器实例分析
``` php
namespace app\models;

use yii\base\Object;
use yii\db\Connection;

// 定义接口
interface UserFinderInterface
{
    function findUser();
}

// 定义类，实现接口
class UserFinder extends Object implements UserFinderInterface
{
    public $db;

    // 从构造函数看，这个类依赖于 Connection
    public function __construct(Connection $db, $config = [])
    {
        $this->db = $db;
        parent::__construct($config);
    }

    public function findUser()
    {
    }
}

class UserLister extends Object
{
    public $finder;

    // 从构造函数看，这个类依赖于 UserFinderInterface接口
    public function __construct(UserFinderInterface $finder, $config = [])
    {
        $this->finder = $finder;
        parent::__construct($config);
    }
}
```
依赖关系看，这里的UserLister类依赖于接口UserFinderInterface，而接口有一个实现就是 UserFinder 类，但这类又依赖于 Connection 。
那么，按照一般常规的作法，要实例化一个 UserLister 通常这么做:
``` php
$db = new \yii\db\Connection(['dsn' => '...']);
$finder = new UserFinder($db);
$lister = new UserLister($finder);
```
**DI容器怎么做：**
``` php
use yii\di\Container;

// 创建一个DI容器
$container = new Container;

// 为Connection指定一个数组作为依赖，当需要Connection的实例时，
// 使用这个数组进行创建
$container->set('yii\db\Connection', [
    'dsn' => '...',
]);

// 在需要使用接口 UserFinderInterface 时，采用UserFinder类实现
$container->set('app\models\UserFinderInterface', [
    'class' => 'app\models\UserFinder',
]);

// 为UserLister定义一个别名
$container->set('userLister', 'app\models\UserLister');

// 获取这个UserList的实例
$lister = $container->get('userLister');
```
**说明：**采用DI容器的办法，首先各 set() 语句没有前后关系的要求， set()只是写入特定的数据结构， 并未涉及具体依赖关系的解析。所以，前后关系不重要，先定义什么依赖，后定义什么依赖没有关系。

其次，上面根本没有在DI容器中定义UserFinder对于Connection的依赖。但是DI容器通过对 UserFinder 构造函数的分析，能了解到这个类会对 Connection 依赖。这个过程是自动的。

最后，上面只有一个 get() 看起来好像根本没有实例化其他如Connection单元一样，但事实上，DI容器已经安排好了一切。 在获取 userLister 之前， Connection和UserFinder 都会被自动实例化。 其中， Connection 是根据依赖定义中的配置数组进行实例化的。

至于DI容器内部执行原理，有兴趣同学可深入源码分析研究。


