title: 【Springloc流程解析】--1. Springloc的基本组件和流程

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/cs/cs-20.jpg

toc: true 

tags:

  - 源码
  - spring
  - loc

categories: 

  - [源码,spring] 

date: 2020-04-28 19:22:34

---

## 概述

​		本章开始解析SpringLoc的执行流程，我认为要搞懂spring的源码，最好的办法不是立即针对核心功能进行解析，而是我们首先要了解SpringLoc执行的步骤，纵观全局，最后再对单独的步骤逐个击破，最终串起SpringLoc的整个执行逻辑。<!--more -->

​		解析代码之前我们先来回顾一下，我们平时了解的SpringLoc步骤。我们知道SpringLoc是一个**容器**，根据我们定义bean信息的xml文件(此处不讨论注解)，最终来解析成一个个的**bean对象**存放到容器中，在我们需要使用到某个bean时，springloc会自动帮我们注入，当然我们也可以通过一个ApplicationContext的getBean("myBean")来获取一个我们需要的bean对象，那么从xml的读取转为bean，到最后我们成为一个可供我们使用的bean对象，这其中经历了哪些步骤呢？就让我们一起来看看吧！

## 重要组件

### Resource

`org.springframework.core.io.Resource`，资源访问定位。由于springloc支持以不同的方式、从不同的位置进行资源的获取，因此通过`Resource`类将资源抽象，而它的每一个实现类都代表了一种资源的访问策略，如`ClassPathResource`、`FileSystemResource`、`ServletContextResource`它们分别从不同的位置获取资源。

- `FileSystemResource`：以文件系统绝对路径的方式进行访问

- `ClassPathResource`：以类路径的方式进行访问
- `ServletContextResource`：以相对于Web应用根目录的方式进行访问。

### ResourceLoader

`org.springframework.core.io.ResourceLoader`，有了资源，我们就需要加载资源。`ResourceLoader`的作用就是用来进行资源的加载。

```java
public interface ResourceLoader {
    //默认在classpath路径下，寻找文件加载
    String CLASSPATH_URL_PREFIX = "classpath:";

    //获取资源，返回一个Resource对象
    Resource getResource(String var1);

    //获取当前的类加载器
    @Nullable
    ClassLoader getClassLoader();
}

//根据不同的定位路径，选择不同的策略
public Resource getResource(String location) {
    Assert.notNull(location, "Location must not be null");
    Iterator var2 = this.getProtocolResolvers().iterator();

    Resource resource;
    do {
        if (!var2.hasNext()) {
            if (location.startsWith("/")) {
                return this.getResourceByPath(location);
            }

            if (location.startsWith("classpath:")) {
                return new ClassPathResource(location.substring("classpath:".length()), this.getClassLoader());
            }

            try {
                URL url = new URL(location);
                return (Resource)(ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
            } catch (MalformedURLException var5) {
                return this.getResourceByPath(location);
            }
        }

        ProtocolResolver protocolResolver = (ProtocolResolver)var2.next();
        resource = protocolResolver.resolve(location, this);
    } while(resource == null);

    return resource;
}
```

`ResourceLoader`中只定义两个方法，`getResource()`根据给定的资源文件地址返回一个对应的`Resource`，`getClassLoader()`获取类加载器。

### BeanDefinition

`org.springframework.beans.factory.config.BeanDefinition`，它并不是我们使用的bean，而是用于描述bean的信息，里面存放着的是bean的元数据。相当于`BeanDefinition`是bean的一个模板，我们通过模板来创建的bean具有`BeanDefinition`的基础信息，当然我们也可以对将要创建的bean进行其他的扩展，自己定制不同特性的bean对象。

### BeanDefinitionReader

`org.springframework.beans.factory.support.BeanDefinitionReader`，用于读取配置的资源文件，并将其转化成容器中的`BeanDefinition`。`BeanDefinitionReader`是一个接口，类似于上面的`Resource`抽象资源，实现`BeanDefinitionReader`的子类，代表着不同的读取策略。而我们本系列的解析主要是针对`XmlBeanDefinitionReader`来进行讲解的。



### BeanFactory

