# CodeIgniter 4 的依赖注入

> 原文: [Dependency Injection in CodeIgniter 4](http://blog.newmythmedia.com/blog/show/2016-03-04_Dependency_Injection_in_CodeIgniter_4)

I remember reading a forum thread during the time that we were originally asking for community input on the future of the framework. In it, they ridiculed the community for even considering whether or not we would be using Dependency Injection. At the time, I believe the council was pretty set on having it, but we were letting the discussions and suggestions arise naturally within the community. I read another forum thread the other day on a different site that was looking at our features and wondering why we were bothering since it just read like Laravel, due in large part to the DI, the namespacing, the PSR4 autoloading, etc. I guess you just can't please everyone, right?

当我们在社区中寻求一些对框架未来发展有帮助的建议时，我记得看到过这样一个帖子，在帖子中，一些人嘲笑社区中竟然仍在考虑是否在框架中使用依赖注入机制。当时，理事会已经非常确定将要引入依赖注入机制，但社区中对这方面的讨论和建议却越来越多。而就在另一天，我在一个论坛上看到一个关于 CI 框架特性的讨论，在讨论中他们认为 CI 框架缺乏 Laravel 框架的一些特性，主要是 DI、名字空间和 PSR4 自动加载等特性。

## 为什么 DI 很重要？

Dependency Injection decouples your code from other, specific, classes. When used correctly, it allows you to easily replace dependencies with mock classes during testing, or replace the dependency with a completely different class that handles the task better. In short, it makes the code more resilient to change. It makes it more portable. It's a good thing, without a doubt. If you've spent most of your PHP career using CodeIgniter, you might not have run across Dependency Injection before, so a short example will help clear things up.

> Note: The database layer is still under early development. This is purely an example.

Let's say you have a model for managing Users. You will, naturally need the database class to work with, so without DI you might do something like:

依赖注入的机制或思想的使用，对于你的代码中的具体的类的设计和实现过程可以起到解耦作用，如果使用正确，它可以使你在测试过程中更轻松的用模拟类替换一些依赖关系，或者用一个更完整的完全不同的类替换这层依赖关系，从而达到更好的完成一些功能的效果。简而言之，它使得代码更有灵活性、也有更好的可移植性。毫无疑问这是一种好的编程思想，如果你的大部分PHP生涯都在使用CodeIgniter框架，你可能还没有遇到DI，所以接下来我就举个简单的例子来说明这个思想。

> 注意：数据库层仍在早期开发阶段，这只是个纯粹的简单范例

假如你用一个数据模型去管理用户Users，你当然要用到数据库相关的类去完成，但如果不使用DI思想，你可能会这么写：

```php
class UserModel
{
    protected $db;

    public function __construct()
    {
        $this->db = new Database();
    }
}
```

But there's a problem here. If you ever need to use a different database library, or switch from MySQL to MongoDB, you have to change code in every class. If you're running tests, you can never separate the logic in your UserModel from the Database class. Long term maintenance can become a problem, too.

To fix this, the next step is to pass the Database class into the constructor as a dependency. This solves all of those problems, especially when you're requiring the a class that implements an interface, instead of any specific class.

这样就存在一个问题，如果你需要使用一个不同的数据库，或者从MySQL库换做使用MongoDB，你可能就需要去修改每一个类中的代码。如果你跑一些测试用例，你可能永远没有办法把UserModel与数据库Database类的逻辑分离开来。长期的维护更是一个棘手的难题。

所以解决这种问题，下一步就是把Database类以参数形式传入构造器_constructor中作为一种依赖。这样就解决了以上问题，特别是当你需要一个类去实现一个接口，而不是一个具体的类时。

```php
class UserModel
{
    protected $db;

    public function __construct(DatabaseInterface $db)
    {
        $this->db = $db;
    }
}
```

This is the purest form of Dependency Injection. Any external classes needed are injected into the class, either through the constructor, as shown here, or through a setter method.

这是一种纯粹的依赖注入的形式，任何需要的外部类都被注入到这个类中，以构造器形式或者setter方法的方式。

Within the core of CodeIgniter 4, constructor-based DI is used everywhere. While this has advantages for the developers of the framework, it has huge implications for you. No longer do you need `MY_*` classes to extend or replace core files. Now, you can simply create a new class that conforms to the Interface we're expecting, and ensure that class is passed in it's place. How you make sure it gets used instead of the original file requires a bit more story.

在CodeIgniter4核心理念中，基于构造器的DI无处不在。这对于框架的开发者是有一定优势，对于你也有巨大的影响。你不用再使用MY_* 类去扩展或替换核心文件。现在，你可以简单的创建一个符合接口要求的新类，同时保证类是被正确注入的。但如何确保它的使用替代了原始文件，就需要我们再深入的研究了。

## The Rise and Fall of the Container

If you spend any time at all reading up on "modern PHP" best practices, you'll always see a DI Container (sometimes called an Inversion of Control Container) used. Most of the major frameworks use one, including Symfony, Laravel, Nette, Zend, and most others. Because of this, my natural first reaction was to [create one](https://github.com/newmythmedia/di) for the framework. I thought it turned it pretty sweet. You could configure classes with their alias, and, through the magic of Reflection, it would examine the constructor and automatically insert a shared instance of any configured classes, or parameters. It worked great, and was pretty fast.

如果你花时间看一些所谓的“现代PHP”的最佳实例，你会发现很多DI的容器的使用，有些被叫做控制反转容器。很多主流框架都使用一种容器，包括Symfony，Laravel，NEtte， Zend等等。所以，我的本能第一反应就是为框架创造一个这样的容器。我认为这会将框架变得非常好。你可以用别名配置一些类，之后通过反射机制，检查构造器并自动插入任何一个配置过的类的实例或参数。

Then I read a [blog post by Anthony Ferrara](http://blog.ircmaxell.com/2015/11/simple-easy-risk-and-change.html) that was discussing the differences between simple and easy when it comes to programming, and recomended optimizing for simplicty. One section in particular hit a chord: "A simple example is the way many PHP frameworks have adopted 'Dependency Injection' . . . But what if instead of using this complex system, we just created a series of functions? Real code that you can debug and understand." Bam. This was shortly after I had written the container, thinking the problem solved. This comment gnawed inside of me for a couple of weeks.

At first, I tried to make excuses about why we needed that container. But as I looked back on the things that keep bringing me back to CodeIgniter over the last 10 years, I realized a big part of it was the simplicity of the framework. It was simple code, that made things simple to understand and trust. You didn't have to wade through 6 different abstractions to understand what was going on. So, I ripped the container out, replacing it with a simple class that was just "a series of functions".

And you know what? It works great. It even came with a few unintended benefits, the biggest being that backtraces during errors are MUCH more understandable now. I'm currently working on a project that's using Laravel and get so frustrated by the backtrace being full of 15 calls through Laravel's sub-structure before I can find my code, if I even can.

然后我读到了一篇Anthony Ferrara的博客，其中探讨了编程过程中的简洁与简单的差异，并建议以优化简洁性为主。有一个段落这样说，”一个简洁的例子就是许多PHP框架都采用了依赖注入这种方式，但如果我们只是创建一系列的方法而不是使用这个复杂的框架系统呢。真正可以调试和理解的代码。“，在我写完容器以后，思考并解决了这个疑问，这个评论纠结了我几个星期。

起初，我尝试去找一些关于我们为什么需要容器的理由。但当我回顾CodeIgniter这过去10年的历程，我想最重要的一个部分就是框架的简洁性。简洁的代码，可以使得人们更容易理解。你不用通过6种不同的抽象去理解这代码正在做什么。所以，我把容器剥离出来，用一些简洁的类进行了替换。

结果，这样整个框架工作的很好，甚至出现了一些额外的好处，一些回溯的错误更容易被人抓到和理解。我现在正在忙于一项使用Laravel框架的项目，其中我要通过15个回溯调用的Laravel子结构才能找到我自己的代码，我被这件事情搞到崩溃。

## Services

At the core of the way this whole thing ties together is the Services config file. While it's possible that the name may change, Services are simply other classes, and the config file tells how to call them. Almost all of the core has an entry here. So, a quick look at a couple of the Services methods to see how they work, and then we'll move on to a quick example of Services and DI as you'd use it in your application.

将所有东西关联起来的关键就在于Service的配置文件。然而，名称可能会变，Services只是一些其他的类，配置文件描述怎样调用他们。下面，我们来看一些Services的方法来看他是如何工作的，之后我们会继续看一个Services和DI的实例。

```php
/**
 * The Logger class is a PSR-3 compatible Logging class that supports
 * multiple handlers that process the actual logging.
 */
public static function logger($getShared = false)
{
    if (! $getShared)
    {
        return new \CodeIgniter\Log\Logger(new \Config\Logger());
    }

    return self::getSharedInstance('logger');
}
```

Here's a service at it's simplest. All of the methods allow you to get either a brand new instance of the class, or one that's shared among all other uses, which is a great option for things like this logger, where there's no real reason to waste memory on multiple instances. Assuming that you did want the shared instance, you'd simply call `$logger = Config\Servies::logger(true)`.

这是个Service的最简单的例子。所有的方法允许你获取一个全新的类的实例，或者一个已经在使用的共享实例，例如日志这种没有必要浪费内存在多个实例上的功能。假如你想要使用这个共享的实例，你就可以简单的调用$logger = Config\Servies::logger(true)

Because they are just simple methods, some of them support parameters to customize how the class works. For instance, the `renderer()` method, which handles displaying views, can take the path to the views folder as the first parameter.
因为他们只是一些简单的方法，其中有一些可以传一些参数去进行自定义。例如，renderer()这个用于显示层的方法，可以用路径作为第一个参数用于作为访问views文件夹的入口。


```php
/**
 * The Renderer class is the class that actually displays a file to the user.
 * The default View class within CodeIgniter is intentionally simple, but this
 * service could easily be replaced by a template engine if the user needed to.
 */
public static function renderer($viewPath = APPPATH.'Views/', $getShared = false)
{
    if (! $getShared)
    {
        return new \CodeIgniter\View\View($viewPath, self::loader(true), CI_DEBUG, self::logger(true));
    }

    return self::getSharedInstance('renderer');
}
```

If you want to replace the renderer with a simple solution to use a template engine like Twig, you'd create a small adapter class that implented the `CodeIgniter\View\RenderableInterface` and modify the Services::renderer() method to return an instance of your new adapter class. Then, you could call the `view()` command that is always available, and it would use Twig instead of the simple view-based solution that CodeIgniter provides. Couldn't be simpler.

Alright, you've seen how to define the services, and we've talked about why DI is a great thing, so it's time to take a look at how to use the two together in your own application.

如果你想要用像Twig这样的模板引擎去替换掉renderer这个方法，你可以创建一个实现CodeIgniter\View\RenderableInterface接口的适配器类，同时更改Services::renderer()方法去返回一个适配器类的实例对象。然后，你就能调用view()命令，它会使用Twig而不是原先的CodeIgniter提供的简单显示层实现。

这里你已经了解到了如何定义Services，同时，我们也探讨了为什么DI是一个好东西，下面我们就来看一下这两者是如何在你自己的应用里结合一起的。

## A Quick Example

> Using Dependency Injection in your applications is not required, though it's recommended. The framework itself uses these files, and they provide a simple way to modify the core, but any other use by yourself is optional.
> 在你的应用中使用依赖注入不是必须的，但是十分推荐的做法。框架本身使用了这些文件，同时他提供了简单的方法去修改核心的代码，但一些其他方面的使用就要靠你自己了。

With that disclaimer out of the way, let's look at our `UserModel` again. Assume that we're in the `Users` controller and you need to pull all active users. Earlier, we showed the UserModel taking a Database object in it's constructor. Ignoring the exact class names for now, getting a new instance of the model would be done something like this:
通过这种声明方式，让我们再看看看UserModel。假如我们在Users这个控制器中，你需要抓取所有的活跃用户。之前，我们提到过UserModel的构造器中注入一个数据库对象。忽略类名，现在我们获取一个新model实例的做法应该是这样：


```php
class Users extends \CodeIgniter\Controller
{
    public function index()
    {
        $model = new UserModel( Config\Services::database() );

        $data = [
            'users' => $model->findAll()
        ];

        echo view('users/list_all', $data);
    }
}
```

This way, if you ever change the database solution, you don't have to hunt around trying to find every location. Simply change it in the Services config file, and you're golden. If you're only using a single database connection, you could modify the `database()` service to insert the correct config, etc. Since there is actually a couple of parameters needed to create a database connection, you could make things even simpler and create a new service for your model if you needed to.

这样的话，如果你改变了数据库方案，你不用去到处寻找每一个数据库相关表位置。只需要简单的在Services配置文件中改一下就可以了。如果你只是使用一个单一的数据库连接，你可以修改数据库的service去插入正确的配置。既然我们需要一系列参数去创建一个数据库连接，你可以更轻松简洁的创建一个新的服务达到目的。

> Remember - all database stuff shown here is for example only, and doesn't reflect the end product in any fashion!

## Coupling?

As a keen observer, you might be yelling at me that this added services stuff just moves the coupling of these classes from the libraries, models, etc, to the controller. And you'd be correct. At some point, though, you have to be able to specify which classes you want to use.

And the truth is that is what a Controller's job is. It glues the other pieces together. It is the one piece of your application that should be tightly coupled with your framework. If you design the rest of your application correctly, it's about the only place that even knows about the framework you're using. And that's great. If, some years down the road, you need to switch frameworks for some reason (it happens, unfortunately), you will mostly just have to change the controllers.

Even better - this simple Services class further reduces your dependency on any specific framework. It's just a simple class, and could be used with any framework you wanted to use.

作为一个敏锐的观察者，你可能会对我有异议和疑问，这些新加的services不就是将耦合的类从库、模型等地方移到控制器中了么，恭喜你，答对了。但是在一些时候，你仍然要指定一些你要使用的类。

事实上，这就是一个Controller的工作，他将所有的模块拼接在一起。他们每一项都是你应用的一部分，并需要与框架紧密耦合在一起。如果你正确的设计了你应用的其余部分， it's about the only place that even knows about the framework you're using。

如果很多年过去，你想要换框架，你将只需要更改controller即可。

这个简单的Services设计的更好的减少了对于特定框架的依赖。他只是一个简单的类，而且他可以被用于任何你想要使用的框架。
