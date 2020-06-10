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

### 2.2.10 依赖管理

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

### 2.3.1 根据插件路径加载该插件

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

    pluginId = pluginDescriptor.getPluginId();

    plugins.put(pluginId, pluginWrapper);
    getUnresolvedPlugins().add(pluginWrapper);

    getPluginClassLoaders().put(pluginId, pluginClassLoader);

    return pluginWrapper;
}
~~~ 


### 2.3。1 插件依赖解决

~~~
 protected void resolvePlugins() {
    // 待加载的所有插件基本信息
    List<PluginDescriptor> descriptors = new ArrayList<>();
    for (PluginWrapper plugin : plugins.values()) {
        descriptors.add(plugin.getDescriptor());
    }

    // 插件版本依赖解决（构建插件依赖关系图）
    DependencyResolver.Result result = dependencyResolver.resolve(descriptors);

    // 判断是否有循环依赖
    if (result.hasCyclicDependency()) {
        throw new DependencyResolver.CyclicDependencyException();
    }

    // 判断是否存在没有找到的依赖插件
    List<String> notFoundDependencies = result.getNotFoundDependencies();
    if (!notFoundDependencies.isEmpty()) {
        throw new DependencyResolver.DependenciesNotFoundException(notFoundDependencies);
    }

    // 判断是否存在错误的版本
    List<DependencyResolver.WrongDependencyVersion> wrongVersionDependencies = result.getWrongVersionDependencies();
    if (!wrongVersionDependencies.isEmpty()) {
        throw new DependencyResolver.DependenciesWrongVersionException(wrongVersionDependencies);
    }

    // 根据插件依赖关系构建加载插件的顺序
    List<String> sortedPlugins = result.getSortedPlugins();

    // 按照顺序加载插件
    for (String pluginId : sortedPlugins) {
        PluginWrapper pluginWrapper = plugins.get(pluginId);
        if (unresolvedPlugins.remove(pluginWrapper)) {
            PluginState pluginState = pluginWrapper.getPluginState();
            if (pluginState != PluginState.DISABLED) {
                pluginWrapper.setPluginState(PluginState.RESOLVED);
            }

            resolvedPlugins.add(pluginWrapper);

            firePluginStateEvent(new PluginStateEvent(this, pluginWrapper, pluginState));
        }
    }
}
~~~

&nbsp;&nbsp;&nbsp;&nbsp;构建插件依赖关系图的逻辑如下：

~~~
public Result resolve(List<PluginDescriptor> plugins) {
    dependenciesGraph = new DirectedGraph<>();
    dependentsGraph = new DirectedGraph<>();

    // 填充依赖关系图
    Map<String, PluginDescriptor> pluginByIds = new HashMap<>();
    for (PluginDescriptor plugin : plugins) {
        // 填充依赖关系图节点
        addPlugin(plugin);
        // 映射表构建
        pluginByIds.put(plugin.getPluginId(), plugin);
    }

    // 拓扑排序
    List<String> sortedPlugins = dependenciesGraph.reverseTopologicalSort();

    Result result = new Result(sortedPlugins);

    resolved = true;

    // 当存在循环依赖时，sortedPlugins为空
    if (sortedPlugins != null) {
        for (String pluginId : sortedPlugins) {
            // 依赖插件不存在的记录
            if (!pluginByIds.containsKey(pluginId)) {
                result.addNotFoundDependency(pluginId);
            }
        }
    }
    
    // 校验依赖版本
    for (PluginDescriptor plugin : plugins) {
        String pluginId = plugin.getPluginId();
        String existingVersion = plugin.getVersion();

        List<String> dependents = new ArrayList<>(getDependents(pluginId));
        while (!dependents.isEmpty()) {
            String dependentId = dependents.remove(0);
            PluginDescriptor dependent = pluginByIds.get(dependentId);
            String requiredVersion = getDependencyVersionSupport(dependent, pluginId);
            boolean ok = checkDependencyVersion(requiredVersion, existingVersion);
            if (!ok) {
                result.addWrongDependencyVersion(new WrongDependencyVersion(pluginId, dependentId, existingVersion, requiredVersion));
            }
        }
    }

    return result;
}