`org.springframework.beans.factory.BeanFactory`，顾名思义bean工厂，用来存储`BeanDefinition`信息，以及创建并存储bean对象。前面说过通过`BeanDefinition`当做模板来进行bean的创建，当我们需要创建大量的bean对象时，使用工厂的方式来进行创建也就顺理成章了。通过实现`BeanFactory`我们就可以通过`BeanDefinition`创建不同的特性的bean对象了。`BeanFactory`在`springloc`中极为重要，实现了`BeanFactory`的接口也是非常之多，我们也不在这里一一列举了。

![](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/BeanFactory.png)

`DefaultListableBeanFactory`几乎继承了所有其他`BeanFactory`父接口，是一个集大成者的`BeanFactory`，涵盖了`BeanFactory`中的大部分功能，是功能最全的一个`BeanFactory`，同时继承自`AliasRegistry`具有别名注册的功能。

- `AliasRegistry`：提供别名注册的接口。
- `BeanDefinitionRegistry`：提供对`BeanDefinition`注册的接口。
- `SimpleAliasRegistry`：实现了`AliasRegistry`，使用map作为alias的缓存。
- `SingletonBeanRegistry`: 提供单例bean注册的接口。
- `DefaultSingletonBeanRegistry`：继承自`SimpleAliasRegistry`、实现了`SingletonBeanRegistry`接口，因此它同时具有注册别名和单例bean的功能。
- `HierarchicalBeanFactory`：实现了bean工厂的分层，可以将各个`BeanFactory`设置为父子关系，同时提供了父容器的访问功能。
- `ListableBeanFactory`：提供了批量获取bean实例的方法。
- `FactoryBeanRegistrySupport`：继承自`DefaultSingletonBeanRegistry`，增加了对`FactoryBean`的处理功能。
- `ConfigurableBeanFactory`：提供了配置各种bean的方法。
- `AbstractBeanFactory`：综合了`FactoryBeanRegistrySupport`与`ConfigurableBeanFactory`的功能。
- `AutowireCapableBeanFactory`：继承自`BeanFactory`，提供创建bean、自动注入、初始化以及应用bean的后置处理器。
- `AbstractAutowireCapableBeanFactory`：综合了`AbstractBeanFactory`与`AutowireCapableBeanFactory`的功能。
- `ConfigurableListableBeanFactory`：`BeanFactory`的配置清单，指定忽略类型以及接口。
- `DefaultListableBeanFactory`综合上面的所有功能，主要是对bean注册后的处理。

### ApplicationContext

`org.springframework.context.ApplicationContext`，Spring的应用上下文，`ApplicationContext`继承自`BeanFactory`，同时还继承了其他丰富的组件，基本上`ApplicationContext`将Springloc中的所有组件功能都组合到一起了。 

通过实现`ApplicationContext`提供不同的上下文环境。

- `ClassPathXmlApplicationContext`：从类路径下的一个或多个xml配置文件中加载上下文定义，适用于xml配置的方式，也就是我们本系列需要讲解的重点。
- `AnnotationConfigApplicationContext`：从一个或多个基于java的配置类中加载上下文定义，适用于java注解的方式。
- `FileSystemXmlApplicationContext`：从文件系统下的一个或多个xml配置文件中加载上下文定义，也就是说系统盘符中加载xml配置文件。
- `XmlWebApplicationContext`： 从web应用下的一个或多个xml配置文件加载上下文定义，适用于xml配置方式。后面我们讲解Springmvc的时候会讲到。
- `AnnotationConfigWebApplicationContext`：专门为web应用准备的，适用于注解方式。



```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    ApplicationContext context = new ClassPathXmlApplicationContext("testA.xml");
    TestA testA = (TestA)context.getBean("testA");
}
```

通过创建一个`ClassPathXmlApplicationContext`通过传入配置bean信息的xml文件名称，最终返回一个`ApplicationContext`对象。我们通过`ApplicationContext`对象的`getBean()`方法，就可以获取到我们想要的对象，而不用我们自己通过new方法去实例化一个对象，那么spring是怎么做的，这其中又经历了哪些方法，就让我们从这段代码开始。

​		我们先来看看`ClassPathXmlApplicationContext`的结构图。

![](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/ClassPathXmlApplicationContext.png)`ClassPathXmlApplicationContext`到顶层`BeanFactory`、`ResourceLoader`继承关系较为复杂，让我们从最顶层的`BeanFactory`与`ResourceLoader`来介绍每个类的特点与作用。

