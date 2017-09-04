#### index.php


1. 根据$_SERVER['CI_ENV']定义ENVIRONMENT常量，默认是开发环境
1. 根据环境变量确定错误屏蔽级别：开发环境开启所有错误，测试环境和生产环境时，如果php版本小于5.3，
1. 只需要去掉notice、strict和user_notice。当php版本大于等于5.3的时候还要在以上基础上去掉deprecated和user_deprecated
1. 定义system文件所在目录为常量BASEPATH
1. 定义FCPATH为内部控制器目录
1. 定义APPPATH应用目录
1. 定义VIEWPATH为内部视图目录
1. 定义APPLOGPATH为内部日志目录
1. 加载CodeIgniter.php文件



###### 总结：
1.realpath返回的绝对路径最后面是不带反斜杠的

2.pathinfo函数经常用，但是一直记不住，所以在这里mark一下。

```
参数设置和对应的返回值:
PATHINFO_DIRNAME - 只返回 dirname,如/data/www/test.txt，则返回/data/www,注意：不带斜杠！
PATHINFO_BASENAME - 只返回 basename，如"test.txt"
PATHINFO_EXTENSION - 只返回 extension，如"txt"
```


3.

```
error_reporting(-1);
ini_set('display_errors', 1);
ini_set('display_errors', 0);

if (version_compare(PHP_VERSION, '5.3', '>='))
{
    error_reporting(E_ALL & ~E_NOTICE & ~E_DEPRECATED & ~E_STRICT & ~E_USER_NOTICE & ~E_USER_DEPRECATED);
}
else
{   
    error_reporting(E_ALL & ~E_NOTICE & ~E_STRICT & ~E_USER_NOTICE);
}
```


- ==使用 A ^ B  : A中有B就去掉, A中没有B就加上==
- ==使用 A & B  : 返回双方都包含的部分, 可用于判断某个或某些位是否存在==
- ==使用 A & ~B : A中有没有B都去掉B,不影响其他位==
- ==使用 A | B  : A中有没有B都加上B, 不影响其他位==
- ==PHP 6.0，E_STRICT 是 E_ALL 的一部分==



#### CodeIgniter.php



1. 加载常量定义文件constants.php
1. 加载常用的全局函数文件common.php
1. 在php5.3及其以下版本，为了安全考虑，清除EGPCS注册的全局变量。由于php.ini开启全局变量注册功能，将会导致EGPCS五个全局变量数组的内部键值映射到超级全局变量$GLOBAL。然而一开始这些键值并没有被安全过滤，所以在这里最好被清除。 注意：PHP 5.3.0 已经开始建议弃用这种做法，PHP 5.4.0，则彻底移除了。
1. 设置错误处理函数、异常处理函数、程序结束回调函数
1. 加载config.php文件，缓存配置数组$config。如果index文件中定义了控制器子类前缀$assign_to_config['subclass_prefix']，则替换缓存中相应配置项
1. Composer 是PHP的一个包依赖管理工具，在这里判断是否有必要加载“vendor/autoload.php”里面的包
1. 加载Benchmark基准检测类
1. 加载Hook钩子类，并执行hooks.php配置文件中定义的“pre_system”钩子函数（这种钩子不是重点，以后就忽略了不讲了）
1. 加载并初始化Config配置管理类，手动设置在index.php中定义的配置项
1. 根据配置设置默认编码以及mbstring、iconv相关函数的字符编码参数选项
1. 加载mbstring、password、hash、standard四个兼容处理库
1. 加载并实例化Utf-8、URI、Router、Output、Scurity、Input、Lang七个核心类库
1. 加载控制器基类Controller、加载应用目录下自定义控制器基类
1. 404路由处理
1. 实例化控制器
1. 调用并传参控制器对象方法
###### 总结：

1. 上面第5步中的缓存配置数组$config，是通过如下方式实现，不仔细看的话，容易忽略掉。CI把$config作为全局函数get_config的内部静态变量，而不是注册为一个全局变量。
```
function &get_config(Array $replace = array())
{
	static $config;
        ……  
        …… 
	if (empty($config))
	{
		$file_path = APPPATH.'config/config.php';
		$found = FALSE;
		if (file_exists($file_path))
		{
			$found = TRUE;
			require($file_path);
		}

	}
        ……  
        …… 
	return $config;
	
}
```