private void addPlugin(PluginDescriptor descriptor) {
    String pluginId = descriptor.getPluginId();
    List<PluginDependency> dependencies = descriptor.getDependencies();
    if (dependencies.isEmpty()) {
        // 若当前插件没有依赖其他插件，添加节点
        dependenciesGraph.addVertex(pluginId);
        dependentsGraph.addVertex(pluginId);
    } else {
        boolean edgeAdded = false;
        for (PluginDependency dependency : dependencies) {
            if (!dependency.isOptional()) {
                edgeAdded = true;
                // 添加边（若节点不存在，先添加节点）
                dependenciesGraph.addEdge(pluginId, dependency.getPluginId());
                dependentsGraph.addEdge(dependency.getPluginId(), pluginId);
            }
        }

        // 可有可无的依赖，只添加节点，依赖边不添加
        if (!edgeAdded) {
            dependenciesGraph.addVertex(pluginId);
            dependentsGraph.addVertex(pluginId);
        }
    }
}
~~~

## 2.4 插件激活启动

&nbsp;&nbsp;&nbsp;&nbsp;该方法仅执行插件的启动方法，并设置插件状态为激活状态，后续获取插件扩展时候才会具体加载构建。

~~~
public void startPlugins() {
    for (PluginWrapper pluginWrapper : resolvedPlugins) {
        PluginState pluginState = pluginWrapper.getPluginState();
        if ((PluginState.DISABLED != pluginState) && (PluginState.STARTED != pluginState)) {
            try {
                // 执行插件启动方法
                pluginWrapper.getPlugin().start();
                pluginWrapper.setPluginState(PluginState.STARTED);
                pluginWrapper.setFailedException(null);
                startedPlugins.add(pluginWrapper);
            } catch (Exception e) {
                pluginWrapper.setPluginState(PluginState.FAILED);
                pluginWrapper.setFailedException(e);
            } finally {
                firePluginStateEvent(new PluginStateEvent(this, pluginWrapper, pluginState));
            }
        }
    }
}
~~~

## 2.5 插件扩展点实现获取

&nbsp;&nbsp;&nbsp;&nbsp;该方法为正式使用插件扩展点实现，以其中某一个方法分析开始，即分析"<T> List<T> getExtensions(Class<T> type)"该方法。

## 2.5.1 根据插件扩展接口获取所有扩展点实现

~~~
public <T> List<T> getExtensions(Class<T> type) {
    // 1. 先获取插件接口对应的所有扩展点实现Wapper
    // 2. 根据Wapper创建扩展点实现类
   return getExtensions(extensionFinder.find(type));
}
~~~

## 2.5.2 根据扩展点接口获取所有Wapper

~~~
public <T> List<ExtensionWrapper<T>> find(Class<T> type) {
    List<ExtensionWrapper<T>> extensions = new ArrayList<>();
    for (ExtensionFinder finder : finders) {
        // 遍历所有ExtensionFinder，获取扩展点实现Wapper
        extensions.addAll(finder.find(type));
    }

    return extensions;
}
~~~

~~~
public <T> List<ExtensionWrapper<T>> find(Class<T> type) {
    // 加载所有扩展点实现类
    Map<String/*pluginId*/, Set<String/*扩展点实现类*/>> entries = getEntries();
    List<ExtensionWrapper<T>> result = new ArrayList<>();

    for (String pluginId : entries.keySet()) {
        // 根据插件ID和扩展点接口获取所有Wapper
        List<ExtensionWrapper<T>> pluginExtensions = find(type, pluginId);
        result.addAll(pluginExtensions);
    }

    Collections.sort(result);

    return result;
}
~~~

## 2.5.3 加载所有扩展点实现类

~~~
private Map<String, Set<String>> getEntries() {
    if (entries == null) {
        entries = readStorages();
    }

    return entries;
}

private Map<String, Set<String>> readStorages() {
    Map<String, Set<String>> result = new LinkedHashMap<>();

    // 加载所有finder classpath的扩展点实现类（即META-INF/extensions.idx记录的和META-INF/services记录的，这个根据PluginManager配置的finder有关系）
    result.putAll(readClasspathStorages());
    // 加载所有finder 插件记录的扩展点实现类（即META-INF/extensions.idx记录的和META-INF/services记录的，这个根据PluginManager配置的finder有关系）
    result.putAll(readPluginsStorages());

    return result;
}
~~~

## 2.5.4 根据插件ID和扩展点接口获取所有Wapper

