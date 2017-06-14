# php-resque : PHP Resque Worder (and Enqueue) 

翻译自[php-resque](https://github.com/chrisboulton/php-resque/blob/master/README.md)

## 名词解释
* job:       任务
* queue：    任务队列
* worker：   任务执行者
* interface: 接口
* feature:   特性
* priority:  优先级
* event:     事件
* hook:      钩子

## 什么是Resque?
Resque是一套基于Redis数据库的用于创建任务，并将新任务添加到一个和多个任务队列中，用于延迟处理的系统。

项目地址: [php-resque](https://github.com/chrisboulton/php-resque)

## php-resque项目背景
Resque项目是一套在Github平台开源的Ruby项目。php-resque是该项目的PHP重写版本。

可以在[https://github.com/resque/resque](https://github.com/resque/resque)Resque官方项目地址了解更多关于它的知识。

同时可以查看这个[blog](https://github.com/blog/542-introducing-resque)加深对Resque的了解

php-resque不但提供了自己的web接口查看任务队列的统计信息，同时实现了Ruby Resque项目相同的数据存储方式。

可以说php-resque提供了与Ruby版本的相同特性：
* 分布式的任务执行者
* 支持任务队列优先级设置
* 防内存泄漏
* 任务失败处理机制

此外，php-resque提供了这些新的特性：
* 提供跟踪任务状态的能力
* 任务执行失败处理：如果在子进程中任务状态码非0而非正常推出，该任务即判为任务执行失败
* 内建执行任务前和执行中调用的`setUp`和`tearDown`方法

## 系统要求
* PHP 5.3 以上
* Redis 2.2以上
* 推荐使用Composer包管理工具(可选)

## 入门 (Getting Startted)
使用Composer包管理工具，可以非常容易的创建一个php-resque项目。即使Composer非必需，但是它可以让你的开发更加便利

如果你还不了解Composer，可以查阅[http://getcomposer.org/](http://getcomposer.org/)(中文版本在[这里](http://docs.phpcomposer.com/))。

1. 将php-resque项目依赖加入到composer.json文件中

```javascript
    {
        "require" : {
            "chrisboulton/php-resque" : "1.2.x"
        }
    }
```
2. 运行 `composer install`命令

3. 将Composer 自动加载器引入项目初始化文件（示例）

```
require "vendor/autoload.php";
```

## 任务

### 加入任务到任务队列(Queueing Jobs)
所有任务通过下面的方式加入到任务队列中：
```php
Resque::setBackend('localhost:6379');

$args = array(
            "name" => "Chris"
        );

Resque::enqueue("default", "My_Job", $args);
```

### 定义任务(Defining Jobs)
任务是一个必须定义`perform`方法的类，并且每个任务都对应一个实现类
```php
class My_Job
{
    public function perform()
    {
        //TODO: work work work 重要的事情说三遍么？
        echo $this->args["name"];
    }
}

```

当任务执行时，任务实现类将会被实例化，同时所有的参数都会设置到对象实例中通过`$this->args`参数可以获取对应参数值

任何的异常都会导致任务执行失败 - 所以在执行任务时务必小心谨慎，确保所有的异常都被捕捉处理了。

任务实现类中可以定义`setUp`和`tearDown`方法。
`setUp`方法将在`perform`方法调用前调用
`tearDown`方法将在任务完成后执行

```php
class My_Job
{
    public function setUp()
    {
        //该任务的环境设置
    }

    public function perform()
    {
        //任务执行方法
    }

    public function tearDown()
    {
        //回收该任务环境设置
    }
}
```

## 弹出任务队列中任务(Dequeueing Jobs)
通过`Resque::dequeue()`方法可以很轻松的删除一个任务
```php
//删除`default`任务队列中的"My_Job"任务
Resque::dequeue("default", ["My_Job"]);

//删除`default`任务队列中任务ID为`087df5819a790ac666c9608e2234b21e`的"My_Job"任务
Resque::dequeue("default", ["My_Job" => "087df5819a790ac666c9608e2234b21e"]);

//删除`default`任务队列中的带参数的"My_Job"任务
Resque::dequeue("default", ["My_Job" => array("foo" => 1, "bar" => 2)]);

//删除多个任务
Resque::dequeue("default", ["My_Job", "My_Job2"]);
```

如果为给定`dequeue`方法第二个参数，将会删除所有匹配的任务
```php
//删除所有`default`任务队列中的任务
Resque::dequeue("default");
```

## 任务状态跟踪
php-resque提供简单的队列任务状态跟踪的能力。这些状态提供了判断任务是否在任务列中、任务是否处于执行中、任务是否执行完成或是否失败的有效信息。

### 开启任务状态跟踪
通过设置布尔值`true`到`Resque::enqueue`方法的第四个参数，可以开启任务状态更总功能。并且`Resque::enqueue`方法会返回一个用户状态跟踪的任务ID:

```php
$token = Resque::enqueue("default", "My_Job", $args, true);
echo $token;
```

### 查询任务状态

```php
$status = new Resque_Job_Status($token);
echo $status->get();//输出状态信息
```

### 任务状态定义
任务状态定义在`Resque_Job_Status`类中，以下是有效的状态：
* `Resque_Job_Status::STATUS_WAITING`   任务仍处于任务队列中
* `Resque_Job_Status::STATUS_RUNNINT`   任务执行中
* `Resque_Job_Status::STATUS_FAILED`    任务执行失败
* `Resque_Job_Status::STATUS_COMPLETE`  任务执行完成
* false                                 任务状态获取失败 - 可以通过查看给定的任务ID是否有效？

任务状态有效期为任务执行完成或任务执行失败后的24小时，超过24小时则自动清除。也可以通过`stop`方法手动清除任务状态

## 任务执行者(Workers)
php-resque任务执行者的运行方式和Ruby Resque项目一样。可以查看Ruby Resque官方文档了解关于任务执行者的相关内容。

一个典型的启动和运行文件`bin/resque`包含启动运行任务执行者的运行环境。（`vendor/bin/resque`通过Composer安装自动生成）

同样的php-resque拥有与Ruby Resque任务执行者形似的初始化方法。

然而为了让php-resque在所有环境中都能执行，php-resque提供了自定义的初始化功能，而Ruby Resque则仅能在单一环境下运行。

想要启动一个任务执行者，非常简单：

```
$ QUEUE=file_serve php bin/resque
```

此外你还需要通过`APP_INCLUDE`环境变量设置，告知任务执行者哪个文件是你的项目入口：
```
$ QUEUE=file_serve APP_INCLUDE=../application/init.php php bin/resque
```

提示： 如果你是通过Composer管理工具安装的项目，则无需设置`APP_INCLUDE`选项，因为Composer已经给你自动载入了！

在项目运行入口中，你还需要引入任务执行者需要执行的任务实现类，可以通过自动加载和include引入这些任务实现类

此外，你也可以在项目中使用`include ("bin/resque")`文件跳过`APP_INCLUDE`设置，但需要确保所有的环境变量都设置(setenv)好了

## 日志
所有的日志都会被记录到`STDOUT`中，`VERBASE`会打印基本的调试信息，而`VVERBASE`会打印详细的调试信息

```
$ VERBASE=1 QUEUE=file_serve bin/resque
$ VVERBASE=1 QUEUE=file_serve bin/resque
```

## 任务队列中的优先级设置（Priorities and Queue Lists）
优先级设置也与Ruby的Resque相同。多个队列任务通过逗号分隔，它们的先后顺序即优先级级别

上一个示例
```
$ QUEUE=file_serve,warm_cache bin/resque
```

在每个迭代中，file_serve任务队列中的每个任务都会优先与warm_cache队列中的任务

## 执行所有任务队列（Running All Queues）
通过下面的命令将执行所有任务队列执行，执行顺序为字符顺序：
```
$ QUEUE="*" bin/resque
```

## 同时运行多个任务执行者（Running Multiple Workers）

```
$ COUNT=5 bin/resque
```
是不是很简单，但是需要注意的是，每个任务执行者都有各自的子进程，所有的子进程都将在任务执行完成后立即注销。如果你需要使用`monit`监控任务执行者执行情况，那么你需要限制该功能的使用

## 任务队列前缀（Custom prefix）

当我们有多个项目使用同一个Redis数据库存储队列任务是，那么设置前缀会非常有用
```
$ PREFIX=my-app-name bin/resque
```

## 子进程（Forking）

同Ruby类似，php-resque也支持任务执行子进程功能，并且子进程会在任务完成后尽快销毁。

不过php版本针对未能完美退出的任务，php-resque将自动标识任务执行失败

## 命令（Signals）
* QUIT      等待任务执行完成，再退出
* TERM/INT  立即停止任务并退出
* USR1      立即停止任务但不退出
* USR2      暂停任务执行，所有新任务将不被处理
* CONT      恢复任务执行

## 进程标题/状态（Process Titles/Statuses）
在Ruby项目中有个很好的功能就是能够给任务执行者打标题，并且所有子进程也支持该功能。

但是知道PHP5.5以后，我们才提供了类似功能

在PHP 5.5之前可以使用[PECL扩展模块](http://pecl.php.net/package/proctitle)实现同样的功能。

## 事件/钩子系统（Event/Hook System）
php-resque提供了基本的事件系统，这样可以在项目中监听php-resque内部处理

通过使用`Resque_Event::listen`函数注册一个事件监听器，同时传入回调函数到第二个参数用于处理事件触发后的程序处理

```php
Resque_Event::listen("eventName" [, callback]);
```

`[callback]`可以下面任意可被`call_user_func_array`函数调用的PHP回调
* 回调函数名
* 包含对象实例和方法的数组
* 包含对象实例和静态方法的数据
* PHP 5.3以后的闭包（Closure）

事件可以传入参数，所有参数可在回调中接收。事件可以手动停止，只需调用和`listen`有相同参数的`Resque_Event::stopListening`方法即可实现。

当将事件监听加入到项目队列中时，我们需要确保php-resque能够轻松载入和调用`Resque_Event::listen`方法。

当运行任务执行者时。如果是通过默认的`bin/resque`模块运行的任务执行者，我们的`APP_INCLUDE`模块，会初始化并注册所有项目操作所必要的监听器。如果我们通过自定义的任务执行者管理器管理事件，注册监听器则需要自行处理

一个简单的插件示例在[`extra`](https://github.com/chrisboulton/php-resque/tree/master/extras)目录下

## 事件（Events）

* beforeFirstFork

在任务执行者初始化过程中仅执行一次。参数值为`Resque_Worker`实例。

* beforeFork

在php-resque子进程准备运行任务前执行。参数值包含`Resque_Job`实例。

`beforeFork`在父进程中被触发，在任务执行者生命周期内的任何更新都是永久性的。

* afterFork

在php-resque子进程准备运行任务后执行(且在任务运行前)，参数值包含`Resque_Job`实例。

`afterFork` is triggered in the child process after forking out to complete a job. Any changes made will only live as long as the job is being processed

* beforePerform

在`setUp`和`perform`方法运行前调用。参数包含`Resque_Job`实例。

通过抛出`Resque_Job_DontPerform`异常可以停止任务执行，任何其它的异常都将判定任务执行失败

* afterPerform

该方法在`tearDown`和`perform`方法之后执行，参数值包含`Resque_Job`实例。

任何异常都将判定任务执行失败并标记该任务状态为失败状态。

* onFailure

当任务处理失败是被调用，参数为

    1.  Exception  任务失败的异常实例
    2.  Resque_Job 失败的任务实例

* beforeEnqueue

在`Resque::enqueue`方法调用之前立即执行，所有参数列表如下：

    1. Class      加入到任务队列的任务名
    2. Arguments  当前任务执行所需参数数组
    3. Queue      包含当前任务名的任务队列
    4. ID         当前任务的任务ID

可以通过抛出`Resque_Job_DontCreate`异常阻止任务入列

* afterEnqueue

在`Resque::enqueue`方法调用后调用，所有参数列表如下：

    1. Class      计划任务的任务名
    2. Arguments  该任务所需的参数数组
    3. Queue      包含当前任务名的任务队列
    4. ID         当前任务的任务ID

## php-resque生命周期

[看这里](https://github.com/chrisboulton/php-resque/blob/master/HOWITWORKS.md)