2.  下面是整个文件中最核心的代码，1到13步骤都是为了给它做铺垫的。
```
//404标识初始化为false
$e404 = FALSE;
//通过路由类实例获得待执行控制器的类名称
$class = ucfirst($RTR->class);
//通过路由类实例获得待执行的方法名称
$method = $RTR->method;
//如果类名称或者文件不存在，则标识为404
if (empty($class) OR ! file_exists(APPPATH.'controllers/'.$RTR->directory.$class.'.php'))
{
	$e404 = TRUE;
}
else
{
        //文件存在则加载相应的控制器文件
	require_once(APPPATH.'controllers/'.$RTR->directory.$class.'.php');
        //在不尝试自动加载的情况下判断该类是否已经声明，没有声明或者基类控制器中已经存在该方法（所以注意了！
        //子类和基类里面的方法名不能相同！），则标识为404
	if ( ! class_exists($class, FALSE) OR $method[0] === '_' OR method_exists('CI_Controller', $method))
	{
		$e404 = TRUE;
	}
	elseif (method_exists($class, '_remap'))
	{       //如果类中声明了_remap方法，则把方法调用重定向到_remap方法，注意：$URI->rsegments保存了从url中分解
	        //出来的路径，控制器，方法和参数等信息，从数组第三项开始保存的是参数
		$params = array($method, array_slice($URI->rsegments, 2));
		$method = '_remap';
	}
        //如果该控制器类中没有声明这个方法，则标识为404
	elseif ( ! in_array(strtolower($method), array_map('strtolower', get_class_methods($class)), TRUE))
	{
		$e404 = TRUE;
	}
}
//经过上面的的重重关卡之后，现在可以判断是否出现404了
if ($e404)
{
    //如果配置文件中自定义了404处理路由
	if ( ! empty($RTR->routes['404_override']))
	{
	    //从自定义404路由中解析出类名和方法名
		if (sscanf($RTR->routes['404_override'], '%[^/]/%s', $error_class, $error_method) !== 2)
		{
			$error_method = 'index';
		}

		$error_class = ucfirst($error_class);
                //在不尝试自动加载的情况下，该类没有被声明
		if ( ! class_exists($error_class, FALSE))
		{
		        //如果404处理控制器文件（url中没有定义本目录，则值为‘’，所以下面的写法能够兼容本目录存在
		        //和不存在两种情况）存在，则加载该文件，然后根据该类是否被声明，重新定义404标识
			if (file_exists(APPPATH.'controllers/'.$RTR->directory.$error_class.'.php'))
			{
				require_once(APPPATH.'controllers/'.$RTR->directory.$error_class.'.php');
				$e404 = ! class_exists($error_class, FALSE);
			}
			// 本目录下面不存在该控制器文件，则做一下最后的努力，判断父目录是否存在该控制器文件，
			//然后根据是否声明了该类，重置404标识
			elseif ( ! empty($RTR->directory) && file_exists(APPPATH.'controllers/'.$error_class.'.php'))
			{
				require_once(APPPATH.'controllers/'.$error_class.'.php');
				//注意优先级！
				if (($e404 = ! class_exists($error_class, FALSE)) === FALSE)
				{
					$RTR->directory = '';
				}
			}
		}
		else
		{
		    //已经声明，则撤销404标识
			$e404 = FALSE;
		}
	}

	// Did we reset the $e404 flag? If so, set the rsegments, starting from index 1
	//万法归宗，现在可以做最终决策了
	if ( ! $e404)
	{
		$class = $error_class;
		$method = $error_method;

		$URI->rsegments = array(
			1 => $class,
			2 => $method
		);
	}
	else
	{
		show_404($RTR->directory.$class.'/'.$method);
	}
}

//如果方法是_remap则不需要重新抽离出参数了，上面已经做过处理
if ($method !== '_remap')
{
	$params = array_slice($URI->rsegments, 2);
}



$CI = new $class();


call_user_func_array(array(&$CI, $method), $params);

```	

