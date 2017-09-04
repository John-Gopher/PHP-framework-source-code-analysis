###                                       php文件锁

bool flock ( int handle, int operation [, int &wouldblock] );
flock() 操作的 handle 必须是一个已经打开的文件指针。operation 可以是以下值之一：

要取得共享锁定（读取程序），将 operation 设为 LOCK_SH（PHP 4.0.1 以前的版本设置为 1）
要取得独占锁定（写入程序），将 operation 设为 LOCK_EX（PHP 4.0.1 以前的版本中设置为 2）
要释放锁定（无论共享或独占），将 operation 设为 LOCK_UN（PHP 4.0.1 以前的版本中设置为 3）
如果你不希望 flock() 在锁定时堵塞，则给 operation 加上 LOCK_NB（PHP 4.0.1 以前的版本中设置为 4）
建两个文件
(1) a.php

```php
$file = "temp.txt";    
$fp = fopen($file , 'w');    
if(flock($fp , LOCK_EX)){    
     fwrite($fp , "abc\n");    
     sleep(10);    
     fwrite($fp , "123\n");    
    flock($fp , LOCK_UN);    
}    
fclose($fp);   
```


(2) b.php

```php
$file = "temp.txt";    
$fp = fopen($file , 'r');    
echo fread($fp , 100);    
fclose($fp); 
```


运行 a.php 后，马上运行 b.php ，可以看到输出：
abc
等 a.php 运行完后运行 b.php ，可以看到输出：
abc
123
显然，当 a.php 写文件时数据太大，导致时间比较长时，这时 b.php 读取数据不完整

修改 b.php 为：

```php
$file = "temp.txt";    
$fp = fopen($file , 'r');    
if(flock($fp , LOCK_EX)){    
    echo fread($fp , 100);    
    flock($fp , LOCK_UN);    
} else{    
    echo "Lock file failed...\n";    
}    
fclose($fp); 
```


运行 a.php 后，马上运行 b.php ，可以发现 b.php 会等到 a.php 运行完成后(即 10 秒后)才显示：
abc
123
读取数据完整，但时间过长，他要等待写锁释放。

修改 b.php 为：

```php
$file = "temp.txt";    
$fp = fopen($file , 'r');    
if(flock($fp , LOCK_SH | LOCK_NB)){    
    echo fread($fp , 100);    
    flock($fp , LOCK_UN);    
} else{    
    echo "Lock file failed...\n";    
}    
fclose($fp);   
```


fclose($fp);   
运行 a.php 后，马上运行 b.php ，可以看到输出：
Lock file failed…
证明可以返回锁文件失败状态，而不是向上面一样要等很久。

结论：
建议作文件缓存时，选好相关的锁，不然可能导致读取数据不完整，或重复写入数据。
file_get_contents 好像选择不了锁，不知道他默认用的什么锁，反正和不锁得到的输出一样，是不完整的数据。
我是要做文件缓存，所以只需要知道是否有写锁存在即可，有的话就查数据库就可以了。
测试环境：Linux(Ubuntu 6) , PHP 5.1.2 , Apache 2

再转：

文件锁有两种：共享锁和排他锁，也就是读锁(LOCK_SH)和写锁(LOCK_EX) 
文件的锁一般这么使用：


```php
$fp = fopen("filename", "a");   

flock($fp, LOCK_SH) or die("lock error")   

str = fread(fp, 1024);   

flock($fp, LOCK_UN);   

fclose($fp);  

```



注意fwrite之后，文件立即就被更新了，而不是等fwrite然后fclose之后文件才会更新，这个可以通过在fwrite之后fclose之前读取这个文件进行检查 

但是什么时候使用lock_ex什么时候使用lock_sh呢？ 

读的时候： 
如果不想出现dirty数据，那么最好使用lock_sh共享锁。可以考虑以下三种情况： 
1. 如果读的时候没有加共享锁，那么其他程序要写的话（不管这个写是加锁还是不加锁）都会立即写成功。如果正好读了一半，然后被其他程序给写了，那么读的后一半就有可能跟前一半对不上（前一半是修改前的，后一半是修改后的） 
2. 如果读的时候加上了共享锁（因为只是读，没有必要使用排他锁），这个时候，其他程序开始写，这个写程序没有使用锁，那么写程序会直接修改这个文件，也会导致前面一样的问题 
3. 最理想的情况是，读的时候加锁(lock_sh),写的时候也进行加锁(lock_ex),这样写程序会等着读程序完成之后才进行操作，而不会出现贸然操作的情况 

写的时候： 
如果多个写程序不加锁同时对文件进行操作，那么最后的数据有可能一部分是a程序写的，一部分是b程序写的 
如果写的时候加锁了，这个时候有其他的程序来读，那么他会读到什么东西呢？ 
1. 如果读程序没有申请共享锁，那么他会读到dirty的数据。比如写程序要写a,b,c三部分，写完a,这时候读读到的是a，继续写b，这时候读读到的是ab，然后写c，这时候读到的是abc. 
2. 如果读程序在之前申请了共享锁，那么读程序会等写程序将abc写完并释放锁之后才进行读。 
