PF4J源码学习
=================

# 1. PF4J

&nbsp;&nbsp;&nbsp;&nbsp;PF4J文档地址：https://pf4j.org/ <br/>
&nbsp;&nbsp;&nbsp;&nbsp;PF4J开源GITHUB地址：https://github.com/pf4j/pf4j <br/>

# 2. 插件管理器PluginManager

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

    // 插件基本信息提取器
    pluginDescriptorFinder = createPluginDescriptorFinder();

    // 插件扩展点提取器
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

### 2.2.2 插件仓库PluginRepository

&nbsp;&nbsp;&nbsp;&nbsp;插件仓库记录了插件可以通过那些载体记录，默认插件管理器DefaultPluginManager为例说明，默认插件构建器通过组合方式来构建插件仓库。1）JarPluginRepository仓库：在根目录下，以JAR结尾的文件；2）DefaultPluginRepository仓库：在根目录下，目录文件下的所有非隐藏文件；3）开发模式下，会使用DevelopmentPluginRepository仓库，其含义为：在target或build目录下的所有非隐藏文件。<br/>
<ul>
    PluginRepository
    <ul>
        BasePluginRepository
        <ul>DefaultPluginRepository</ul>
        <ul>DevelopmentPluginRepository</ul>
        <ul>JarPluginRepository</ul>
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

### 2.2.3 插件工厂PluginFactory

&nbsp;&nbsp;&nbsp;&nbsp;默认提供的插件工厂为DefaultPluginFactory。

### 2.2.4 插件扩展点创建工厂ExtensionFactory

&nbsp;&nbsp;&nbsp;&nbsp;插件扩展点工厂提供两个实现SingletonExtensionFactory和DefaultExtensionFactory，其中SingletonExtensionFactory工厂为每个扩展点类仅创建一个实例，根据业务需要灵活选择。

<ul>
    ExtensionFactory
    <ul>
        DefaultExtensionFactory
        <ul>
            SingletonExtensionFactory
        </ul>
    </ul>
</ul>

### 2.2.5 插件基本信息提取器PluginDescriptorFinder

&nbsp;&nbsp;&nbsp;&nbsp;插件基本信息内置的存储方式有两种：1）properties文件；2）jar的manifest文件。默认的插件管理器使用组合的方式构建插件基本信息提取器。

<ul>
    PluginDescriptorFinder
    <ul>CompoundPluginDescriptorFinder</ul>
    <ul>ManifestPluginDescriptorFinder</ul>
    <ul>PropertiesPluginDescriptorFinder</ul>
</ul>

~~~
protected PluginDescriptorFinder createPluginDescriptorFinder() {
    return new CompoundPluginDescriptorFinder()
        .add(new PropertiesPluginDescriptorFinder())
        .add(new ManifestPluginDescriptorFinder());
}
~~~

### 2.2.6 插件扩展点提取器ExtensionFinder

&nbsp;&nbsp;&nbsp;&nbsp;DefaultExtensionFinder使用组合方式遍历所有插件扩展点提取器，其中DefaultExtensionFinder默认添加了LegacyExtensionFinder提取器，也可以手动添加ServiceProviderExtensionFinder提取器，只需要调用方法addServiceProviderExtensionFinder即可。

&nbsp;&nbsp;&nbsp;&nbsp;LegacyExtensionFinder含义为：插件打包时，会将所有扩展点打包到META-INF/extensions.idx文件中，每行记录一个扩展点实现，LegacyExtensionFinder是读取extensions.idx文件中的所有扩展点实现类列表。

&nbsp;&nbsp;&nbsp;&nbsp;ServiceProviderExtensionFinder含义为：插件打包时，会将所有扩展点打包写到META-INF/services/{扩展点接口类}文件中，每行记录一个扩展点实现，ServiceProviderExtensionFinder读取所有META-INF/services/XXX文件下的扩展点实现类列表。

<ul>
    ExtensionFinder
    <ul>DefaultExtensionFinder</ul>
    <ul>
        AbstractExtensionFinder
        <ul>LegacyExtensionFinder</ul>
        <ul>ServiceProviderExtensionFinder</ul>
    </ul>
</ul>

### 2.2.7 插件状态判断PluginStatusProvider

&nbsp;&nbsp;&nbsp;&nbsp;插件状态判断是使用PluginStatusProvider判断，内置提供了DefaultPluginStatusProvider实现，该实现读取插件root目录下的enabled.txt和disabled.txt文件来判断插件状态。

### 2.2.8 插件加载器PluginLoader

&nbsp;&nbsp;&nbsp;&nbsp;默认的插件加载器是组合多种加载器实现的，类层次如下，其中DefaultPluginLoader与DevelopmentPluginLoader只是定义了加载插件的class位置不同而已，DevelopmentPluginLoader开发阶段使用，DefaultPluginLoader非开发阶段使用。JarPluginLoader只是一种简化的插件加载器实现，仅处理标准的JAR插件。

