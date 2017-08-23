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
#### Hooks.php
略


```
<?php

defined('BASEPATH') OR exit('No direct script access allowed');

class CI_Config {

	public $config = array();

	public $is_loaded =	array();


	public $_config_paths =	array(APPPATH);

	
	public function __construct()
	{
		$this->config =& get_config();

		// Set the base_url automatically if none was provided
		if (empty($this->config['base_url']))
		{
			if (isset($_SERVER['SERVER_ADDR']))
			{
				if (strpos($_SERVER['SERVER_ADDR'], ':') !== FALSE)
				{
					$server_addr = '['.$_SERVER['SERVER_ADDR'].']';
				}
				else
				{
					$server_addr = $_SERVER['SERVER_ADDR'];
				}

				$base_url = (is_https() ? 'https' : 'http').'://'.$server_addr
 					.substr($_SERVER['SCRIPT_NAME'], 0, strpos($_SERVER['SCRIPT_NAME'],                 basename($_SERVER['SCRIPT_FILENAME'])));
			}
			else
			{
				$base_url = 'http://localhost/';
			}

			$this->set_item('base_url', $base_url);
		}

		log_message('info', 'Config Class Initialized');
	}


	public function load($file = '', $use_sections = FALSE, $fail_gracefully = FALSE)
	{
		$file = ($file === '') ? 'config' : str_replace('.php', '', $file);
		$loaded = FALSE;

		foreach ($this->_config_paths as $path)
		{
			foreach (array($file, ENVIRONMENT.DIRECTORY_SEPARATOR.$file) as $location)
			{
				$file_path = $path.'config/'.$location.'.php';
				if (in_array($file_path, $this->is_loaded, TRUE))
				{
					return TRUE;
				}

				if ( ! file_exists($file_path))
				{
					continue;
				}

				include($file_path);

				if ( ! isset($config) OR ! is_array($config))
				{
					if ($fail_gracefully === TRUE)
					{
						return FALSE;
					}

					show_error('Your '.$file_path.' file does not appear to contain a valid configuration array.');
				}

				if ($use_sections === TRUE)
				{
					$this->config[$file] = isset($this->config[$file])
						? array_merge($this->config[$file], $config)
						: $config;
				}
				else
				{
					$this->config = array_merge($this->config, $config);
				}

				$this->is_loaded[] = $file_path;
				$config = NULL;
				$loaded = TRUE;
				log_message('debug', 'Config file loaded: '.$file_path);
			}
		}

		if ($loaded === TRUE)
		{
			return TRUE;
		}
		elseif ($fail_gracefully === TRUE)
		{
			return FALSE;
		}

		show_error('The configuration file '.$file.'.php does not exist.');
	}

	
	public function item($item, $index = '')
	{
		if ($index == '')
		{
			return isset($this->config[$item]) ? $this->config[$item] : NULL;
		}

		return isset($this->config[$index], $this->config[$index][$item]) ? $this->config[$index][$item] : NULL;
	}


	public function slash_item($item)
	{
		if ( ! isset($this->config[$item]))
		{
			return NULL;
		}
		elseif (trim($this->config[$item]) === '')
		{
			return '';
		}

		return rtrim($this->config[$item], '/').'/';
	}

	public function site_url($uri = '', $protocol = NULL)
	{
		$base_url = $this->slash_item('base_url');

		if (isset($protocol))
		{
			// For protocol-relative links
			if ($protocol === '')
			{
				$base_url = substr($base_url, strpos($base_url, '//'));
			}
			else
			{
				$base_url = $protocol.substr($base_url, strpos($base_url, '://'));
			}
		}

		if (empty($uri))
		{
			return $base_url.$this->item('index_page');
		}

		$uri = $this->_uri_string($uri);

		if ($this->item('enable_query_strings') === FALSE)
		{
			$suffix = isset($this->config['url_suffix']) ? $this->config['url_suffix'] : '';

			if ($suffix !== '')
			{
				if (($offset = strpos($uri, '?')) !== FALSE)
				{
					$uri = substr($uri, 0, $offset).$suffix.substr($uri, $offset);
				}
				else
				{
					$uri .= $suffix;
				}
			}

			return $base_url.$this->slash_item('index_page').$uri;
		}
		elseif (strpos($uri, '?') === FALSE)
		{
			$uri = '?'.$uri;
		}

		return $base_url.$this->item('index_page').$uri;
	}


	public function base_url($uri = '', $protocol = NULL)
	{
		$base_url = $this->slash_item('base_url');

		if (isset($protocol))
		{
			// For protocol-relative links
			if ($protocol === '')
			{
				$base_url = substr($base_url, strpos($base_url, '//'));
			}
			else
			{
				$base_url = $protocol.substr($base_url, strpos($base_url, '://'));
			}
		}

		return $base_url.ltrim($this->_uri_string($uri), '/');
	}


	protected function _uri_string($uri)
	{
		if ($this->item('enable_query_strings') === FALSE)
		{
			if (is_array($uri))
			{
				$uri = implode('/', $uri);
			}
			return trim($uri, '/');
		}
		elseif (is_array($uri))
		{
			return http_build_query($uri);
		}

		return $uri;
	}


	public function system_url()
	{
		$x = explode('/', preg_replace('|/*(.+?)/*$|', '\\1', BASEPATH));
		return $this->slash_item('base_url').end($x).'/';
	}

	public function set_item($item, $value)
	{
		$this->config[$item] = $value;
	}

}

```

```
class CI_Config
$config
所有已加载的配置项组成的数组。

$is_loaded
所有已加载的配置文件组成的数组。

item($item[, $index=''])
参数:	
$item (string) -- Config item name
$index (string) -- Index name
返回:	
Config item value or NULL if not found

返回类型:	
mixed

获取某个配置项。

set_item($item, $value)
参数:	
$item (string) -- Config item name
$value (string) -- Config item value
返回类型:	
void

设置某个配置项的值。

slash_item($item)
参数:	
$item (string) -- config item name
返回:	
Config item value with a trailing forward slash or NULL if not found

返回类型:	
mixed

这个方法和 item() 一样，只是在获取的配置项后面添加一个斜线，如果配置项不存在，返回 NULL 。

load([$file = ''[, $use_sections = FALSE[, $fail_gracefully = FALSE]]])
参数:	
$file (string) -- Configuration file name
$use_sections (bool) -- Whether config values shoud be loaded into their own section (index of the main config array)
$fail_gracefully (bool) -- Whether to return FALSE or to display an error message
返回:	
TRUE on success, FALSE on failure

返回类型:	
bool

加载配置文件。

site_url()
返回:	Site URL
返回类型:	string
该方法返回你的网站的 URL ，包括你在配置文件中设置的 "index" 值。

这个方法通常通过 URL 辅助函数 中函数来访问。

base_url()
返回:	Base URL
返回类型:	string
该方法返回你的网站的根 URL ，你可以在后面加上样式和图片的路径来访问它们。

这个方法通常通过 URL 辅助函数 中函数来访问。

system_url()
返回:	URL pointing at your CI system/ directory
返回类型:	string
该方法返回 CodeIgniter 的 system 目录的 URL 。

注解

该方法已经废弃，因为这是一个不安全的编码实践。你的 system/ 目录不应该被公开访问。·
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