PF4J源码学习
=================

# 1. PF4J

&nbsp;&nbsp;&nbsp;&nbsp;PF4J文档地址：https://pf4j.org/ <br/>
&nbsp;&nbsp;&nbsp;&nbsp;PF4J开源GITHUB地址：https://github.com/pf4j/pf4j <br/>

# 2. PluginManager

&nbsp;&nbsp;&nbsp;&nbsp;负责插件的加载、启动、停止和插件的使用等，是PF4J使用的核心入口。

## 2.1 应用程序中使用PF4J使用步骤

&nbsp;&nbsp;&nbsp;&nbsp;在您的应用程序中添加PF4J非常简单：

~~~
public static void main(String[] args) {
    ...

    // create the plugin manager
    PluginManager pluginManager = new JarPluginManager(); // or "new ZipPluginManager() / new DefaultPluginManager()"
    
    // start and load all plugins of application
    pluginManager.loadPlugins();
    pluginManager.startPlugins();

    // retrieve all extensions for "Greeting" extension point
    List<Greeting> greetings = pluginManager.getExtensions(Greeting.class);
    for (Greeting greeting : greetings) {
        System.out.println(">>> " + greeting.getGreeting());
    }
    
    // stop and unload all plugins
    pluginManager.stopPlugins();
    pluginManager.unloadPlugins();
    
    ...
}
~~~

## 2.2 插件的初始化（实例化）

&nbsp;&nbsp;&nbsp;&nbsp;以DefaultPluginManager为例介绍，自带的几类插件管理器的区别在介绍初始化中可以明显看出来啥区别，主要体现在扩展点的插件打包方式在那种地方、如何加载这些打包好的插件，其他的逻辑完全一致，插件初始化方法如下：<br/>

~~~
protected void initialize() {
    // Map<String, PluginWrapper>插件映射表
    plugins = new HashMap<>();

    // Map<String, ClassLoader>每个插件对应自己不同的ClassLoader
    pluginClassLoaders = new HashMap<>();

    // 记录不同状态的插件列表
    unresolvedPlugins = new ArrayList<>();
    resolvedPlugins = new ArrayList<>();
    startedPlugins = new ArrayList<>();

    // 订阅插件状态变化消息处理器
    pluginStateListeners = new ArrayList<>();

    // 插件根目录
    if (pluginsRoot == null) {
        pluginsRoot = createPluginsRoot();
    }

    // 创建插件仓库（插件可以从那些载体获取）
    pluginRepository = createPluginRepository();

    // 创建插件工厂
    pluginFactory = createPluginFactory();

    // 创建扩展点工厂
    extensionFactory = createExtensionFactory();

    // 插件基本信息获取器
    pluginDescriptorFinder = createPluginDescriptorFinder();

    // 插件扩展点获取器
    extensionFinder = createExtensionFinder();

    // 插件状态（禁用、启用那些插件，通过配置文件定义）
    pluginStatusProvider = createPluginStatusProvider();

    // 创建插件加载器
    pluginLoader = createPluginLoader();

    // 版本校验与管理
    versionManager = createVersionManager();

    // 插件依赖管理
    dependencyResolver = new DependencyResolver(versionManager);
}
~~~

### 2.2.1 插件根目录

&nbsp;&nbsp;&nbsp;&nbsp;构建插件管理器PluginManager可以传入根目录，若没有传入，则可以根据下面逻辑获取：插件根目录优先从系统属性（pf4j.pluginsDir）中获取，若没有设置，则默认从plugins目录获取。

### 2.2.2 插件仓库

&nbsp;&nbsp;&nbsp;&nbsp;插件仓库记录了插件可以通过那些载体记录，默认插件管理器DefaultPluginManager为例说明，默认插件构建器通过组合方式来构建插件仓库。1）JarPluginRepository仓库：在根目录下，以JAR结尾的文件；2）DefaultPluginRepository仓库：在根目录下，目录文件下的所有非隐藏文件；3）开发模式下，会使用DevelopmentPluginRepository仓库，其含义为：在target或build目录下的所有非隐藏文件。<br/>

&nbsp;&nbsp;&nbsp;&nbsp;类层次如下:<br/>

<ul>
    PluginRepository
    <ul>
        BasePluginRepository
        <ul>
            <li>DefaultPluginRepository</li>
            <li>DevelopmentPluginRepository</li>
            <li>JarPluginRepository</li>
        </ul>
    </ul>
    <ul>CompoundPluginRepository</ul>
</ul>

~~~
protected PluginRepository createPluginRepository() {
    return new CompoundPluginRepository()
        .add(new DevelopmentPluginRepository(getPluginsRoot()), this::isDevelopment)
        .add(new JarPluginRepository(getPluginsRoot()), this::isNotDevelopment)
        .add(new DefaultPluginRepository(getPluginsRoot()), this::isNotDevelopment);
}
~~~


### 2.2.3 插件工厂
