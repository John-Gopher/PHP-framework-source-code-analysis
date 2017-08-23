```php

/**
	 * Load the Database Forge Class
	 *
	 * @param	object	$db	Database object
	 * @param	bool	$return	Whether to return the DB Forge class object or not
	 * @return	object
	 */
	public function dbforge($db = NULL, $return = FALSE)
	{
		$CI =& get_instance();
		if ( ! is_object($db) OR ! ($db instanceof CI_DB))
		{
			class_exists('CI_DB', FALSE) OR $this->database();
			$db =& $CI->db;
		}

		require_once(BASEPATH.'database/DB_forge.php');
		require_once(BASEPATH.'database/drivers/'.$db->dbdriver.'/'.$db->dbdriver.'_forge.php');

		if ( ! empty($db->subdriver))
		{
			$driver_path = BASEPATH.'database/drivers/'.$db->dbdriver.'/subdrivers/'.$db->dbdriver.'_'.$db->subdriver.'_forge.php';
			if (file_exists($driver_path))
			{
				require_once($driver_path);
				$class = 'CI_DB_'.$db->dbdriver.'_'.$db->subdriver.'_forge';
			}
		}
		else
		{
			$class = 'CI_DB_'.$db->dbdriver.'_forge';
		}

		if ($return === TRUE)
		{
			return new $class($db);
		}

		$CI->dbforge = new $class($db);
		return $this;
	}
```
#### 数据库

  数据库这一块，CI框架设计的非常巧妙，里面应用了一些设计模式。

