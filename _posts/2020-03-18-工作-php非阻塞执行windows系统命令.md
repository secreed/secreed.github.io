---
type: 工作
title: php非阻塞执行windows系统命令
date: 2020-03-18
category: 探针

tags:
- php
- 工作
description: php非阻塞执行windows系统命令
---
>积硅步至千里--总有一天你会看到不一样的风景，当你坚持不懈含着泪水全力向前！
凌晨1点，静心梳理
TODO:完善python异步
## 参考
1. https://www.phpclasses.org/package/9674-PHP-Get-commands-output-without-waiting-to-finish.html
2. https://ourcodeworld.com/articles/read/207/how-to-execute-a-shell-command-using-php-without-await-for-the-result-asynchronous-in-linux-and-windows-environments
3. https://www.php.net/manual/en/function.exec.php
4. https://blog.csdn.net/weixin_37281289/article/details/99686926

## 非阻塞调用分析

并发IO问题一直是服务器端编程中的技术难题，从最早的同步阻塞直接Fork进程，到Worker进程池/线程池，到现在的异步IO、协程。阻塞和非阻塞关注的是程序在等待调用结果（消息，返回值）时的状态。现在网络的日益发展，对响应时间和处理并发的要求越来越高，所以异步非阻塞的需求也越来越多。

### 非阻塞的优点：
1. 非阻塞调用指在不能立刻得到结果之前，该调用不会阻塞当前线程，能及时返回响应，减少响应时间；
2. 高并发，同步阻塞IO模型的并发能力依赖于进程/线程数量，响应时间的下降可以带来更多的并发能力。
### 非阻塞的缺点：
1. **启动大量进程会带来额外的进程调度消耗**。非阻塞模式下，如果大量线程已经返回响应而仍然在执行计算操作，会使CPU利用率不可控的增高；
2. 这种模型严重依赖进程的数量解决并发问题，一个客户端连接就需要占用一个进程，工作进程的数量有多少，并发处理能力就有多少。**操作系统可以创建的进程数量是有限的**;
3. 过多的异步和多线程模型会造成编码困难或线程混乱，如**出现大量僵尸进程**等


## 官网的两个解决方案

1. popen()

```php
    function execInBackground($cmd) {

    if (substr(php_uname(), 0, 7) == "Windows"){
        //echo "haha-----------------";
        $log = [];
	    $flag = "1";
        //exec($cmd . " > NUL &", $log, $status);
        pclose(popen("start /B ". $cmd, "r"));//"start /B ". 
    }
    else {
        $log = [];
	    $flag = "1";
        exec($cmd . " > /dev/null &", $log, $status);
    }
    
}
```

2. system()

```php
    function executeAsyncShellCommand($comando = null){
        if(!$comando){
            throw new Exception("No command given");
        }
        // If windows, else
        if (strtoupper(substr(PHP_OS, 0, 3)) === 'WIN') {
            //echo "system---------------------";
            system($comando." > NUL");
        }else{
            shell_exec("/usr/bin/nohup ".$comando." >/dev/null 2>&1 &");
        }
    }
```

## 方案分析

经测试，网上各种方法，均为成功（win10 wamp搭的php web服务），最后通过调用python程序（python的异步操作）实现。

### 调用python，实现非阻塞响应

subprocess


### 提前结束会话（请求）  
  
1. For PHP using FastCGI and PHP_FPM: fastcgi_finish_request() 
2. PHP using mod_php (standard Apache): flush

#### FastCGI的非阻塞方法：fastcgi_finish_request() 

