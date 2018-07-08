接着上节内容[六、spring ioc之AnnotationConfigApplicationContext源码解析](https://www.jianshu.com/p/f16aa99b7daa),这节我们分析ConfigurationClassPostProcessor是如何加载注解配置中的bean。
# 类图
![ConfigurationClassPostProcessor类图](https://upload-images.jianshu.io/upload_images/10236819-5cfb78e7628f4635.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
由类图可以看出ConfigurationClassPostProcessor实现了BeanDefinitionRegistry-PostProcessor，由上节分析，会调用postProcessBeanDefinitionRegistry方法。

# 解析postProcessBeanDefinitionRegistry执行逻辑
## 时序图
![ConfigurationClassPostProcessor时序图](https://upload-images.jianshu.io/upload_images/10236819-d70e0272c5549e61.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 1.postProcessBeanDefinitionRegistry()
由AbstractApplicationContext#refresh()中的invokeBeanFactory-PostProcessors(beanFactory)调用
## 2.processConfigBeanDefinitions(BeanDefinitionRegistry registry)
```java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    List<BeanDefinitionHolder> configCandidates = new ArrayList<BeanDefinitionHolder>();
    String[] candidateNames = registry.getBeanDefinitionNames();
    //遍历标记@Configuration注解的bean
    for (String beanName : candidateNames) {
        BeanDefinition beanDef = registry.getBeanDefinition(beanName);
        if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
                ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
            if (logger.isDebugEnabled()) {
                logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
            }
        }
        else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
            configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
        }
    }

    // Return immediately if no @Configuration classes were found
    if (configCandidates.isEmpty()) {
        return;
    }

    // Sort by previously determined @Order value, if applicable
    Collections.sort(configCandidates, new Comparator<BeanDefinitionHolder>() {
        @Override
        public int compare(BeanDefinitionHolder bd1, BeanDefinitionHolder bd2) {
            int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
            int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
            return (i1 < i2) ? -1 : (i1 > i2) ? 1 : 0;
        }
    });

    // Detect any custom bean name generation strategy supplied through the enclosing application context
    SingletonBeanRegistry sbr = null;
    if (registry instanceof SingletonBeanRegistry) {
        sbr = (SingletonBeanRegistry) registry;
        if (!this.localBeanNameGeneratorSet && sbr.containsSingleton(CONFIGURATION_BEAN_NAME_GENERATOR)) {
            BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);
            this.componentScanBeanNameGenerator = generator;
            this.importBeanNameGenerator = generator;
        }
    }

    // Parse each @Configuration class
    ConfigurationClassParser parser = new ConfigurationClassParser(
            this.metadataReaderFactory, this.problemReporter, this.environment,
            this.resourceLoader, this.componentScanBeanNameGenerator, registry);

    Set<BeanDefinitionHolder> candidates = new LinkedHashSet<BeanDefinitionHolder>(configCandidates);
    Set<ConfigurationClass> alreadyParsed = new HashSet<ConfigurationClass>(configCandidates.size());
    //解析所有的@Configuration
    do {
        parser.parse(candidates);
        parser.validate();

        Set<ConfigurationClass> configClasses = new LinkedHashSet<ConfigurationClass>(parser.getConfigurationClasses());
        configClasses.removeAll(alreadyParsed);

        // Read the model and create bean definitions based on its content
        if (this.reader == null) {
            this.reader = new ConfigurationClassBeanDefinitionReader(
                    registry, this.sourceExtractor, this.resourceLoader, this.environment,
                    this.importBeanNameGenerator, parser.getImportRegistry());
        }
        this.reader.loadBeanDefinitions(configClasses);
        alreadyParsed.addAll(configClasses);

        candidates.clear();
        if (registry.getBeanDefinitionCount() > candidateNames.length) {
            String[] newCandidateNames = registry.getBeanDefinitionNames();
            Set<String> oldCandidateNames = new HashSet<String>(Arrays.asList(candidateNames));
            Set<String> alreadyParsedClasses = new HashSet<String>();
            for (ConfigurationClass configurationClass : alreadyParsed) {
                alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
            }
            for (String candidateName : newCandidateNames) {
                if (!oldCandidateNames.contains(candidateName)) {
                    BeanDefinition bd = registry.getBeanDefinition(candidateName);
                    if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
                            !alreadyParsedClasses.contains(bd.getBeanClassName())) {
                        candidates.add(new BeanDefinitionHolder(bd, candidateName));
                    }
                }
            }
            candidateNames = newCandidateNames;
        }
    }
    while (!candidates.isEmpty());

    // Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
    if (sbr != null) {
        if (!sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
            sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
        }
    }

    if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
        ((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
    }
}
```
1. 遍历标记@Configuration注解的bean
2. 解析所有的@Configuration,解析逻辑交给ConfigurationClassParser处理
## 3.parse(Set<BeanDefinitionHolder> configCandidates)
```java
protected final void parse(AnnotationMetadata metadata, String beanName) throws IOException {
    processConfigurationClass(new ConfigurationClass(metadata, beanName));
}
```
## 4.processConfigurationClass
```java
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
    if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
        return;
    }

    ConfigurationClass existingClass = this.configurationClasses.get(configClass);
    if (existingClass != null) {
        if (configClass.isImported()) {
            if (existingClass.isImported()) {
                existingClass.mergeImportedBy(configClass);
            }
            // Otherwise ignore new imported config class; existing non-imported class overrides it.
            return;
        }
        else {
            // Explicit bean definition found, probably replacing an import.
            // Let's remove the old one and go with the new one.
            this.configurationClasses.remove(configClass);
            for (Iterator<ConfigurationClass> it = this.knownSuperclasses.values().iterator(); it.hasNext();) {
                if (configClass.equals(it.next())) {
                    it.remove();
                }
            }
        }
    }

    // Recursively process the configuration class and its superclass hierarchy.
    SourceClass sourceClass = asSourceClass(configClass);
    do {
        sourceClass = doProcessConfigurationClass(configClass, sourceClass);
    }
    while (sourceClass != null);

    this.configurationClasses.put(configClass, configClass);
}
```
## 5.doProcessConfigurationClass
一层层调用，到这步才是真正的解析bean, 从注释上不难看出，每步在处理哪个注解
```java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
			throws IOException {
    // Recursively process any member (nested) classes first
    processMemberClasses(configClass, sourceClass);

    // Process any @PropertySource annotations
    for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), PropertySources.class,
            org.springframework.context.annotation.PropertySource.class)) {
        if (this.environment instanceof ConfigurableEnvironment) {
            processPropertySource(propertySource);
        }
        else {
            logger.warn("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
                    "]. Reason: Environment must implement ConfigurableEnvironment");
        }
    }

    // Process any @ComponentScan annotations
    Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
    if (!componentScans.isEmpty() &&
            !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
        for (AnnotationAttributes componentScan : componentScans) {
            // The config class is annotated with @ComponentScan -> perform the scan immediately
            Set<BeanDefinitionHolder> scannedBeanDefinitions =
                    this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
            // Check the set of scanned definitions for any further config classes and parse recursively if needed
            for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                if (bdCand == null) {
                    bdCand = holder.getBeanDefinition();
                }
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                    parse(bdCand.getBeanClassName(), holder.getBeanName());
                }
            }
        }
    }

    // Process any @Import annotations
    processImports(configClass, sourceClass, getImports(sourceClass), true);

    // Process any @ImportResource annotations
    if (sourceClass.getMetadata().isAnnotated(ImportResource.class.getName())) {
        AnnotationAttributes importResource =
                AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
        String[] resources = importResource.getStringArray("locations");
        Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
        for (String resource : resources) {
            String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
            configClass.addImportedResource(resolvedResource, readerClass);
        }
    }

    // Process individual @Bean methods
    Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
    for (MethodMetadata methodMetadata : beanMethods) {
        configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
    }

    // Process default methods on interfaces
    processInterfaces(configClass, sourceClass);

    // Process superclass, if any
    if (sourceClass.getMetadata().hasSuperClass()) {
        String superclass = sourceClass.getMetadata().getSuperClassName();
        if (!superclass.startsWith("java") && !this.knownSuperclasses.containsKey(superclass)) {
            this.knownSuperclasses.put(superclass, configClass);
            // Superclass found, return its annotation metadata and recurse
            return sourceClass.getSuperClass();
        }
    }

    // No superclass -> processing is complete
    return null;
}
```
1. 处理@PropertySource注解
2. 处理@ComponentScan，这步会去扫描所有标记@Component的类,将所有的类注册bean。
3. 处理@Import注解
4. 处理@ImportResource
5. 处理@Bean注解

在这一步中，只有@ComponentScan把扫描到的类注册成bean，@Import、@ImportResource和@Bean还没有注册成bean，它们把所有的注解的内容解析存放到ConfigurationClass中。

## 6.loadBeanDefinitions(configClasses)
在解析完所有的注解后，又回到ConfigurationClassPostProcessor#processConfigBeanDefinitions,开始遍历ConfigurationClass，注册由@Import、@ImportResource、@Bean标记的bean。
```java
private void loadBeanDefinitionsForConfigurationClass(
			ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

    if (trackedConditionEvaluator.shouldSkip(configClass)) {
        String beanName = configClass.getBeanName();
        if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
            this.registry.removeBeanDefinition(beanName);
        }
        this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
        return;
    }

    if (configClass.isImported()) {
        registerBeanDefinitionForImportedConfigurationClass(configClass);
    }
    //加载@Bean
    for (BeanMethod beanMethod : configClass.getBeanMethods()) {
        loadBeanDefinitionsForBeanMethod(beanMethod);
    }
    //加载@ImportResource
    loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
    //加载@Import
    loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```
1. 加载@Bean
2. 加载@ImportResource
3. 加载@Import
# 遇到的坑
在练习注解配置的时候，一个类我使用三种方式配置，在xml中配置、@Bean配置、@Component配置，这样配置会不会报错，如果能够存在，到底最后会使用哪个bean?
1. bean能够重复配置
在ConfigurationClassBeanDefinitionReader中存在下面方法，判断@Bean是否能够重复创建
```java
protected boolean isOverriddenByExistingDefinition(BeanMethod beanMethod, String beanName) {
    if (!this.registry.containsBeanDefinition(beanName)) {
        return false;
    }
    BeanDefinition existingBeanDef = this.registry.getBeanDefinition(beanName);

    // Is the existing bean definition one that was created from a configuration class?
    // -> allow the current bean method to override, since both are at second-pass level.
    // However, if the bean method is an overloaded case on the same configuration class,
    // preserve the existing bean definition.
    //如果两个@Bean位于不同的配置类中，可以覆盖，如果相同的类，通过方法重载的方式配置，是不能被覆盖
    if (existingBeanDef instanceof ConfigurationClassBeanDefinition) {
        ConfigurationClassBeanDefinition ccbd = (ConfigurationClassBeanDefinition) existingBeanDef;
        return ccbd.getMetadata().getClassName().equals(
                beanMethod.getConfigurationClass().getMetadata().getClassName());
    }

    // A bean definition resulting from a component scan can be silently overridden
    // by an @Bean method, as of 4.2...
    //扫描的Bean能够被@Bean覆盖
    if (existingBeanDef instanceof ScannedGenericBeanDefinition) {
        return false;
    }

    // Has the existing bean definition bean marked as a framework-generated bean?
    // -> allow the current bean method to override it, since it is application-level
    if (existingBeanDef.getRole() > BeanDefinition.ROLE_APPLICATION) {
        return false;
    }

    // At this point, it's a top-level override (probably XML), just having been parsed
    // before configuration class processing kicks in...
    //xml可以通过配置allowBeanDefinitionOverriding属性，来标识允不允许重复，默认为true.
    if (this.registry instanceof DefaultListableBeanFactory &&
            !((DefaultListableBeanFactory) this.registry).isAllowBeanDefinitionOverriding()) {
        throw new BeanDefinitionStoreException(beanMethod.getConfigurationClass().getResource().getDescription(),
                beanName, "@Bean definition illegally overridden by existing bean definition: " + existingBeanDef);
    }
    if (logger.isInfoEnabled()) {
        logger.info(String.format("Skipping bean definition for %s: a definition for bean '%s' " +
                "already exists. This top-level bean definition is considered as an override.",
                beanMethod, beanName));
    }
    return true;
}
```
- 如果两个@Bean位于不同的配置类中，可以覆盖，如果相同的类，通过方法重载的方式配置，是不能被覆盖
- 扫描的Bean能够被@Bean覆盖
- xml可以通过配置allowBeanDefinitionOverriding属性，来标识允不允许重复，默认为true.
2. 能够重复，最后使用的是哪个？
从上节解析过程我们知道，Spring的解析顺序是@Component>@Bean>@ImportResource,后面加载的能够覆盖前面的,所以最后使用的是xml配置.