下面是我画的主要几个数据库类之间的关系图
![image](http://chuantu.biz/t5/74/1493255577x1822613067.png)
数据库模型切入有几种方式：

  $this->load->database();
  $this->load->dbforge();
  $this->load->dbutil();
第一种是最常用的，先从第一种入手。

先从loader类中找到database这个方法，按照惯例把里面的英文翻译成中文。

```php
/**
 * 数据库加载器
 *
 * @param  mixed  $params       数据库配置数组
 * @param  bool   $return    是否返回数据库对象
 * @param  bool   $query_builder 查询构造器是否可用，这个会覆盖数据库配置里面的相关配置
 *
 * @return object|bool    如果$return参数设为true，则返回数据库对象,执行失败则返回false
 *          
 */
public function database($params = '', $return = FALSE, $query_builder = NULL)
{
   // 获取CI全局对象
   $CI =& get_instance();

   //不返回数据库对象，并且不启用查询构造器，并且数据库对象已经存在了，并且数据库连接句柄已经存在，则返回false
   if ($return === FALSE && $query_builder === NULL && isset($CI->db) && is_object($CI->db) && ! empty($CI->db->conn_id))
   {
      return FALSE;
   }

   require_once(BASEPATH.'database/DB.php');

   if ($return === TRUE)
   {
      return DB($params, $query_builder);
   }

   //清空数据库对象属性，防止受上次影响
   $CI->db = '';

   // 加载并实例化DB类
   $CI->db =& DB($params, $query_builder);
   return $this;
}
```



接下来进一步跟踪DB构造函数，这个函数在DB.php文件中，整个文件看起来不大。

**DB.php**

```php
function &DB($params = '', $query_builder_override = NULL)
{
	//如果连接参数是字符串，且不是url，则认定是字符串代表的是数据库配置组里的某一配置，加载数据库配置文件
	if (is_string($params) && strpos($params, '://') === FALSE)
	{
		// 先判断配置文件是不是都不存在
		if ( ! file_exists($file_path = APPPATH.'config/'.ENVIRONMENT.'/database.php')
			&& ! file_exists($file_path = APPPATH.'config/database.php'))
		{
			show_error('The configuration file database.php does not exist.');
		}

		include($file_path);

		// Make packages contain database config files,
		// given that the controller instance already exists
		if (class_exists('CI_Controller', FALSE))
		{
			foreach (get_instance()->load->get_package_paths() as $path)
			{
				if ($path !== APPPATH)
				{
					if (file_exists($file_path = $path.'config/'.ENVIRONMENT.'/database.php'))
					{
						include($file_path);
					}
					elseif (file_exists($file_path = $path.'config/database.php'))
					{
						include($file_path);
					}
				}
			}
		}
        //数据库配置不存在
		if ( ! isset($db) OR count($db) === 0)
		{
			show_error('No database connection settings were found in the database config file.');
		}

		if ($params !== '')
		{
			$active_group = $params;
		}


		if ( ! isset($active_group))
		{
			show_error('You have not specified a database connection group via $active_group in your config/database.php file.');
		}
		elseif ( ! isset($db[$active_group]))
		{

			show_error('You have specified an invalid database connection group ('.$active_group.') in your config/database.php file.');
		}

        //从配置组里提取指定配置
		$params = $db[$active_group];
	}
	elseif (is_string($params))
	{

		/**从url中解析出连接参数
		 * $dsn = 'driver://username:password@hostname:port/database';
		 */
		if (($dsn = @parse_url($params)) === FALSE)
		{
			show_error('Invalid DB Connection String');
		}

		$params = array(
			'dbdriver'	=> $dsn['scheme'],
			'hostname'	=> isset($dsn['host']) ? rawurldecode($dsn['host']) : '',
			'port'		=> isset($dsn['port']) ? rawurldecode($dsn['port']) : '',
			'username'	=> isset($dsn['user']) ? rawurldecode($dsn['user']) : '',
			'password'	=> isset($dsn['pass']) ? rawurldecode($dsn['pass']) : '',
			'database'	=> isset($dsn['path']) ? rawurldecode(substr($dsn['path'], 1)) : ''
		);

		//解析额外的带在查询字符串后面的配置
		if (isset($dsn['query']))
		{
			parse_str($dsn['query'], $extra);

			foreach ($extra as $key => $val)
			{
				if (is_string($val) && in_array(strtoupper($val), array('TRUE', 'FALSE', 'NULL')))
				{
					$val = var_export($val, TRUE);
				}

				$params[$key] = $val;
			}
		}
	}

	// No DB specified yet? Beat them senseless...
	if (empty($params['dbdriver']))
	{
		show_error('You have not selected a database type to connect to.');
	}

	//优先考虑由本函数$query_builder_override参数来决定是否启用查询构造器
	if ($query_builder_override !== NULL)
	{
		$query_builder = $query_builder_override;
	}
    //由连接参数决定
	elseif ( ! isset($query_builder) && isset($active_record))
	{
		$query_builder = $active_record;
	}
    //数据库驱动基类
	require_once(BASEPATH.'database/DB_driver.php');
    //如果需要查询构造器则CI_DB继承自DB_query_builder，否则继承自基类CI_DB_driver 
	if ( ! isset($query_builder) OR $query_builder === TRUE)
	{
		require_once(BASEPATH.'database/DB_query_builder.php');
		if ( ! class_exists('CI_DB', FALSE))
		{
			/**
			 * CI_DB
			 *
			 * Acts as an alias for both CI_DB_driver and CI_DB_query_builder.
			 *
			 * @see	CI_DB_query_builder
			 * @see	CI_DB_driver
			 */
			class CI_DB extends CI_DB_query_builder { }
		}
	}
	elseif ( ! class_exists('CI_DB', FALSE))
	{
		/**
	 	 * @ignore
		 */
		class CI_DB extends CI_DB_driver { }
	}

	// 加载相应的数据库驱动
	$driver_file = BASEPATH.'database/drivers/'.$params['dbdriver'].'/'.$params['dbdriver'].'_driver.php';

	file_exists($driver_file) OR show_error('Invalid DB driver');
	require_once($driver_file);

	// 实例化数据库驱动
	$driver = 'CI_DB_'.$params['dbdriver'].'_driver';
	$DB = new $driver($params);

	// 检查是否存在子驱动（pdo里面有，这里暂不做研究）
	if ( ! empty($DB->subdriver))
	{
		$driver_file = BASEPATH.'database/drivers/'.$DB->dbdriver.'/subdrivers/'.$DB->dbdriver.'_'.$DB->subdriver.'_driver.php';

		if (file_exists($driver_file))
		{
			require_once($driver_file);
			$driver = 'CI_DB_'.$DB->dbdriver.'_'.$DB->subdriver.'_driver';
			$DB = new $driver($params);
		}
	}
    //初始化，返回
	$DB->initialize();
	return $DB;
}

```

**新技能get**

**var_export (PHP 4 >= 4.2.0, PHP 5)** 
var_export -- 输出或返回一个变量的字符串表示 
描述 mixed var_export ( mixed expression [, bool return] ) 

此函数返回关于传递给该函数的变量的结构信息，它和 var_dump() 类似，不同的是其返回的表示是合法的 PHP 代码。 
您可以通过将函数的第二个参数设置为 TRUE，从而返回变量的表示。

代码如下:

```php
$data = array ('name' => 'abc', 'job' => 'programmer','a'=>array('aa','cc','bb')); 

data = var_export(data,TRUE); 

echo $data; 

```



输出形式如下： 

> array ( 
>
> 'name' => 'abc', 
>
> 'job' => 'programmer', 
>
> 'a' => 
>
> array ( 
>
> 0 => 'aa', 
>
> 1 => 'cc', 
>
> 2 => 'bb', 
>
> ), 
>
> ) 
>



注意： 

var_export必须返回合法的php字面量， 也就是说，var_export返回的字符串，cp下来可以直接赋值给一个变量。但是， 当var_export的变量是resource类型时， var_export会返回NULL. 



**parse_url**

解析 URL 字符串。

语法: array parse_url(string url);

返回值: 数组

函数种类: 资料处理

内容说明：本函数将 URL 字符串予以解析，并将结果返回数组中。完整的 URL 类似这样子scheme://user:pass@host:port/path?query。

如 http://john:john1234@john.wilson.gs:88/abcdef.php?a=1234

因此返回的数组包括了下列元素：scheme、host、port、user、pass、path、query 与 fragment 等。

输出形式如下： 

> array(7) {
>   ["scheme"]=>
>   string(4) "http"
>   ["host"]=>
>   string(14) "john.wilson.gs"
>   ["port"]=>
>   int(88)
>   ["user"]=>
>   string(4) "john"
>   ["pass"]=>
>   string(8) "john1234"
>   ["path"]=>
>   string(11) "/abcdef.php"
>   ["query"]=>
>   string(6) "a=1234"
> }





