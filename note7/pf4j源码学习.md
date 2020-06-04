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

    pluginRepository = createPluginRepository();
    pluginFactory = createPluginFactory();
    extensionFactory = createExtensionFactory();
    pluginDescriptorFinder = createPluginDescriptorFinder();
    extensionFinder = createExtensionFinder();
    pluginStatusProvider = createPluginStatusProvider();
    pluginLoader = createPluginLoader();

    versionManager = createVersionManager();
    dependencyResolver = new DependencyResolver(versionManager);
}
~~~
