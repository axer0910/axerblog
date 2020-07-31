---
title: PHP自定义异常 
tag: frontend
date: 2018-09-11
updated: 2018-09-11
---

```php
<?php
class Exception{
  protected $message = 'Unknown exception';   // 异常信息
protected $code = 0;                  // 用户自定义异常代码
protected $file;                      // 发生异常的文件名
protected $line;                      // 发生异常的代码行号
 function __construct($message = null, $code = 0);
final function getMessage();          // 返回异常信息
final function getCode();             // 返回异常代码
final function getFile();             // 返回发生异常的文件名
final function getLine();             // 返回发生异常的代码行号
 final function getTrace();            // backtrace() 数组
 final function getTraceAsString();  // 已格成化成字符串的 getTrace() 信息
  /* 可重载的方法 */
 function __toString();                // 可输出的字符串
}
```

```php
<?php
    /* 自定义的一个异常处理类，但必须是扩展内异常处理类的子类 */
    class MyException extends Exception{
        //重定义构造器使第一个参数 message 变为必须被指定的属性
        public function __construct($message, $code=0){
            //可以在这里定义一些自己的代码
         //建议同时调用 parent::construct()来检查所有的变量是否已被赋值
            parent::__construct($message, $code);
        }   
        public function __toString() {        
          //重写父类方法，自定义字符串输出的样式
          return __CLASS__.":[".$this->code."]:".$this->message."<br>";
        }
        public function customFunction() {    
             //为这个异常自定义一个处理方法
             echo "按自定义的方法处理出现的这个类型的异常<br>";
        }
    }
?>

```

```php
<?php
   try { //使用自定义的异常类捕获一个异常，并处理异常
        $error = '允许抛出这个错误';       
        throw new MyException($error);    
            //创建一个自定义的异常类对象，通过throw语句抛出
        echo 'Never executed'; 
            //从这里开始，try代码块内的代码将不会再被执行
    } catch (MyException $e) {        //捕获自定义的异常对象
        echo '捕获异常: '.$e;        //输出捕获的异常消息
        $e->customFunction();  //通过自定义的异常对象中的方法处理异常
    }
    echo '你好呀';

```

```php
<?php
/*
1. 自定义的异常类， 必须是系统类Exception的子类
2. 如果继承Exception类， 重写了构造方法，一定要调一下父类中被覆盖的方法
 */


    //写出对应这个异常解决方法, 就是一下正常类的结构
    class MyBtException extends Exception{
        function __construct($mess) {
            parent::__construct($mess);


        }

        function changBt() {
            echo "换上备胎!";
        }
    }


    echo "早上起床<br>";


try{

    echo "开车上班<br>";

    //抛出异常
    throw  new MyBtException("车子爆胎了");

    echo "路况很好<br>";

} catch(MyBtException $e) {    //  Exception $e = new Exception('');
    echo $e->getMessage()."<br>";
    //自定义类中的解决方法调用， 解决了问题
    $e->changBt()."<br>";
}

    echo "到公司开始工作<br>";

```
