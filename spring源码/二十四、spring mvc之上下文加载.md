由上节[二十三、spring mvc之简单使用](https://www.jianshu.com/p/970a457e7633),SpringServletContainerInitializer找到所有的WebApplicationInitializer后，会调用它们的onStartup方法，这节我们看下AbstractAnnotationConfigDispatcherServletInitializer的onStartup执行逻辑。
AbstractAnnotationConfigDispatcherServletInitializer类图如下：
![AbstractAnnotationConfigDispatcherServletInitializer](https://upload-images.jianshu.io/upload_images/10236819-68c3000d813c4a17.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
onStartup方法在AbstractAnnotationConfigDispatcherServletInitializer父类AbstractDispatcherServletInitializer实现：
```java
@Override
public void onStartup(ServletContext servletContext) throws ServletException {
    super.onStartup(servletContext);
    registerDispatcherServlet(servletContext);
}
```
AbstractDispatcherServletInitializer又调用它的父类AbstractContextLoaderInitializer的onStartup方法。
```java
@Override
public void onStartup(ServletContext servletContext) throws ServletException {
    registerContextLoaderListener(servletContext);
}
```
两个onStartup方法分别调用了registerDispatcherServlet和registerContextLoaderListener方法，从方法名上我们能够猜出他们的功能，分别是注册DispatcherServlet和ContextLoaderListener。接下来我们具体分析这两个方法。
# registerContextLoaderListener方法
```java
protected void registerContextLoaderListener(ServletContext servletContext) {
    //1. 创建Spring上下文
    WebApplicationContext rootAppContext = createRootApplicationContext();
    if (rootAppContext != null) {
        //2. 创建ContextLoaderListener监听器
        ContextLoaderListener listener = new ContextLoaderListener(rootAppContext);
        listener.setContextInitializers(getRootApplicationContextInitializers());
        servletContext.addListener(listener);
    }
    else {
        logger.debug("No ContextLoaderListener registered, as " +
                "createRootApplicationContext() did not return an application context");
    }
}
```
registerContextLoaderListener方法执行逻辑如下:
1. 创建Spring上下文，上下文的创建交给子类AbstractAnnotationConfigDispatcherServletInitializer实现。
```java
@Override
protected WebApplicationContext createRootApplicationContext() {
    //1. 得到配置类路径，这个方法给使用者重写的
    Class<?>[] configClasses = getRootConfigClasses();
    if (!ObjectUtils.isEmpty(configClasses)) {
        //2. 创建AnnotationConfigWebApplicationContext对象
        AnnotationConfigWebApplicationContext rootAppContext = new AnnotationConfigWebApplicationContext();
        rootAppContext.register(configClasses);
        return rootAppContext;
    }
    else {
        return null;
    }
}
```
2. 创建ContextLoaderListener监听器,并把监听器添加的servlet上下文中。
## 初始化Spring上下文
ContextLoaderListener实现ServletContextListener，在servlet容器启动的时候就会调用它的contextInitialized方法。我们看下它的实现。
```java
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
	public ContextLoaderListener() {
	}

	public ContextLoaderListener(WebApplicationContext context) {
		super(context);
	}

	@Override
	public void contextInitialized(ServletContextEvent event) {
		initWebApplicationContext(event.getServletContext());
	}

	@Override
	public void contextDestroyed(ServletContextEvent event) {
		closeWebApplicationContext(event.getServletContext());
		ContextCleanupListener.cleanupAttributes(event.getServletContext());
	}
}
```
ContextLoaderListener的contextInitialized又调用父类ContextLoader的initWebApplicationContext方法。
```java
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
    //1. spring上下文是否已经加载过
    if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
        throw new IllegalStateException(
                "Cannot initialize context because there is already a root application context present - " +
                "check whether you have multiple ContextLoader* definitions in your web.xml!");
    }

    Log logger = LogFactory.getLog(ContextLoader.class);
    servletContext.log("Initializing Spring root WebApplicationContext");
    if (logger.isInfoEnabled()) {
        logger.info("Root WebApplicationContext: initialization started");
    }
    long startTime = System.currentTimeMillis();

    try {
        // Store context in local instance variable, to guarantee that
        // it is available on ServletContext shutdown.
        if (this.context == null) {
            this.context = createWebApplicationContext(servletContext);
        }
        if (this.context instanceof ConfigurableWebApplicationContext) {
            ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
            if (!cwac.isActive()) {
                // The context has not yet been refreshed -> provide services such as
                // setting the parent context, setting the application context id, etc
                if (cwac.getParent() == null) {
                    // The context instance was injected without an explicit parent ->
                    // determine parent for root web application context, if any.
                    ApplicationContext parent = loadParentContext(servletContext);
                    cwac.setParent(parent);
                }
                //2. 调用refresh加载Spring上下文
                configureAndRefreshWebApplicationContext(cwac, servletContext);
            }
        }
        //3. 设置spring上下文已经加载完成
        servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

        ClassLoader ccl = Thread.currentThread().getContextClassLoader();
        if (ccl == ContextLoader.class.getClassLoader()) {
            currentContext = this.context;
        }
        else if (ccl != null) {
            currentContextPerThread.put(ccl, this.context);
        }

        if (logger.isDebugEnabled()) {
            logger.debug("Published root WebApplicationContext as ServletContext attribute with name [" +
                    WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE + "]");
        }
        if (logger.isInfoEnabled()) {
            long elapsedTime = System.currentTimeMillis() - startTime;
            logger.info("Root WebApplicationContext: initialization completed in " + elapsedTime + " ms");
        }

        return this.context;
    }
    catch (RuntimeException ex) {
        logger.error("Context initialization failed", ex);
        servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
        throw ex;
    }
    catch (Error err) {
        logger.error("Context initialization failed", err);
        servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, err);
        throw err;
    }
}
```
initWebApplicationContext方法代码很长，业务逻辑不难，主要是做各种判断。逻辑如下：
1. 判断Spring上下文是否已经加载过，保证只加载一次。
2. 加载Spring上下文，调用的configureAndRefreshWebApplicationContext，调用的是我们很熟悉的refresh方法。
```java
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
    if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
        // The application context id is still set to its original default value
        // -> assign a more useful id based on available information
        String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
        if (idParam != null) {
            wac.setId(idParam);
        }
        else {
            // Generate default id...
            wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
                    ObjectUtils.getDisplayString(sc.getContextPath()));
        }
    }

    wac.setServletContext(sc);
    String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
    if (configLocationParam != null) {
        wac.setConfigLocation(configLocationParam);
    }

    // The wac environment's #initPropertySources will be called in any case when the context
    // is refreshed; do it eagerly here to ensure servlet property sources are in place for
    // use in any post-processing or initialization that occurs below prior to #refresh
    ConfigurableEnvironment env = wac.getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
        ((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
    }

    customizeContext(sc, wac);
    //刷新
    wac.refresh();
}
```
3. 设置spring上下文已经加载完成
# registerDispatcherServlet
```java
protected void registerDispatcherServlet(ServletContext servletContext) {
    //1. 得到servlet名字，默认是dispatcher
    String servletName = getServletName();
    Assert.hasLength(servletName, "getServletName() must not return empty or null");

    //2. 创建Servlet上下文
    WebApplicationContext servletAppContext = createServletApplicationContext();
    Assert.notNull(servletAppContext,
            "createServletApplicationContext() did not return an application " +
            "context for servlet [" + servletName + "]");
    //3. 创建dispatcherServlet
    FrameworkServlet dispatcherServlet = createDispatcherServlet(servletAppContext);
    dispatcherServlet.setContextInitializers(getServletApplicationContextInitializers());

    //4. 注册dispatcherServlet
    ServletRegistration.Dynamic registration = servletContext.addServlet(servletName, dispatcherServlet);
    Assert.notNull(registration,
            "Failed to register servlet with name '" + servletName + "'." +
            "Check if there is another servlet registered under the same name.");

    registration.setLoadOnStartup(1);
    registration.addMapping(getServletMappings());
    registration.setAsyncSupported(isAsyncSupported());

    //5. 注册过滤器，这些过滤器都是针对dispatcherServlet，所以不需要配置Mapping
    Filter[] filters = getServletFilters();
    if (!ObjectUtils.isEmpty(filters)) {
        for (Filter filter : filters) {
            registerServletFilter(servletContext, filter);
        }
    }

    customizeRegistration(registration);
}
```
registerDispatcherServlet方法的逻辑也很简单,命名好的重要性。
1. 得到servlet名字，默认是dispatcher
2. 创建Servlet上下文,还是交给子类AbstractAnnotationConfigDispatcherServletInitializer完成.
```java
@Override
protected WebApplicationContext createServletApplicationContext() {
    //和Spring上下问使用的是同一种上下文
    AnnotationConfigWebApplicationContext servletAppContext = new AnnotationConfigWebApplicationContext();
    //子类重写
    Class<?>[] configClasses = getServletConfigClasses();
    if (!ObjectUtils.isEmpty(configClasses)) {
        servletAppContext.register(configClasses);
    }
    return servletAppContext;
}
```
3. 创建dispatcherServlet
4. 注册dispatcherServlet
5. 注册过滤器,获取过滤器由子类重写，这些过滤器主要是针对dispatcherServlet，对其他的Servlet不生效。
## 初始化Servlet上下文
registerDispatcherServlet方法中只是创建了Servlet上下文，并没有加载上下文。那么加载的动作在哪做的呢？
DispatcherServlet的类图如下:
![DispatcherServlet](https://upload-images.jianshu.io/upload_images/10236819-956eb17e9a650f31.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
DispatcherServlet实现自HttpServlet，在Servlet容器加载的时候，会调用Servlet的init方法。DispatcherServlet的init方法在父类HttpServletBean中.
```java
@Override
public final void init() throws ServletException {
    if (logger.isDebugEnabled()) {
        logger.debug("Initializing servlet '" + getServletName() + "'");
    }

    // Set bean properties from init parameters.
    //1. 设置初始化参数
    PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
    if (!pvs.isEmpty()) {
        try {
            BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
            ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
            bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
            initBeanWrapper(bw);
            bw.setPropertyValues(pvs, true);
        }
        catch (BeansException ex) {
            if (logger.isErrorEnabled()) {
                logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
            }
            throw ex;
        }
    }

    // Let subclasses do whatever initialization they like.
    //2. 初始Servlet,由子类实现
    initServletBean();

    if (logger.isDebugEnabled()) {
        logger.debug("Servlet '" + getServletName() + "' configured successfully");
    }
}
```
init方法做了两件事：
1. 设置初始化参数
2. 初始Servlet,子类FrameworkServlet实现了这个方法
```java
@Override
protected final void initServletBean() throws ServletException {
    getServletContext().log("Initializing Spring FrameworkServlet '" + getServletName() + "'");
    if (this.logger.isInfoEnabled()) {
        this.logger.info("FrameworkServlet '" + getServletName() + "': initialization started");
    }
    long startTime = System.currentTimeMillis();

    try {
        //初始化Servlet上下文
        this.webApplicationContext = initWebApplicationContext();
        initFrameworkServlet();
    }
    catch (ServletException ex) {
        this.logger.error("Context initialization failed", ex);
        throw ex;
    }
    catch (RuntimeException ex) {
        this.logger.error("Context initialization failed", ex);
        throw ex;
    }

    if (this.logger.isInfoEnabled()) {
        long elapsedTime = System.currentTimeMillis() - startTime;
        this.logger.info("FrameworkServlet '" + getServletName() + "': initialization completed in " +
                elapsedTime + " ms");
    }
}
```
上面方法这么长的代码，只做了一件事，初始化Servlet上下文。
```java
protected WebApplicationContext initWebApplicationContext() {
    //1. 得到Spring上下文
    WebApplicationContext rootContext =
            WebApplicationContextUtils.getWebApplicationContext(getServletContext());
    WebApplicationContext wac = null;

    if (this.webApplicationContext != null) {
        // A context instance was injected at construction time -> use it
        wac = this.webApplicationContext;
        if (wac instanceof ConfigurableWebApplicationContext) {
            ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
            if (!cwac.isActive()) {
                // The context has not yet been refreshed -> provide services such as
                // setting the parent context, setting the application context id, etc
                if (cwac.getParent() == null) {
                    // The context instance was injected without an explicit parent -> set
                    // the root application context (if any; may be null) as the parent
                    //2. 设置Spring上下文作为当前父上下文
                    cwac.setParent(rootContext);
                }
                //调用refresh()
                configureAndRefreshWebApplicationContext(cwac);
            }
        }
    }
    if (wac == null) {
        // No context instance was injected at construction time -> see if one
        // has been registered in the servlet context. If one exists, it is assumed
        // that the parent context (if any) has already been set and that the
        // user has performed any initialization such as setting the context id
        wac = findWebApplicationContext();
    }
    if (wac == null) {
        // No context instance is defined for this servlet -> create a local one
        wac = createWebApplicationContext(rootContext);
    }

    if (!this.refreshEventReceived) {
        // Either the context is not a ConfigurableApplicationContext with refresh
        // support or the context injected at construction time had already been
        // refreshed -> trigger initial onRefresh manually here.
        onRefresh(wac);
    }

    if (this.publishContext) {
        // Publish the context as a servlet context attribute.
        String attrName = getServletContextAttributeName();
        getServletContext().setAttribute(attrName, wac);
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Published WebApplicationContext of servlet '" + getServletName() +
                    "' as ServletContext attribute with name [" + attrName + "]");
        }
    }

    return wac;
}
```
初始化Servlet上下文流程如下:
1. 得到Spring上下文，把Spring上下文设置成Servlet上下文的parent。
2. 调用refresh方法加载上下文。这里和Spring上下文的逻辑差不多，就不贴代码了。
# 疑惑
为什么Servlet上下文能够获得到Spring上下文中的bean。我在DefaultListableBeanFactory的getBean中找到答案:
```java
@Override
public <T> T getBean(Class<T> requiredType, Object... args) throws BeansException {
    //1. 从自己的容器中获取bean
    NamedBeanHolder<T> namedBean = resolveNamedBean(requiredType, args);
    if (namedBean != null) {
        return namedBean.getBeanInstance();
    }
    //2. 如果自己容器中没有，则尝试从父类容器中获取
    BeanFactory parent = getParentBeanFactory();
    if (parent != null) {
        return parent.getBean(requiredType, args);
    }
    throw new NoSuchBeanDefinitionException(requiredType);
}
```