1. 加载函数load_class是核心的函数，注意和is_loaded做区别，前者缓存记载类的实例，后者只是缓存了已加载的一些类名而已

下面逐个讲解CodeIgniter.php文件里面出现的一些类库

#### Benchmark.php
```
<?php

defined('BASEPATH') OR exit('No direct script access allowed');
class CI_Benchmark {

	/**
	 * List of all benchmark markers
	 *
	 * @var	array
	 */
	public $marker = array();

	//用来打标记的
	public function mark($name)
	{
		$this->marker[$name] = microtime(TRUE);
	}

	//用来计算前后两个标记之间的时间差的，在Output类中有调用
	public function elapsed_time($point1 = '', $point2 = '', $decimals = 4)
	{
		if ($point1 === '')
		{
			return '{elapsed_time}';
		}

		if ( ! isset($this->marker[$point1]))
		{
			return '';
		}

		if ( ! isset($this->marker[$point2]))
		{
			$this->marker[$point2] = microtime(TRUE);
		}

		return number_format($this->marker[$point2] - $this->marker[$point1], $decimals);
	}


	//这个函数整框架并没有用到，写成一个函数，很鸡肋
	public function memory_usage()
	{
		return '{memory_usage}';
	}

}

```


#### Config.php

```
class CI_Config
$config 所有已加载的配置项组成的数组。

$is_loaded 所有已加载的配置文件组成的数组。

item($item[, $index=''])
获取某个配置项。

set_item($item, $value)
设置某个配置项的值。

slash_item($item)
这个方法和 item() 一样，只是在获取的配置项后面添加一个斜线，如果配置项不存在，返回 NULL 。

load([$file = ''[, $use_sections = FALSE[, $fail_gracefully = FALSE]]])
加载配置文件。

site_url()
该方法返回你的网站的 URL ，包括你在配置文件中设置的 "index" 值。

base_url()
该方法返回你的网站的根 URL ，你可以在后面加上样式和图片的路径来访问它们。
```

##### 1. 构造函数中一开始就获取站点地址，注意区分一下几个容易混淆的变量。
```
var_dump($_SERVER['SERVER_ADDR']);
var_dump($_SERVER['SCRIPT_NAME']);
var_dump($_SERVER['SCRIPT_FILENAME']);
```
运行结果：
![image](E:/image/360反馈意见截图16421031708750.png)

##### 2.用到了is_https函数

