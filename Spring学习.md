- BeanFactory

  以Factory结尾，表示它是一个工厂类，负责生产和管理bean的工厂。在Spring中，BeanFacotry是IOC容器的核心接口，它的职责包括：实例化、定位、配置应用程序中的对象以及建立这些对象间的依赖。

  它只是一个接口，有很多实现，比如DefaultListableBeanFactory、ApplicationContext、XmlBeanFactory等等，这些实现不仅实现了BeanFactory，还扩展了一些其他的功能。如ApplicationContext提供了MessageSource、资源访问、事件传播、载入多个上下文

  ```java
  public interface BeanFactory {
  	//对FactoryBean的转义定义，因为如果使用bean的名字检索FactoryBean得到的对象是工厂生成的对象，
  	//如果需要得到工厂本身，需要转义
  	String FACTORY_BEAN_PREFIX = "&";
  
  	//根据bean的名字，获取在IOC容器中得到bean实例
  	Object getBean(String name) throws BeansException;
  
  	//根据bean的名字和Class类型来得到bean实例，增加了类型安全验证机制。
  	<T> T getBean(String name, @Nullable Class<T> requiredType) throws BeansException;
  
  	Object getBean(String name, Object... args) throws BeansException;
  
  	<T> T getBean(Class<T> requiredType) throws BeansException;
  
  	<T> T getBean(Class<T> requiredType, Object... args) throws BeansException;
  
  	//提供对bean的检索，看看是否在IOC容器有这个名字的bean
  	boolean containsBean(String name);
  
  	//根据bean名字得到bean实例，并同时判断这个bean是不是单例
  	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
  
  	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;
  
  	boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;
  
  	boolean isTypeMatch(String name, @Nullable Class<?> typeToMatch) throws NoSuchBeanDefinitionException;
  
  	//得到bean实例的Class类型
  	@Nullable
  	Class<?> getType(String name) throws NoSuchBeanDefinitionException;
  
  	//得到bean的别名，如果根据别名检索，那么其原名也会被检索出来
  	String[] getAliases(String name);
  }
  
  ```

  BeanFactory是简单工厂模式，实质是一个工厂类根据传入参数，动态创建哪个类

- FactoryBean

  Spring通过反射通过class来了实例化bean，某些bean的实例化过程比较复杂，如果按传统的方式，需要在bean中提供大量的配置信息，这样不够灵活。为此，Spring提供了FactoryBean，通过实现该接口定制实例化Bean。FactoryBean对Spring框架来说极其重要，Spring自身提供了70多个实现。它们隐藏了实例化复杂的Bean的细节，给上层带来便利。第三方库不能直接注册到Spring容器，可以实现该接口给出自己对象的实例化代码即可。从Spring3.0开始，支持泛型，FactoryBean<T>

  以Bean结尾，表示它是一个Bean，与普通Bean不同的是，它是能生产或者修饰对象生成的工厂bean（工厂模式、装饰器模式），根据该bean的名称从BeanFacotry获得的Bean实际上是FactoryBean.getObject返回的对象，而不是FacotryBean本身

  ```java
  public interface FactoryBean<T> {
  	//从工厂中获取bean【这个方法是FactoryBean的核心】
  	@Nullable
  	T getObject() throws Exception;
  	
  	//获取Bean工厂创建的对象的类型【注意这个方法主要作用是：该方法返回的类型是在ioc容器中getbean所匹配的类型】
  	@Nullable
  	Class<?> getObjectType();
  	
  	//Bean工厂创建的对象是否是单例模式
  	default boolean isSingleton() {
  		return true;
  	}
  }
  
  ```

  FactoryBean表现的是一个工厂的职责，一个Bean A如果实现了FactoryBean，那么A就变成了一个工厂，根据A的名称获取到的Bean实际上是工厂调用getObject返回的对象，而不是A本身。如果要获取A工厂自身的实例，在名称前加上'&'符号

  ```java
  public class XXX implements FactoryBean {
      @Override
      public Object getObject() throws Exception {
          return new YYY;
      }       
      @Override
      public Class<?> getObjectType() {  //注意这个方法主要作用是：该方法返回的类型是在ioc容器中getbean所匹配的类型
          return AAA.class;
      }
  }
  
  那么要想在ioc中找到XXX这个类的bean（实际上是YYY） ，在getbean的时候写法如下
  
  public class Demo1 {
      public static void main(String[] args) {
          AnnotationConfigApplicationContext annotationConfigApplicationContext = 
                  new AnnotationConfigApplicationContext(Appconfig.class);
                  
          annotationConfigApplicationContext.getBean( AAA.class ); // 【注意这里是AAA.class】
  
      }
  }
  
  ```

  FactoryBean是工厂方法模式，也是简单工厂模式的升级或抽象，可以应用于更复杂的场景，更灵活。简单工厂中是工厂类进行所有逻辑判断、实例创建；而在工厂方法中，不同产品提供不同工厂，不同工厂生产不同产品，每个工厂只对应一个相应对象