## 流程解析

从我们的 `new ClassPathXmlApplicationContext("testA.xml")`方法开始说起

```java
public class ClassPathXmlApplicationContext extends AbstractXmlApplicationContext {
    
    @Nullable
    private Resource[] configResources;

    public ClassPathXmlApplicationContext() {
    }

	//将传入的ApplicationContext对象设置为当前的父类
    public ClassPathXmlApplicationContext(ApplicationContext parent) {
        super(parent);
    }

    //我们上面调用的构造方法
    public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
        this(new String[]{configLocation}, true, (ApplicationContext)null);
    }

    //传入多个XML配置的构造方法
    public ClassPathXmlApplicationContext(String... configLocations) throws BeansException {
        this(configLocations, true, (ApplicationContext)null);
    }

    //传入多个XML配置，并指定父类ApplicationContext
    public ClassPathXmlApplicationContext(String[] configLocations, @Nullable ApplicationContext parent) throws BeansException {
        this(configLocations, true, parent);
    }

    //传入多个XML配置，选择是否需要刷新容器
    public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh) throws BeansException {
        this(configLocations, refresh, (ApplicationContext)null);
    }

    //上面的构造方法最终都要调用此方法
    public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, @Nullable ApplicationContext parent) throws BeansException {
        //调用父类构造方法，最终在AbstractApplicationContext构造方法中完成构建
        super(parent);
        //根据提供的路径，解析成配置文件数组
        this.setConfigLocations(configLocations);
        if (refresh) {
            //刷新容器
            this.refresh();
        }
    }
	//省略一些方法.....
    //.......
}
```

`ClassPathXmlApplicationContext`中的`super(parent)`，一直到`AbstractApplicationContext`中完成构造工作。

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader implements ConfigurableApplicationContext {
    public AbstractApplicationContext() {
        this.logger = LogFactory.getLog(this.getClass());
        //生成id
        this.id = ObjectUtils.identityToString(this);
        //生成展示名
        this.displayName = ObjectUtils.identityToString(this);
        //初始化beanFactory的后置处理集合
        this.beanFactoryPostProcessors = new ArrayList();
        this.active = new AtomicBoolean();
        this.closed = new AtomicBoolean();
        this.startupShutdownMonitor = new Object();
        //初始化监听器
        this.applicationListeners = new LinkedHashSet();
        //初始化资源解析器
        this.resourcePatternResolver = this.getResourcePatternResolver();
    }
}
```

再来看看`setConfigLocations`方法中做了什么。

### setConfigLocations方法

```java
//AbstractRefreshableConfigApplicationContext #setConfigLocations
public void setConfigLocations(@Nullable String... locations) {
    if (locations != null) {
        Assert.noNullElements(locations, "Config locations must not be null");
        this.configLocations = new String[locations.length];
        for(int i = 0; i < locations.length; ++i) {
            //通过resolvePath解析路径并返回
            this.configLocations[i] = this.resolvePath(locations[i]).trim();
        }
    } else {
        this.configLocations = null;
    }
}
//AbstractRefreshableConfigApplicationContext #setConfigLocations
protected String resolvePath(String path) {
    //获取一个ConfigurableEnvironment对象，用来解析占位符
    return this.getEnvironment().resolveRequiredPlaceholders(path);
}

//
public ConfigurableEnvironment getEnvironment() {
    if (this.environment == null) {
        //如果environment不存在，那么创建一个
        this.environment = this.createEnvironment();
    }

    return this.environment;
}

protected ConfigurableEnvironment createEnvironment() {
    return new StandardEnvironment();
}

public class StandardEnvironment extends AbstractEnvironment {
    public static final String SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME = "systemEnvironment";
    public static final String SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME = "systemProperties";

    public StandardEnvironment() {
    }

    protected void customizePropertySources(MutablePropertySources propertySources) {
        propertySources.addLast(new PropertiesPropertySource("systemProperties", this.getSystemProperties()));
        propertySources.addLast(new SystemEnvironmentPropertySource("systemEnvironment", this.getSystemEnvironment()));
    }
}
```

未完待续。。。



