判断一个请求是否是https可以根据$_SERVER[=='HTTPS==']、$_SERVER['==HTTP_X_FORWARDED_PROTO==']、$_SERVER['==HTTP_FRONT_END_HTTPS=='来判断。

```
function is_https()
	{
		if ( ! empty($_SERVER['HTTPS']) && strtolower($_SERVER['HTTPS']) !== 'off')
		{
			return TRUE;
		}
		elseif (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && strtolower($_SERVER['HTTP_X_FORWARDED_PROTO']) === 'https')
		{
			return TRUE;
		}
		elseif ( ! empty($_SERVER['HTTP_FRONT_END_HTTPS']) && strtolower($_SERVER['HTTP_FRONT_END_HTTPS']) !== 'off')
		{
			return TRUE;
		}

		return FALSE;
	}
```
#### Utf8.php
mb_convet_encoding和iconv的参数刚好是==相反==的哦，==iconv先进后出==。
```
public function convert_to_utf8($str, $encoding)
	{
		if (MB_ENABLED)
		{
			return mb_convert_encoding($str, 'UTF-8', $encoding);
		}
		elseif (ICONV_ENABLED)
		{
		        //先进后出
			return @iconv($encoding, 'UTF-8', $str);
		}

		return FALSE;
	}
```
ascii码总共128个，从0到127。正则匹配的时候要加S：==“/[^\x00-\x7F]/S”==
```
	/**
	 * Is ASCII?
	 *
	 * Tests if a string is standard 7-bit ASCII or not.
	 *
	 * @param	string	$str	String to check
	 * @return	bool
	 */
	public function is_ascii($str)
	{
		return (preg_match('/[^\x00-\x7F]/S', $str) === 0);
	}
```

#### Security.php
安全类里面的方法主要分为两类，一类是防csrf攻击，一类是防Xss攻击

```
	/**
	 * CSRF Verify
	 *
	 * @return	CI_Security
	 */
	public function csrf_verify()
	{

        //如果不是一个POST请求，则说明是一般的页面请求，而非数据提交，我们将把csrf hash值注入进cookie，作为响应返回
		if (strtoupper($_SERVER['REQUEST_METHOD']) !== 'POST')
		{
			return $this->csrf_set_cookie();
		}
        //开始验证cookie里的CSRF hash值是否和post里面的token值相等

        //检查URI是否已列入CSRF白名单,如果是，则直接返回
		if ($exclude_uris = config_item('csrf_exclude_uris'))
		{
			$uri = load_class('URI', 'core');
			foreach ($exclude_uris as $excluded)
			{
				if (preg_match('#^'.$excluded.'$#i'.(UTF8_ENABLED ? 'u' : ''), $uri->uri_string()))
				{
					return $this;
				}
			}
		}

		//验证是否相等
		$valid = isset($_POST[$this->_csrf_token_name], $_COOKIE[$this->_csrf_cookie_name])
			&& hash_equals($_POST[$this->_csrf_token_name], $_COOKIE[$this->_csrf_cookie_name]);

		// 清理 _POST array
		unset($_POST[$this->_csrf_token_name]);

		// 根据配置决定是否在每次验证完成之后重新设置cookie里的csrf hash值，默认是需要重新设置的
		if (config_item('csrf_regenerate'))
		{
			//直接导致_csrf_set_hash函数重新生成csrf hash
			unset($_COOKIE[$this->_csrf_cookie_name]);
			$this->_csrf_hash = NULL;
		}

		$this->_csrf_set_hash();
		$this->csrf_set_cookie();

		if ($valid !== TRUE)
		{
			$this->csrf_show_error();
		}

		log_message('info', 'CSRF token verified');
		return $this;
	}

	// --------------------------------------------------------------------

	/**
	 * CSRF Set Cookie
	 *
	 * @codeCoverageIgnore
	 * @return	CI_Security
	 */
	public function csrf_set_cookie()
	{
		$expire = time() + $this->_csrf_expire;
		$secure_cookie = (bool) config_item('cookie_secure');

		if ($secure_cookie && ! is_https())
		{
			return FALSE;
		}

		setcookie(
			$this->_csrf_cookie_name,
			$this->_csrf_hash,
			$expire,//一定要设置时效，才有意义
			config_item('cookie_path'),
			config_item('cookie_domain'),
			$secure_cookie,//为true需要开启https
			config_item('cookie_httponly')
		);
		log_message('info', 'CSRF cookie sent');

		return $this;
	}
    /**
     * Set CSRF Hash and Cookie
     *
     * @return	string
     */
    protected function _csrf_set_hash()
    {
        if ($this->_csrf_hash === NULL)
        {

            /* 如果cookie存在，我们将直接使用它的值。不一定要在每个页面加载的时候都重新生成，因为一个页面可能包含嵌入的子页面导致此功能失败*/
            if (isset($_COOKIE[$this->_csrf_cookie_name]) && is_string($_COOKIE[$this->_csrf_cookie_name])
                && preg_match('#^[0-9a-f]{32}$#iS', $_COOKIE[$this->_csrf_cookie_name]) === 1)
            {
                return $this->_csrf_hash = $_COOKIE[$this->_csrf_cookie_name];
            }

            $rand = $this->get_random_bytes(16);
            $this->_csrf_hash = ($rand === FALSE)
                ? md5(uniqid(mt_rand(), TRUE))
                : bin2hex($rand);
        }

        return $this->_csrf_hash;
    }

```

$_POST[$this->_csrf_token_name]需要结合input类去看
```
/**
 * @param $str
 * @param bool $url_encoded
 * @return mixed
 * 移除不可见字符
 */
function remove_invisible_characters($str, $url_encoded = TRUE)
{
	$non_displayables = array();
	if ($url_encoded)
	{
	    //url编码中的不可见字符     
		$non_displayables[] = '/%0[0-8bcef]/';
		$non_displayables[] = '/%1[0-9a-f]/';
	}

	$non_displayables[] = '/[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]+/S';

	do
	{
		$str = preg_replace($non_displayables, '', $str, -1, $count);
	}
	while ($count);

	return $str;
}
```
'/[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]+/S'匹配的是assic码中那些不可见的字符


##### 以下是过滤xss攻击的相关函数

###### 主函数

CodeIgniter 自带了一个 XSS 过滤器来防御攻击，
它可以设置为自动运行过滤所有遇到的 POST 和 COOKIE 数据，
也可以针对某一条数据进行过滤。默认情况下它不是全局运行的，
因为它会有相当的开销，况且你并不是在所有地方都需要它。
        
```
/**
        public function xss_clean($str, $is_image = FALSE)
        {
            if (is_array($str)) {
                while (list($key) = each($str)) {
                    $str[$key] = $this->xss_clean($str[$key]);
                }
                return $str;
            }
            //删除不可见字符
            $str = remove_invisible_characters($str);
            //对url字符串进行编码
            do {
                $str = rawurldecode($str);
            } while (preg_match('/%[0-9a-f]{2,}/i', $str));
            //转换为ASCII字符实体
            $str = preg_replace_callback("/[^a-z0-9>]+[a-z0-9]+=([\'\"]).*?\\1/si", array($this, '_convert_attribute'), $str);
            $str = preg_replace_callback('/<\w+.*/si', array($this, '_decode_entity'), $str);
            //再次删除不可见字符！
            $str = remove_invisible_characters($str);
            //将所有标签转换为空格
            $str = str_replace("\t", ' ', $str);
            //捕获转换后的字符串
            $converted_string = $str;
            //删除不允许的字符串
            $str = $this->_do_never_allowed($str);
    
            //使PHP标签安全
            if ($is_image === TRUE) {
                //图像有一种倾向，有PHP短开闭标签常常让我们跳过这些，
                //只做长期开放标签。
                $str = preg_replace('/<\?(php)/i', '<?\\1', $str);
            } else {
                $str = str_replace(array('<?', '?' . '>'), array('<?', '?>'), $str);
            }
            $words = array(
                'javascript', 'expression', 'vbscript', 'jscript', 'wscript',
                'vbs', 'script', 'base64', 'applet', 'alert', 'document',
                'write', 'cookie', 'window', 'confirm', 'prompt', 'eval'
            );
            foreach ($words as $word) {
                $word = implode('\s*', str_split($word)) . '\s*';
                $str = preg_replace_callback('#(' . substr($word, 0, -3) . ')(\W)#is', array($this, '_compact_exploded_words'), $str);
            }
            do {
                $original = $str;
                if (preg_match('/<a/i', $str)) {
                    $str = preg_replace_callback('#<a[^a-z0-9>]+([^>]*?)(?:>|$)#si', array($this, '_js_link_removal'), $str);
                }
                if (preg_match('/<img/i', $str)) {
                    $str = preg_replace_callback('#<img[^a-z0-9]+([^>]*?)(?:\s?/?>|$)#si', array($this, '_js_img_removal'), $str);
                }
                if (preg_match('/script|xss/i', $str)) {
                    $str = preg_replace('#</*(?:script|xss).*?>#si', '[removed]', $str);
                }
            } while ($original !== $str);
            unset($original);
			//\042\047是双引号和单引号的8进制表示
			//(?<labelName>...)可以标记捕获组，具体表现为捕获组的匹配结果作为以labelName为key的值
            $pattern = '#'
                . '<((?<slash>/*\s*)(?<tagName>[a-z0-9]+)(?=[^a-z0-9]|$)' // tag start and name, followed by a non-tag character
                . '[^\s\042\047a-z0-9>/=]*' // a valid attribute character immediately after the tag would count as a separator
                // optional attributes
                . '(?<attributes>(?:[\s\042\047/=]*' // non-attribute characters, excluding > (tag close) for obvious reasons
                . '[^\s\042\047>/=]+' // attribute characters
                // optional attribute-value
                . '(?:\s*=' // attribute-value separator
                . '(?:[^\s\042\047=><`]+|\s*\042[^\042]*\042|\s*\047[^\047]*\047|\s*(?U:[^\s\042\047=><`]*))' // single, double or non-quoted value
                . ')?' // end optional attribute-value group
                . ')*)' // end optional attributes group
                . '[^>]*)(?<closeTag>\>)?#isS';
            do {
                $old_str = $str;
                $str = preg_replace_callback($pattern, array($this, '_sanitize_naughty_html'), $str);
            } while ($old_str !== $str);
            unset($old_str);
    
            $str = preg_replace(
                '#(alert|prompt|confirm|cmd|passthru|eval|exec|expression|system|fopen|fsockopen|file|file_get_contents|readfile|unlink)(\s*)\((.*?)\)#si',
                '\\1\\2(\\3)',
                $str
            );
            //这增加了一点额外的预防措施的情况下，通过上述过滤器
            $str = $this->_do_never_allowed($str);
            if ($is_image === TRUE) {
                return ($str === $converted_string);
            }
            return $str;
        }
    
```
看到上面一大段正则匹配字符串是不是有点蒙，下面做进一步剖析。

```

$str = '<div "name"="attr" class="radio-inline" id="radio-wraper">  
    <input onclick="alert(doc)" type="radio" name="optionsRadios" id="Radiostxt" value="txt"/>
    文本方式显示
</div>
</select';
//\042\047是双引号和单引号的8进制表示
//(?<labelName>...)可以标记捕获组，具体表现为捕获组的匹配结果作为以labelName为key的值
$pattern = '#'   //第一个匹配组会去掉两边的<>
	. '<((?<slash>/*\s*)(?<tagName>[a-z0-9]+)(?=[^a-z0-9]|$)' 
	// 当遇到结束标签时，slash是/，否则是空。tagName是标签名，例如div、input,注意这里标签名只能是字母和数字的组合，自定义标签如section-item不适用
	//非常规字符或者直接在字符串末尾，例如上面的‘</select’
	. '(?<separator>[^\s\042\047a-z0-9>/=])*' // a valid attribute character immediately after the tag would count as a separator
	// optional attributes
	. '(?<attributes>(?:[\s\042\047/=]*' // 匹配属性组，例如<div class="radio-inline" id="radio-wraper">中的'class="radio-inline" id="radio-wraper"'
	. '[^\s\042\047>/=]+' // attribute characters
	// optional attribute-value
	. '(?:\s*=' // attribute-value separator
	. '(?:[^\s\042\047=><`]+|\s*\042[^\042]*\042|\s*\047[^\047]*\047|\s*(?U:[^\s\042\047=><`]*))' // single, double or non-quoted value
	. ')?' // end optional attribute-value group
	. ')*)' // end optional attributes group
	. '[^>]*)(?<closeTag>\>)?#isS';//关闭符号都是>，例如：‘ <input type="text" name="fname" />’
	do {
		$old_str = $str;
		$str = preg_replace_callback($pattern, 'test', $str);
		
	} while ($old_str !== $str);
   
	function test($maches){
		echo '<pre>';
		var_dump($maches);
		echo '</pre>';
	}
	
	echo '----------------------';
	echo '<pre>';
	var_dump(htmlentities($str));
	echo '</pre>';
```
运行结果如下：
```

<pre>array(12) {
  [0]=>
  string(58) "<div "name"="attr" class="radio-inline" id="radio-wraper">"
  [1]=>
  string(56) "div "name"="attr" class="radio-inline" id="radio-wraper""
  ["slash"]=>
  string(0) ""
  [2]=>
  string(0) ""
  ["tagName"]=>
  string(3) "div"
  [3]=>
  string(3) "div"
  ["separator"]=>
  string(0) ""
  [4]=>
  string(0) ""
  ["attributes"]=>
  string(53) " "name"="attr" class="radio-inline" id="radio-wraper""
  [5]=>
  string(53) " "name"="attr" class="radio-inline" id="radio-wraper""
  ["closeTag"]=>
  string(1) ">"
  [6]=>
  string(1) ">"
}
</pre><pre>array(12) {
  [0]=>
  string(90) "<input onclick="alert(doc)" type="radio" name="optionsRadios" id="Radiostxt" value="txt"/>"
  [1]=>
  string(88) "input onclick="alert(doc)" type="radio" name="optionsRadios" id="Radiostxt" value="txt"/"
  ["slash"]=>
  string(0) ""
  [2]=>
  string(0) ""
  ["tagName"]=>
  string(5) "input"
  [3]=>
  string(5) "input"
  ["separator"]=>
  string(0) ""
  [4]=>
  string(0) ""
  ["attributes"]=>
  string(82) " onclick="alert(doc)" type="radio" name="optionsRadios" id="Radiostxt" value="txt""
  [5]=>
  string(82) " onclick="alert(doc)" type="radio" name="optionsRadios" id="Radiostxt" value="txt""
  ["closeTag"]=>
  string(1) ">"
  [6]=>
  string(1) ">"
}
</pre><pre>array(12) {
  [0]=>
  string(6) "</div>"
  [1]=>
  string(4) "/div"
  ["slash"]=>
  string(1) "/"
  [2]=>
  string(1) "/"
  ["tagName"]=>
  string(3) "div"
  [3]=>
  string(3) "div"
  ["separator"]=>
  string(0) ""
  [4]=>
  string(0) ""
  ["attributes"]=>
  string(0) ""
  [5]=>
  string(0) ""
  ["closeTag"]=>
  string(1) ">"
  [6]=>
  string(1) ">"
}
</pre><pre>array(10) {
  [0]=>
  string(8) "</select"
  [1]=>
  string(7) "/select"
  ["slash"]=>
  string(1) "/"
  [2]=>
  string(1) "/"
  ["tagName"]=>
  string(6) "select"
  [3]=>
  string(6) "select"
  ["separator"]=>
  string(0) ""
  [4]=>
  string(0) ""
  ["attributes"]=>
  string(0) ""
  [5]=>
  string(0) ""
}
</pre>----------------------<pre>string(0) ""
</pre>
```
##### 净化敏感标签和属性
'on\w+', 'style', 'xmlns', 'formaction', 'form', 'xlink:href', 'FSCommand', 'seekSegmentTime'都属于敏感属性。

'alert', 'prompt', 'confirm', 'applet', 'audio', 'basefont', 'base', 'behavior', 'bgsound',
                'blink', 'body', 'embed', 'expression', 'form', 'frameset', 'frame', 'head', 'html', 'ilayer',
                'iframe', 'input', 'button', 'select', 'isindex', 'layer', 'link', 'meta', 'keygen', 'object',
                'plaintext', 'style', 'script', 'textarea', 'title', 'math', 'video', 'svg', 'xml', 'xss'都属于敏感标签
```
function _sanitize_naughty_html($matches)
        
            static $naughty_tags = array(
                'alert', 'prompt', 'confirm', 'applet', 'audio', 'basefont', 'base', 'behavior', 'bgsound',
                'blink', 'body', 'embed', 'expression', 'form', 'frameset', 'frame', 'head', 'html', 'ilayer',
                'iframe', 'input', 'button', 'select', 'isindex', 'layer', 'link', 'meta', 'keygen', 'object',
                'plaintext', 'style', 'script', 'textarea', 'title', 'math', 'video', 'svg', 'xml', 'xss'
            );
            static $evil_attributes = array(
                'on\w+', 'style', 'xmlns', 'formaction', 'form', 'xlink:href', 'FSCommand', 'seekSegmentTime'
            );
            //未闭合标签不予处理
            if (empty($matches['closeTag'])) {
                return '<' . $matches[1];
				
            } // 敏感标签，清除标签属性内容
            elseif (in_array(strtolower($matches['tagName']), $naughty_tags, TRUE)) {
                return '<' . $matches[1] . '>';// /select或者div
            } // 不属于敏感标签，则过滤不规则属性
            elseif (isset($matches['attributes'])) {
                $attributes = array();
                $attributes_pattern = '#'
                    . '(?<name>[^\s\042\047>/=]+)' // attribute characters
                    . '(?:\s*=(?<value>[^\s\042\047=><`]+|\s*\042[^\042]*\042|\s*\047[^\047]*\047|\s*(?U:[^\s\042\047=><`]*)))' // attribute-value separator
                    . '#i';
                $is_evil_pattern = '#^(' . implode('|', $evil_attributes) . ')$#i';
                do {
                    $matches['attributes'] = preg_replace('#^[^a-z]+#i', '', $matches['attributes']);
                    if (!preg_match($attributes_pattern, $matches['attributes'], $attribute, PREG_OFFSET_CAPTURE)) {
                        break;
                    }
                    if (preg_match($is_evil_pattern, $attribute['name'][0]) OR (trim($attribute['value'][0]) === '')) {
                        $attributes[] = 'xss=removed';
                    } else {
                        $attributes[] = $attribute[0][0];
                    }
                    $matches['attributes'] = substr($matches['attributes'], $attribute[0][1] + strlen($attribute[0][0]));
                } while ($matches['attributes'] !== '');
	         //重新组装属性组
                $attributes = empty($attributes)
                    ? ''
                    : ' ' . implode(' ', $attributes);
                return '<' . $matches['slash'] . $matches['tagName'] . $attributes . '>';
            }
            return $matches[0];
        }
```

#####  HTML实体解码函数
######  原理是：获取实体字符与原字符的映射表，在目标字符串中查找相关实体字符，然后替换
```
/**
         * HTML实体解码
         */
        public function entity_decode($str, $charset = NULL)
        {
            if (strpos($str, '&') === FALSE) {
                return $str;
            }
            static $_entities;
            isset($charset) OR $charset = $this->charset;
            $flag = is_php('5.4') ? ENT_COMPAT | ENT_HTML5 : ENT_COMPAT;
            do {
                $str_compare = $str;
                //解码标准实体，避免误报
                if (preg_match_all('/&[a-z]{2,}(?![a-z;])/i', $str, $matches)) {
                    if (!isset($_entities)) {
                        $_entities = array_map(
                            'strtolower',
                            is_php('5.3.4') ? get_html_translation_table(HTML_ENTITIES, $flag, $charset) : get_html_translation_table(HTML_ENTITIES, $flag)
                        );
                        if ($flag === ENT_COMPAT) {
                            $_entities[':'] = ':';
                            $_entities['('] = '(';
                            $_entities[')'] = ')';
                            $_entities["\n"] = '&newline;';
                            $_entities["\t"] = '&tab;';
                        }
                    }
                    $replace = array();
                    $matches = array_unique(array_map('strtolower', $matches[0]));
                    foreach ($matches as &$match) {
                        if (($char = array_search($match . ';', $_entities, TRUE)) !== FALSE) {
                            $replace[$match] = $char;
                        }
                    }
                    $str = str_ireplace(array_keys($replace), array_values($replace), $str);
                }
                //解码数字和UTF16两字节单位
                $str = html_entity_decode(
                    preg_replace('/(&#(?:x0*[0-9a-f]{2,5}(?![0-9a-f;])|(?:0*\d{2,4}(?![0-9;]))))/iS', '$1;', $str),
                    $flag,
                    $charset
                );
            } while ($str_compare !== $str);
            return $str;
        }
    
```

#### Common.php


1. 判断php版本
1. 判断是否可写
1. 类加载函数
1. 跟踪类加载
1. 判断是否是https请求
1. 判断是否是cli模式
1. 显示错误
1. 记录消息
1. 设置返回状态
1. 异常处理函数
1. 错误处理函数
1. 关闭处理函数
1. 移除不可见字符
1. 实体化字符串或者数组
1. 字符串化参数
1. 判断函数可用性

```
/**
	 * Is CLI?
	 *
	 * Test to see if a request was made from the command line.
	 *
	 * @return 	bool
	 */
	function is_cli()
	{
		return (PHP_SAPI === 'cli' OR defined('STDIN'));
	}
```