<ul>
    PluginLoader
    <ul>CompoundPluginLoader</ul>
    <ul>JarPluginLoader</ul>
    <ul>
        BasePluginLoader
        <ul>DefaultPluginLoader</ul>
        <ul>DevelopmentPluginLoader</ul>
    </ul>
</ul>

### 2.2.9 版本管理VersionManager

&nbsp;&nbsp;&nbsp;&nbsp;提供了版本判断的实现。checkVersionConstraint可以校验如下，比如：checkVersionConstraint("1.4.3", ">2.0.0")返回false，checkVersionConstraint("1.4.3", ">=1.4.0 & <1.6.0")返回true等。compareVersions判断版本前后，返回大于、小于、等于0分别判断，用于排序。

~~~
public interface VersionManager {

    boolean checkVersionConstraint(String version, String constraint);

    int compareVersions(String v1, String v2);

}
~~~

### 2.3.0 依赖管理

&nbsp;&nbsp;&nbsp;&nbsp;用于将插件版本依赖关系构建成一张依赖关系图，用于控制插件的加载顺序。

## 2.3 插件加载

~~~
public void loadPlugins() {    
    // 检查插件根目录
    if (Files.notExists(pluginsRoot) || !Files.isDirectory(pluginsRoot)) {
        log.warn("No '{}' root", pluginsRoot);
        return;
    }

    // 获取所有扩展插件路径
    List<Path> pluginPaths = pluginRepository.getPluginPaths();
    if (pluginPaths.isEmpty()) {
        log.info("No plugins");
        return;
    }

    // 遍历插件路径，依次加载插件
    for (Path pluginPath : pluginPaths) {
        try {
            loadPluginFromPath(pluginPath);
        } catch (PluginRuntimeException e) {
            log.error(e.getMessage(), e);
        }
    }

    // 构建插件依赖关系图，判断插件依赖是否存在问题（比如循环依赖问题、依赖某插件不存在、依赖插件版本不对等），然后根据插件依赖关系图决定加载插件扩展点加载顺序
    try {
        resolvePlugins();
    } catch (PluginRuntimeException e) {
        log.error(e.getMessage(), e);
    }
}
~~~

### 2.3。1 根据插件路径加载该插件

~~~
protected PluginWrapper loadPluginFromPath(Path pluginPath) {
    // 获取插件标识，第一次加载时候插件标识获取不到，返回null
    String pluginId = idForPath(pluginPath);
    if (pluginId != null) {
        throw new PluginAlreadyLoadedException(pluginId, pluginPath);
    }

    // 插件基本信息提取器
    PluginDescriptorFinder pluginDescriptorFinder = getPluginDescriptorFinder();
    // 获取插件基本信息
    PluginDescriptor pluginDescriptor = pluginDescriptorFinder.find(pluginPath);
    // 校验插件基本信息
    validatePluginDescriptor(pluginDescriptor);

    // 获取插件唯一标识
    pluginId = pluginDescriptor.getPluginId();
    // 判断唯一性
    if (plugins.containsKey(pluginId)) {
        PluginWrapper loadedPlugin = getPlugin(pluginId);
        throw new PluginRuntimeException("There is an already loaded plugin ({}) "
                + "with the same id ({}) as the plugin at path '{}'. Simultaneous loading "
                + "of plugins with the same PluginId is not currently supported.\n"
                + "As a workaround you may include PluginVersion and PluginProvider "
                + "in PluginId.",
            loadedPlugin, pluginId, pluginPath);
    }

    // 插件Class
    String pluginClassName = pluginDescriptor.getPluginClass();

    // 插件Class对应的ClassLoader构建，每个插件对应一个独立的ClassLoader，并设置该ClassLoader对应的class路径
    ClassLoader pluginClassLoader = getPluginLoader().loadPlugin(pluginPath, pluginDescriptor);

    // 构建插件包装类
    PluginWrapper pluginWrapper = new PluginWrapper(this, pluginDescriptor, pluginPath, pluginClassLoader);
    // 设置插件工厂
    pluginWrapper.setPluginFactory(getPluginFactory());

    // 读取插件状态，若是无效，则设置包装类为无效状态
    if (isPluginDisabled(pluginDescriptor.getPluginId())) {
        log.info("Plugin '{}' is disabled", pluginPath);
        pluginWrapper.setPluginState(PluginState.DISABLED);
    }

    // 校验插件，版本
    if (!isPluginValid(pluginWrapper)) {
        log.warn("Plugin '{}' is invalid and it will be disabled", pluginPath);
        pluginWrapper.setPluginState(PluginState.DISABLED);
    }

    log.debug("Created wrapper '{}' for plugin '{}'", pluginWrapper, pluginPath);

    pluginId = pluginDescriptor.getPluginId();

    // add plugin to the list with plugins
    plugins.put(pluginId, pluginWrapper);
    getUnresolvedPlugins().add(pluginWrapper);

    // add plugin class loader to the list with class loaders
    getPluginClassLoaders().put(pluginId, pluginClassLoader);

    return pluginWrapper;
}
~~~ 