~~~
public <T> List<ExtensionWrapper<T>> find(Class<T> type, String pluginId) {
    List<ExtensionWrapper<T>> result = new ArrayList<>();

    // 获取该插件所有扩展点实现类
    Set<String> classNames = findClassNames(pluginId);
    if (classNames == null || classNames.isEmpty()) {
        return result;
    }

    if (pluginId != null) {
        PluginWrapper pluginWrapper = pluginManager.getPlugin(pluginId);
        if (PluginState.STARTED != pluginWrapper.getPluginState()) {
            return result;
        }
    } else {
    }

    // 获取该插件的类加载器，若插件ID=null，那么该插件定义在PluginManager工程中，那么用应用的类加载器即可
    ClassLoader classLoader = (pluginId != null) ? pluginManager.getPluginClassLoader(pluginId) : getClass().getClassLoader();

    for (String className : classNames) {
        try {
            if (isCheckForExtensionDependencies()) {
                // 如果使用可选依赖项，则类加载器可能无法加载扩展点信息，此时可以使用asm方式先加载

                // 通过asm方式获取插件扩展点所有信息
                ExtensionInfo extensionInfo = getExtensionInfo(className, classLoader);
                if (extensionInfo == null) {
                    continue;
                }

                // 记录没有激活的插件ID
                List<String> missingPluginIds = new ArrayList<>();
                for (String requiredPluginId : extensionInfo.getPlugins()) {
                    PluginWrapper requiredPlugin = pluginManager.getPlugin(requiredPluginId);
                    if (requiredPlugin == null || !PluginState.STARTED.equals(requiredPlugin.getPluginState())) {
                        missingPluginIds.add(requiredPluginId);
                    }
                }
                
                // 打印没有激活的插件ID信息
                if (!missingPluginIds.isEmpty()) {
                    StringBuilder missing = new StringBuilder();
                    for (String missingPluginId : missingPluginIds) {
                        if (missing.length() > 0) missing.append(", ");
                        missing.append(missingPluginId);
                    }
                    continue;
                }
            }

            // load扩展点实现类
            Class<?> extensionClass = classLoader.loadClass(className);

            if (type.isAssignableFrom(extensionClass)) {
                // 实例化扩展点实现Wapper
                ExtensionWrapper extensionWrapper = createExtensionWrapper(extensionClass);
                result.add(extensionWrapper);
            } else {
                if (RuntimeMode.DEVELOPMENT.equals(pluginManager.getRuntimeMode())) {
                    checkDifferentClassLoaders(type, extensionClass);
                }
            }
        } catch (ClassNotFoundException e) {
        }
    }

    if (result.isEmpty()) {
    } else {
    }

    Collections.sort(result);

    return result;
}
~~~

## 2.5.5 根据扩展点实现Wapper实例化

~~~
private <T> List<T> getExtensions(List<ExtensionWrapper<T>> extensionsWrapper) {
    List<T> extensions = new ArrayList<>(extensionsWrapper.size());
    for (ExtensionWrapper<T> extensionWrapper : extensionsWrapper) {
        try {
            extensions.add(extensionWrapper.getExtension());
        } catch (PluginRuntimeException e) {
        }
    }

    return extensions;
}

public T getExtension() {
    if (extension == null) {
        extension = (T) extensionFactory.create(descriptor.extensionClass);
    }

    return extension;
}
~~~

&nbsp;&nbsp;&nbsp;&nbsp;扩展点实现创建工厂有两种，第一种是单例型的创建，一种是每次请求都创建，根据初始化PluginManager指定，此处以单例型创建工厂介绍。

~~~
public <T> T create(Class<T> extensionClass) {
    String extensionClassName = extensionClass.getName();
    // 若之前已经创建，那么直接返回
    if (cache.containsKey(extensionClassName)) {
        return (T) cache.get(extensionClassName);
    }

    // 否则调用基类实例化
    T extension = super.create(extensionClass);
    if (extensionClassNames.isEmpty() || extensionClassNames.contains(extensionClassName)) {
        cache.put(extensionClassName, extension);
    }

    return extension;
}

public <T> T create(Class<T> extensionClass) {
    try {
        // 因此扩展点实现类必须包含空的构造函数，这个是pf4j扩展点实现的要求
        return extensionClass.newInstance();
    } catch (Exception e) {
        throw new PluginRuntimeException(e);
    }
}
~~~

