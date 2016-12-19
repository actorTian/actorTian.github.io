---
title: php代码分析的前奏--xhprof
date: 2016-12-18 09:02:27
tags: [xhprof,php]
category: 技术
---

代码优化，程序员本职和进阶的必须工作。前天在公司测试服务器部署了xhprfo,今天记录下。

### 什么是xhprof?
XHProf是一个分层PHP性能分析工具。它报告函数级别的请求次数和各种指标，包括阻塞时间，CPU时间和内存使用情况。一个函数的开销，可细分成调用者和被调用者的开销，XHProf数据收集阶段，它记录调用次数的追踪和包容性的指标弧在动态callgraph的一个程序。

### 为什么是xhprof?
轻量且配置灵活。

<!-- more -->

### Xhprof安装
#### 第一步
```vim
wget http://pecl.php.net/get/xhprof-0.9.4.tgz  
tar zxf xhprof-0.9.2.tgz  
cd xhprof-0.9.2  
cp -r xhprof_html xhprof_lib /你的web目录路径/xhprof/  （此处目的是建立数据分析目录，可将此目录配置成虚拟主机访问）  
cd extension/  
/usr/local/webserver/php/bin/phpize  
../configure  --with-php-config=/usr/local/webserver/php/bin/php-config   (php-config的路径)  
make && make install 
```
#### 第二步
php.ini中添加
```vim
extension=xhprof.so
xhprof.output_dir=/www/logs/xhprof  #数据输出目录，没有需建立
```
#### 第三步
重启nginx、php

### Xhprof使用
```php
//头部：  
xhprof_enable();   
//xhprof_enable(XHPROF_FLAGS_NO_BUILTINS); 不记录内置的函数  
//xhprof_enable(XHPROF_FLAGS_CPU + XHPROF_FLAGS_MEMORY);  同时分析CPU和Mem的开销  
$xhprof_on = true;  
 
//生产环境可使用：  
if (mt_rand(1, 10000) == 1) {  
   xhprof_enable();  
   $xhprof_on = true;  
}

//尾部：  
if($xhprof_on){  
$xhprof_data = xhprof_disable();  
$xhprof_root = '/www/（xhprof的虚拟主机目录）/';  
include_once $xhprof_root."xhprof_lib/utils/xhprof_lib.php";   
include_once $xhprof_root."xhprof_lib/utils/xhprof_runs.php";   
$xhprof_runs = new XHProfRuns_Default();   
$run_id = $xhprof_runs->save_run($xhprof_data, "hx");  
echo '统计';  
$url = $xhprof_root."/xhprof_html/index.php?run=$run_id&source=xhprof_foo";
echo ''.$url.'';
} ```

### 改进：使全服务器所有站点可使用？
原理：利用php.ini中的auto_prepend_file 配置项实现全服站点文件注入。
Php.ini中 配置 auto_prepend_file = /alidata/www/xhprof/index.php（我们需要注入的文件）
然后注意针对apache和ngixn有所不同，可利用服务器配置文件方式进行文件引入，也可利用.htaccess文件方式。

#### 服务器方式：
```php
//nginx: fastcgi_param PHP_VALUE "auto_prepend_file=/alidata/www/xhprof/index.php";
// apache好像不需要处理了
重启服务器。
```
#### .htaccess方式：
在需要引入xhprof的文件夹中创建.htaccess文件，并设置如下内容
```vim
php_value auto_prepend_file "/home/fdipzone/header.php"  
php_value auto_append_file "/home/fdipzone/footer.php"  
```
这样在指定.htaccess的文件夹内的页面文件才会加载 /home/fdipzone/header.php 与 /home/fdipzone/footer.php，其他页面文件不受影响。
使用.htaccess设置，比较灵活，不需要重启服务器，也不需要管理员权限，唯一缺点是目录中每个被读取和被解释的文件每次都要进行处理，而不是在启动时处理一次，所以性能会有所降低。

### xhprof参数字段含义解析：
Function Name：方法名称。
Calls：方法被调用的次数。
Calls%：方法调用次数在同级方法总数调用次数中所占的百分比。
Incl.Wall Time(microsec)：方法执行花费的时间，包括子方法的执行时间。（单位：微秒）
IWall%：方法执行花费的时间百分比。
Excl. Wall Time(microsec)：方法本身执行花费的时间，不包括子方法的执行时间。（单位：微秒）
EWall%：方法本身执行花费的时间百分比。
Incl. CPU(microsecs)：方法执行花费的CPU时间，包括子方法的执行时间。（单位：微秒）
ICpu%：方法执行花费的CPU时间百分比。
Excl. CPU(microsec)：方法本身执行花费的CPU时间，不包括子方法的执行时间。（单位：微秒）
ECPU%：方法本身执行花费的CPU时间百分比。
Incl.MemUse(bytes)：方法执行占用的内存，包括子方法执行占用的内存。（单位：字节）
IMemUse%：方法执行占用的内存百分比。
Excl.MemUse(bytes)：方法本身执行占用的内存，不包括子方法执行占用的内存。（单位：字节）
EMemUse%：方法本身执行占用的内存百分比。
Incl.PeakMemUse(bytes)：Incl.MemUse峰值。（单位：字节）
IPeakMemUse%：Incl.MemUse峰值百分比。
Excl.PeakMemUse(bytes)：Excl.MemUse峰值。单位：（字节）
EPeakMemUse%：Excl.MemUse峰值百分比。

### 扩展
#### php_admin_value 与 php_value 的区别
相同的地方是：这两个命令都是用来在Apache服务器中针对不同的虚拟主机、目录设置不同的php选项的。
不同的地方是：php_admin_value(php_admin_flag)命令只能用在apache的httpd.conf文件中，而 php_value(php_flag)则是用在.htaccess文件中的。
#### fastcgi_param
这个命令是设置fastcgi请求中的参数，具体设置的东西可以在$_SERVER中获取到



