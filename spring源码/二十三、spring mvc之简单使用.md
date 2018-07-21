# 配置DispatcherServlet
DispatcherServlet是Spring MVC的核心。它负责将请求路由到其他的组件中。
按照传统的方式，像DispatcherServlet这样的Servlet会配置在web.xml文件中，这个文件会放到应用的WAR包里面，当然，这是配置DispatcherServlet的方法之一。但是，借助于Servlet3和Spring3.0功能的增强，这种方式已经不是唯一的方案了。
我们会使用Java将DispatcherServlet配置在Servlet容器中，而不会再使用web.xml文件。如下的程序展示了所需的Java类。
```java
public class WebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
   //Spring上下文
    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class[]{RootConfig.class};
    }

    //servlet上下文
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class[]{WebConfig.class};
    }

    //DispatcherServlet映射路径
    @Override
    protected String[] getServletMappings() {
        return new String[]{"/"};
    }
}

@Configuration
@EnableWebMvc
@ComponentScan("com.ming.web")
public class WebConfig  extends WebMvcConfigurerAdapter {

}

@Configuration
@ComponentScan(basePackages = {"com.ming"},
        excludeFilters = {
        @ComponentScan.Filter(type = FilterType.ANNOTATION, value = EnableWebMvc.class)})
public class RootConfig {

}
```
要理解上面代码是如何工作的，我们可能只需知道扩展AbstractAnnotationConfigDispatcherServletInitializer的任意类都会自动地配置DispatcherServlet和Spring应用上下文，Spring的应用上下文会位于应用程序的Servlet上下文中。

# AbstractAnnotationConfigDispatcherServletInitializer剖析
在Servlet3.0环境中，容器会在类路径中查找实现javax.servlet.ServletContainerInitializer接口的类，如果能发现的话，就会用它来配置Servlet容器。
Spring提供了这个接口的实现，名为SpringServletContainerInitializer，这个类反过来又会查找实现WebApplicationInitializer的类并将配置任务交给他们来完成。
```java
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer {

	@Override
	public void onStartup(Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
			throws ServletException {

		List<WebApplicationInitializer> initializers = new LinkedList<WebApplicationInitializer>();

		if (webAppInitializerClasses != null) {
			for (Class<?> waiClass : webAppInitializerClasses) {
				// Be defensive: Some servlet containers provide us with invalid classes,
				// no matter what @HandlesTypes says...
				if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
						WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
					try {
						initializers.add((WebApplicationInitializer) waiClass.newInstance());
					}
					catch (Throwable ex) {
						throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
					}
				}
			}
		}

		if (initializers.isEmpty()) {
			servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
			return;
		}

		servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");
		AnnotationAwareOrderComparator.sort(initializers);
		for (WebApplicationInitializer initializer : initializers) {
			initializer.onStartup(servletContext);
		}
	}
}
```
Spring提供引入了一个便利的WebApplicationInitializer基础实现，也就是AbstractAnnotationConfigDispatcherServletInitializer，使用者只需要继承这个类，并重写其中需要的方法。当应用部署到servlet3.0容器中的时候，容器会自动发现它，并用它来配置Servlet上下文。具体如何配置下节分析。

# 参考资料
- 《Spring4实战》
