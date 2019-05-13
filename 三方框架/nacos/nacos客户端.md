[toc]
#### 配置中心
##### 基本使用
[基本使用介绍](https://nacos.io/zh-cn/docs/quick-start.html "nacos官网")
##### 配置文件加载顺序
cloud的配置文件从类```PropertySourceBootstrapConfiguration```开始。其Ordered设置为```Ordered.HIGHEST_PRECEDENCE + 10```。接下来重点看看初始化方法：
```
public void initialize(ConfigurableApplicationContext applicationContext) {
        //首先加载bootstrap配置文件
		CompositePropertySource composite = new CompositePropertySource(
				BOOTSTRAP_PROPERTY_SOURCE_NAME);
		AnnotationAwareOrderComparator.sort(this.propertySourceLocators);
		boolean empty = true;
		ConfigurableEnvironment environment = applicationContext.getEnvironment();
        //调用所有locator加载配置文件
		for (PropertySourceLocator locator : this.propertySourceLocators) {
			PropertySource<?> source = null;
			source = locator.locate(environment);
			if (source == null) {
				continue;
			}
			logger.info("Located property source: " + source);
			composite.addPropertySource(source);
			empty = false;
		}
		if (!empty) {
			MutablePropertySources propertySources = environment.getPropertySources();
			String logConfig = environment.resolvePlaceholders("${logging.config:}");
			LogFile logFile = LogFile.get(environment);
            //移除locators加载的bootstrap配置文件，防止重复加载
			if (propertySources.contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
				propertySources.remove(BOOTSTRAP_PROPERTY_SOURCE_NAME);
			}
            //设置外部配置文件应用顺序
			insertPropertySources(propertySources, composite);
			reinitializeLoggingSystem(environment, logConfig, logFile);
			setLogLevels(applicationContext, environment);
			handleIncludedProfiles(environment);
		}
	}
```
nacos里面实现了一个Locator（```NacosPropertySourceLocator```），接下来看看其locate方法：
```
public PropertySource<?> locate(Environment env) {

		ConfigService configService = nacosConfigProperties.configServiceInstance();

		if (null == configService) {
			LOGGER.warn(
					"no instance of config service found, can't load config from nacos");
			return null;
		}
		long timeout = nacosConfigProperties.getTimeout();
		nacosPropertySourceBuilder = new NacosPropertySourceBuilder(configService,
				timeout);
		String name = nacosConfigProperties.getName();

		String nacosGroup = nacosConfigProperties.getGroup();
		String dataIdPrefix = nacosConfigProperties.getPrefix();
		if (StringUtils.isEmpty(dataIdPrefix)) {
			dataIdPrefix = name;
		}

		if (StringUtils.isEmpty(dataIdPrefix)) {
			dataIdPrefix = env.getProperty("spring.application.name");
		}

		List<String> profiles = Arrays.asList(env.getActiveProfiles());
		nacosConfigProperties.setActiveProfiles(profiles.toArray(new String[0]));

		String fileExtension = nacosConfigProperties.getFileExtension();

		CompositePropertySource composite = new CompositePropertySource(
				NACOS_PROPERTY_SOURCE_NAME);
        // 加载共享配置
		loadSharedConfiguration(composite);
        //加载扩展配置
		loadExtConfiguration(composite);
        //加载应用配置
		loadApplicationConfiguration(composite, nacosGroup, dataIdPrefix, fileExtension);

		return composite;
	}
```
注意，默认情况下是以nacos的配置文件为主，但是有时候需要以本地属性为主则需要在nacos的配置文件添加以下属性：
```
    spring.cloud.config.overrideNone=true: Override from any local property source.
    spring.cloud.config.overrideSystemProperties=false: Only system properties, command line arguments, and environment variables (but not the local config files) should override the remote settings.
```

##### 配置文件同步频率
```NacosConfigService```初始化时初会始化```ClientWorker```,进而会初始化一个定时器:
```
executor.scheduleWithFixedDelay(new Runnable() {
            public void run() {
                try {
                    checkConfigInfo();
                } catch (Throwable e) {
                    log.error(agent.getName(), "NACOS-XXXX", "[sub-check] rotate check error", e);
                }
            }
        }, 1L, 10L, TimeUnit.MILLISECONDS);

 public void checkConfigInfo() {
        // 分任务
        int listenerSize = cacheMap.get().size();
        // 向上取整为批数
        int longingTaskCount = (int)Math.ceil(listenerSize / ParamUtil.getPerTaskConfigSize());
        if (longingTaskCount > currentLongingTaskCount) {
            for (int i = (int)currentLongingTaskCount; i < longingTaskCount; i++) {
                // 要判断任务是否在执行 这块需要好好想想。 任务列表现在是无序的。变化过程可能有问题
                executorService.execute(new LongPollingRunnable(i));
            }
            currentLongingTaskCount = longingTaskCount;
        }
    }
```
其实这里就是10毫秒执行一次检查cacheMap是否有新的元素，有就在executorService添加一个任务。接下来看看cacheMap的数据添加的地方：
```NacosContextRefresher```在监听到```ApplicationReadyEvent```事件后```调用registerNacosListenersForApplications```，在这个方法里面会为每一个从nacos加载的配置文件注册一个监听器，这个监听器实际上就是构建CacheData并添加到cacheMap里面。
上面分析了LongPollingRunnable的添加机制，现在看看其```run```方法的构成：
第一部分是检查本地缓存进行灾备，第二部分是检查服务端数据，根据MD5值判断是否有变动，最后在finally里面又将本身加入executorService，所以配置文件更新不间断拉取数据，每次处理需要2-3秒。
 
##### 属性自动刷新机制
自动刷新主要是依赖spring的Scpoe特性。bean有个scope属性，常见为singleton和prototype，也可以自定义Scope属性，Cloud里面的RefreshScope就是自定义的一种。
当bean的Scope属性为RefreshScope时，获取实例实际是从RefreshScope获取，里面维护了各个bean的缓存，刷新实际就是清空缓存再重新构建bean。接下来分析下刷新流程：
1.在```NacosContextRefresher```注册的配置文件监听器中，当配置文件更新会发布```RefreshEvent```事件，并且在```RefreshEventListener```中监听了该事件。
2.```RefreshEventListener```收到刷新事件后会调用```ContextRefresher.refresh```，详细代码如下：
```
public synchronized Set<String> refresh() {
		Map<String, Object> before = extract(
				this.context.getEnvironment().getPropertySources());
		addConfigFilesToEnvironment();
		Set<String> keys = changes(before,
				extract(this.context.getEnvironment().getPropertySources())).keySet();
		this.context.publishEvent(new EnvironmentChangeEvent(context, keys));
		this.scope.refreshAll();
		return keys;
	}
```
3.一个是发布```EnvironmentChangeEvent```事件重新绑定```@ConfigurationProperties```的bean和属性，一个是调用this.scope.refreshAll();也就是刷新RefreshScope的bean。
#### 注册中心
##### 基本使用
[基本使用介绍](https://nacos.io/zh-cn/docs/quick-start.html "nacos官网")
##### 开始注册
在0.1.x版本是```AbstractDiscoveryLifecycle.start```里面调用```register()```方法，```NacosAutoServiceRegistration```实现了该方法，并触发```NacosServiceRegistry.register```
在0.2.x版本是通过```AbstractAutoServiceRegistration```监听```WebServerInitializedEvent```事件触发
##### 心跳
```BeatReactor```里面主要进行心跳维护,初始化方法里面创建了一个线程池```executorService.scheduleAtFixedRate(new BeatProcessor(), 0, clientBeatInterval, TimeUnit.MILLISECONDS);```，每五秒执行一次```BeatProcessor```的run方法.
```
 public void run() {
            try {
                for (Map.Entry<String, BeatInfo> entry : dom2Beat.entrySet()) {
                    BeatInfo beatInfo = entry.getValue();
                    if (beatInfo.isScheduled()) {
                        continue;
                    }
                    beatInfo.setScheduled(true);
                    executorService.schedule(new BeatTask(beatInfo), 0, TimeUnit.MILLISECONDS);
                }
            } catch (Exception e) {
                LogUtils.LOG.error("CLIENT-BEAT", "Exception while scheduling beat.", e);
            }
        }

public void run() {
            long result = serverProxy.sendBeat(beatInfo);
            beatInfo.setScheduled(false);
            if (result > 0) {
                clientBeatInterval = result;
            }
        }
```
```dom2Beat```里面保存的是需要注册和维持心跳的服务信息，而```BeatTask```的run方法发送心跳信息.
##### 服务信息刷新
```HostReactor```用于获取、保存、更新各Service实例信息,其中serviceInfoMap保存了已经获取到的服务信息。
通过```HostReactor```获取某一服务信息时会调用scheduleUpdateIfAbsent()创建一个future保存在futureMap,并且会创建一个UpdateTask,在UpdateTask的run方法里面会根据服务的cacheMillis值定时更新服务信息，默认值为10秒.
##### 灾备缓存

```FailoverReactor```主要进行本地服务信息缓存处理，留待灾备.
```
public void init() {

        executorService.scheduleWithFixedDelay(new SwitchRefresher(), 0L, 5000L, TimeUnit.MILLISECONDS);

        executorService.scheduleWithFixedDelay(new DiskFileWriter(), 30, DAY_PERIOD_MINUTES, TimeUnit.MINUTES);

        // backup file on startup if failover directory is empty.
        executorService.schedule(new Runnable() {
            @Override
            public void run() {
                try {
                    File cacheDir = new File(failoverDir);

                    if (!cacheDir.exists() && !cacheDir.mkdirs()) {
                        throw new IllegalStateException("failed to create cache dir: " + failoverDir);
                    }

                    File[] files = cacheDir.listFiles();
                    if (files == null || files.length <= 0) {
                        new DiskFileWriter().run();
                    }
                } catch (Throwable e) {
                    LogUtils.LOG.error("NA", "failed to backup file on startup.", e);
                }

            }
        }, 10000L, TimeUnit.MILLISECONDS);
    }
```
1.```SwitchRefresher```是定时读取一个叫00-00---000-VIPSRV_FAILOVER_SWITCH-000---00-00的文件判断是否开启故障转移，如果开启，HostReactor获取服务信息时则从缓存读取。
2.```DiskFileWriter```是每24小时将服务信息本地文件缓存一份。
3.启动时检测本地缓存文件，无则写入一份.