**windows下测试失败**
[官方链接](https://www.php.net/manual/zh/function.fastcgi-finish-request.php)
正常脚本结束时php会自动调用session_write_close()函数, 而脚本在处理中的时候占用者session锁,对于后续请求来说是阻塞的.所以要尽快手动调用session_write_close()结束并保存session数据. 这对于其他有竞争锁情况同样适用,没有用了要尽快释放

1. 条件：
在PHP5.3.3版本之后，不管是Nginx还是Apache服务器，只要运行在FastCGI模式下，均可使用该方法，官方解释的作用是冲刷(flush)所有响应的数据给客户端。
一般模式下(如Apache, Nginx, FastCGI(直接使用fastcgi_finish_request()更方便等), 提前输出内容, 结束会话.
2. ` boolean fastcgi_finish_request ( void )`
此函数冲刷(flush)所有响应的数据给客户端并结束请求。 这使得客户端结束连接后，需要大量时间运行的任务能够继续运行。
用法：可以在读写大文件、循环更新数据库等不影响结果的操作之前，执行该函数，把结果返回给客户端，php会继续执行下面的逻辑而不影响客户端的响应时间。
3. 示例

```php
// your code here

fastcgi_finish_request();

// run other process without the client attached.
```

#### flush

**windows下测试失败**

1. 说明
刷新PHP程序的缓冲，而不论PHP执行在何种情况下（CGI ，web服务器等等）。该函数将当前为止程序的所有输出发送到用户的浏览器。
在以下情况中,**该方法失效**:无论哪个模式,gzip一定要关闭; window32下web服务不行;   [官方说明](https://www.php.net/manual/zh/function.flush.php)
* 个别web服务器程序，特别是Win32下的web服务器程序，在发送结果到浏览器之前，仍然会缓存脚本的输出，直到程序结束为止。
* 有些Apache的模块，比如mod_gzip，可能自己进行输出缓存，这将导致flush()函数产生的结果不会立即被发送到客户端浏览器。

2. 原理
* buffer是一个内存地址空间,Linux系统默认大小一般为4096(1kb),即一个内存页。主要用于存储速度不同步的设备或者优先级不同的 设备之间传办理数据的区域。通过buffer，可以使进程这间的相互等待变少。这里说一个通俗一点的例子，你打开文本编辑器编辑一个文件的时候，你每输入 一个字符，操作系统并不会立即把这个字符直接写入到磁盘，而是先写入到buffer，当写满了一个buffer的时候，才会把buffer中的数据写入磁 盘，当然当调用内核函数flush()的时候，强制要求把buffer中的脏数据写回磁盘。
* 当执行echo,print的时候，输出并没有立即通过tcp传给客户端浏览器显示, 而是将数据写入php buffer。php output_buffering机制，意味在tcp buffer之前，建立了一新的队列，数据必须经过该队列。当一个php buffer写满的时候，脚本进程会将php buffer中的输出数据交给系统内核交由tcp传给浏览器显示。所以，数据会依次写到这几个地方echo/print -> php buffer -> tcp buffer -> browser
3. ob_flush()
默认情况下，php buffer是开启的，而且该buffer默认值是4096，即1kb。你可以通过在php.ini配置文件中找到output_buffering配置.当echo,print等输出用户数据的时候，输出数据都会写入到php output_buffering中，直到output_buffering写满，会将这些数据通过tcp传送给浏览器显示。你也可以通过 ob_start()手动激活php output_buffering机制，使得即便输出超过了1kb数据，也不真的把数据交给tcp传给浏览器，因为ob_start()将php buffer空间设置到了足够大 。只有直到脚本结束，或者调用ob_end_flush函数，才会把数据发送给客户端浏览器。
4. flush() 与 ob_flush() 区别
* 在没有开启缓存时，脚本输出的内容都在服务器端处于等待输出的状态 ，flush()可以将等待输出的内容立即发送到客户端。

* 开启缓存后，脚本输出的内容存入了输出缓存中 ，这时没有处于等待输出状态的内容，你直接使用flush()不会向客户端发出任何内容。而 ob_flush()的作用就是将本来存在输出缓存中的内容取出来，设置为等待输出状态，但不会直接发送到客户端 ，这时你就需要先使用 ob_flush()再使用flush()，客户端才能立即获得脚本的输出。
5. ob_start
* 激活output_buffering机制。一旦激活，脚本输出不再直接出给浏览器，而是先暂时写入php buffer内存区域。
* php默认开启output_buffering机制，只不过，通过调用ob_start()函数据output_buffering值扩展到足够 大 。也可以指定$chunk_size来指定output_buffering的值。
* $chunk_size默认值是0,表示直到脚本运行结束，php buffer中的数据才会发送到浏览器。如果你设置了$chunk_size的大小 ，则表示只要buffer中数据长度达到了该值，就会将buffer中 的数据发送给浏览器。
6. ob_end_flush与ob_end_clean
* 这二个函数有点相似，都会关闭ouptu_buffering机制。但不同的是，ob_end_flush只是把php buffer中的数据冲(flush/send)到客户端浏览器，而ob_clean_clean将php bufeer中的数据清空(erase)，但不发送给客户端浏览器。

* ob_end_flush调用之前 ，php buffer中的数据依然存在，ob_get_contents()依然可以获取php buffer中的数据拷贝。

* 而ob_end_flush()调用之后 ob_get_contents()取到的是空字符串，同时浏览器也接收不到输出，即没有任何输出。
7. 示例
示例1
```php
//1. stack overflow
ob_start();
header("Connection: close\r\n"); 
header('Content-Encoding: none\r\n');

// your code here

$size = ob_get_length();
header("Content-Length: ". $size . "\r\n"); 
// send info immediately and close connection
ob_end_flush();
flush();

// run other process without the client attached.
```
示例2
```php
//适用于大多数运行模式(不包括命令行模式)
set_time_limit(0);  //设置不限执行时间
ignore_user_abort(true);  //忽略客户端中断
//nginx等可能需要达到4k才会输出buffer,所有先输出一些空字符串
$str = str_repeat(' ', 65536);
$str .= '立即输出' . date('Y-m-d H:i:s');
#header('X-Accel-Buffering: no');   // 关闭加速缓冲, 在nginx模式需要开启此行
header("Content-Type: text/html;charset=utf-8");
header("Connection: close");//告诉浏览器不需要保持长连接
header('Content-Length: '. strlen($str));//告诉浏览器本次响应的数据大小只有上面的echo那么多
ob_end_flush();
ob_start();
echo $str;
ob_flush();
flush();
//至此,连接已经关闭. 但是进程还不会结束, 以下程序还能运行但不会输出
sleep(10);
file_put_contents('./log.txt', '10s后我写入log文本: 时间' . date('Y-m-d H:i:s'));

```

### 使用pcntl_fork()

1. 条件：
在PHP4.1.0版本之后都支持了该函数，使用前需要安装支持pcntl扩展并添加支持。<b>不支持windows系统</b>

在当前进程当前位置产生分支（子进程），这个子进程仅PID（进程号） 和PPID（父进程号）与其父进程不同。
2. `int pcntl_fork ( void )`
成功时，在父进程执行线程内返回产生的子进程的PID，在子进程执行线程内返回0；
失败时，在父进程上下文返回-1，不会创建子进程，并且会引发一个PHP错误。
3. 示例

```php
$pid = pcntl_fork();
//父进程和子进程都会执行下面的代码
if($pid == -1){
    //错误处理：创建子进程失败时，返回-1
    die("could not fork");
}else if($pid){
    //父进程会得到子进程的进程号，所里该逻辑段是父进程执行的逻辑
    pcntl_wait($status);//等待子进程中断，防止子进程成为僵尸进程
    echo "the parent thread:".$status;
}else{
    //子进程得到的$pid进程号为0，所里该逻辑段是子进程执行的逻辑
    echo "the child thread";
}
exit();
```

### 调用系统命令

极端的情况下，可以调用系统命令，可以将数据传给后台任务执行，个人感觉不是很高效。
**经测试，windows下未成功，linux下成功**
参考 [官网的两个解决方案](#官网的两个解决方案)
调用系统命令，将输出重定向到文件中，可以起到异步的效果

### 开启子进程 popen  proc_open()

1. 说明
函数通过创建一个管道，调用fork()产生一个子进程
2. `popen(command,mode)`
* command:执行的命令
* mode:链接模式，r == 只读 ； w == 只写（打开并清空已有文件或创建一个新文件）
* 返回值
>返回一个和 fopen() 所返回的相同的文件指针，只不过它是单向的（只能用于读或写）并且必须用 pclose() 来关闭。此指针可以用于 fgets()，fgetss() 和 fwrite()。 当模式为 'r'，返回的文件指针等于命令的 STDOUT，当模式为 'w'，返回的文件指针等于命令的 STDIN。
如果出错返回 FALSE。

 

3. 示例

```php
//例子1 (不等待子进程返回结果直接结束父脚本)
<?php
ini_set('date.timezone', 'Asia/shanghai');
echo "父脚本开始\n";
echo date('Y-m-d H:i:s') . "\n";
//&: 转入后台运行; nohup:不挂断地运行命令, 防止当前的终端窗口被关闭后导致进程结束
$cmd = 'nohup php cmd.php &';
pclose(popen($cmd, 'r'));//开启一个子进程后马上关闭, 子进程进入后台处理耗时的处理
echo "父脚本结束";

//例子2 (等待子进程返回结果在结束父脚本)
<?php
ini_set('date.timezone', 'Asia/shanghai');
echo "父进程开始\n";
echo date('Y-m-d H:i:s') . "\n";
$cmd1 = 'nohup php cmd.php &';
$cmd2 = 'nohup php cmd1.php &';     
//执行耗时异步任务1
$p1 = popen($cmd1, 'r');
//执行耗时异步任务2, 与任务1是并行运行的
$p2 = popen($cmd2, 'r');
register_shutdown_function(function () use ($p1, $p2) {
    $res1 = stream_get_contents($p1);
    echo $res1;
    $res2 = stream_get_contents($p2);
    echo $res2;
    pclose($p1);
    pclose($p2);
    echo "父脚本结束" . date('Y-m-d H:i:s') . "\n";
});

```

**proc_open() 该函数有更强的控制程序执行的能力, 可以双向(读又写)**

### 开启子进程的网址

不等待返回结果(说白了就是发个请求唤起耗时的子脚本, 而不等待这个耗时子脚本返回结果)

1. fsockopen打开一个网络连接或者一个Unix套接字连接, 忽略返回结果(不等待返回结果)
2. curl 设置超时时间为1s, 忽略返回结果(不等待返回结果,直接超时,但最短要1s)

#### 使用 fsockopen() + stream_set_blocking()方法

使用 fsockopen() 打开一个网络连接或者一个Unix套接字连接，再用 stream_set_blocking() 非阻塞模式请求：

```php
$fp = fsockopen("www.example.com", 80, $errno, $errstr, 30);
if (!$fp) {
    die('error fsockopen');
}
// 转换到非阻塞模式
stream_set_blocking($fp, 0);

$http = "GET /save.php  / HTTP/1.1\r\n";
$http .= "Host: www.example.com\r\n";
$http .= "Connection: Close\r\n\r\n";

fwrite($fp, $http);
fclose($fp);
```

#### 使用 cURL

cURL除了我们通常使用的curl_init来初始化和发送post和get请求之外，还可以使用curl_multi_init()方法来实现异步请求，其原理是使用系统的select这个多路I/O复用机制来异步发送请求。

```php
$mh = curl_multi_init();
$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, "http://localhost/");
curl_multi_add_handle($mh, $ch);
curl_multi_exec($mh, $active);

curl_close($ch);
curl_multi_remove_handle($mh, $ch);
curl_multi_close($mh);

echo "End\n";
```

### 使用扩展：Gearman/Swoole 扩展

Gearman 是一个具有 php 扩展的分布式异步处理框架，能处理大批量异步任务。
Swoole 最近很火，有很多异步方法，使用简单。
>swoole框架：fpm里，通过swoole_client把url发送到swoole的server，swoole_server天然支持并行请求，把汇总的结果返回到fpm；
这也是当下PHP最火的异步多线程框架，可以了解一下[韩天峰的文章](http://rango.swoole.com/)

### 使用缓存和队列
使用redis等缓存、队列，将数据写入缓存，使用后台计划任务实现数据异步处理。
这个方法在常见的大流量架构中应该很常见吧

