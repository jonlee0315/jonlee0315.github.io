---
layout:     post
title:      "Spring 学习笔记（一）"
subtitle:   "什么是Spring IoC？"
date:       2018-01-02
author:     "Jon Lee"
header-img: "img/in-post/2018-01-02-spring-notes-1/bg.jpg"
catalog:    true
categories : JavaWeb
tags:
    - Spring
    - Java
---

### 模拟Spring的IoC

构建一个Web应用程序，比较常见的一个分层方式如下。

1.	**Model** 实体类
2.	**DAO** （Data Access Object）数据访问层
3.	**DAOImpl** DAO实现，解决多种数据库的问题，不同的实现对应不同的数据库
4.	**Service** 业务逻辑层
5.	**ServiceImpl** 业务逻辑实现
6.	**Controller** Spring MVC在这一层，相当于struts中的action

![](/img/in-post/2018-01-02-spring-notes-1/1.png)

下面假设我们要完成一个填加用户的操作，按照上面的分层方式应该这么实现。

**Model层**

    public class User {
    	private String userName;
    	private String passWord;

    	public String getUserName() {
    		return userName;
    	}

    	public void setUserName(String userName) {
    		this.userName = userName;
    	}

    	public String getPassWord() {
    		return passWord;
    	}

    	public void setPassWord(String passWord) {
    		this.passWord = passWord;
    	}
    }

**DAO层**

    public interface UserDAO {
    	public void save(User u);
    }

**DAOImpl层**

    public class UserDAOImpl implements UserDAO {
    	@Override
    	public void save(User u) {
    		System.out.println("User Saved");
    	}
    }

**Service**

    public class UserService {
    	private UserDAO userDAO = new UserDAOImpl();

    	public UserDAO getUserDao() {
    		return userDAO;
    	}

    	public void setUserDao(UserDAO userDAO) {
    		this.userDAO = userDAO;
    	}

    	public void add(User u) {
    		this.userDAO.save(u);
    	}
    }

**Test**

    public class UserServiceTest {
    	@Test
    	public void testAdd() throws IOException, Exception {
    		UserService userService = new UserService();
    		User u = new User();
    		userService.add(u);
    	}
    }

上面就简单实现了User的填加操作，下面来简单模拟Spring是怎样实现依赖注入的。  
在上面的代码中可以看到每次调用填加操作都需要创建一系列对象，而当项目非常大时涉及的对象会非常多，而每个对象都要求编程人员自己去创建和维护是非常麻烦的，所以Spring的依赖注入帮我们干了这些事情。  
为了模拟Spring的功能，我们要先创建一个配置文件，Spring的配置文件采用XML格式，可以使用jdom、dom4j解析。

所以下面需要的工作就是创建一个xml配置文件，再通过把其中需要初始化的对象读出来，放到一个容器里以方便我们的使用。

**beans.xml**

    <beans>
    	<bean id="u" class="edu.heu.dao.impl.UserDAOImpl"/>
    	<bean id="userService" class="edu.heu.service.UserService">
    		<property name="userDAO" bean="u"/>
    	</bean>
    </beans>

beans.xml有两个功能，一是初始化`UserDAOImpl`和`UserService`，二是将`UserDAOImpl`对象注入到`UserService`中，**property** 相当于`setter`方法。  
下面来模拟Spring中的`ClassPathXmlApplicationContext`（实现`BeanFactory`接口），实现对xml配置文件的读写。

**BeanFactory**

    public interface BeanFactory {
    	public Object getBean(String name);
    }

**ClassPathXmlApplicationContext**

    public class ClassPathXmlApplicationContext implements BeanFactory {
    	private Map<String, Object> beans = new HashMap<String, Object>();

    	@SuppressWarnings("unchecked")
    	public ClassPathXmlApplicationContext() throws Exception, IOException {
    		SAXBuilder sb = new SAXBuilder();
    		Document doc = sb.build(this.getClass().getClassLoader().getResourceAsStream("beans.xml"));
    		Element root = doc.getRootElement();
    		List<Element> list = root.getChildren("bean");
    		for (Element element : list) {
    			String id = element.getAttributeValue("id");
    			String className = element.getAttributeValue("class");
    			System.out.println(id + ":" + className);
    			Object o = Class.forName(className).newInstance();
    			beans.put(id, o);
    			for (Element propertyElement : (List<Element>) element.getChildren("property")) {
    				String name = propertyElement.getAttributeValue("name"); // userDao
    				String bean = propertyElement.getAttributeValue("bean"); // u
    				Object beanObject = beans.get(bean); // get UserDaoImpl instance
    				String methodName = "set" + name.substring(0, 1).toUpperCase() + name.substring(1);
    				System.out.println("method name = " + methodName);
    				// setUserDao(UserDao.class)
    				Method m = o.getClass().getMethod(methodName, beanObject.getClass().getInterfaces()[0]);
    				m.invoke(o, beanObject);
    			}
    		}
    	}

    	@Override
    	public Object getBean(String name) {
    		return beans.get(name);
    	}
    }

可以看到上面的代码依靠于反射实现，通过上面的代码，我们可以使用`ClassPathXmlApplicationContext`来获得`bean`并且自动进行注入。测试代码如下：

    public class UserServiceTest {
    	@Test
    	public void test() throws IOException, Exception {
    		BeanFactory factory = new ClassPathXmlApplicationContext();
    		UserService service = (UserService) factory.getBean("userService");
    		User u = new User();
    		service.add(u);
    	}
    }

*Output*
>u:edu.heu.dao.impl.UserDAOImpl  
userService:edu.heu.service.UserService  
method name = setUserDAO  
User Saved  

上面的代码简单地模拟了Spring中IoC，由Spring中的容器帮助我们控制这些对象，而不是我们自己，灵活地对对象进行初始化和装配，减小耦合。

### Spring IoC

#### 概述

要了解控制反转(Inversion of Control), 有必要先了解软件设计的一个重要思想：依赖倒置原则（Dependency Inversion Principle）。  
依赖倒置原则就是抽象不应该依赖于细节，细节应当依赖于抽象。换言之，要针对接口编程，而不是针对实现编程。这么说还是有点抽象，下面就举一个知乎上的高赞回答中的例子[Spring IoC有什么好处呢？ - Sevenvidia的回答 - 知乎](
https://www.zhihu.com/question/23277575/answer/169698662)。

>什么是依赖倒置原则？假设我们设计一辆汽车：先设计轮子，然后根据轮子大小设计底盘，接着根据底盘设计车身，最后根据车身设计好整个汽车。这里就出现了一个“依赖”关系：汽车依赖车身，车身依赖底盘，底盘依赖轮子。  
![](/img/in-post/2018-01-02-spring-notes-1/3.png)  
这样的设计看起来没问题，但是可维护性却很低。假设设计完工之后，上司却突然说根据市场需求的变动，要我们把车子的轮子设计都改大一码。这下我们就蛋疼了：因为我们是根据轮子的尺寸设计的底盘，轮子的尺寸一改，底盘的设计就得修改；同样因为我们是根据底盘设计的车身，那么车身也得改，同理汽车设计也得改——整个设计几乎都得改！  
我们现在换一种思路。我们先设计汽车的大概样子，然后根据汽车的样子来设计车身，根据车身来设计底盘，最后根据底盘来设计轮子。这时候，依赖关系就倒置过来了：轮子依赖底盘， 底盘依赖车身， 车身依赖汽车。  
![](/img/in-post/2018-01-02-spring-notes-1/4.png)  
这时候，上司再说要改动轮子的设计，我们就只需要改动轮子的设计，而不需要动底盘，车身，汽车的设计了。  
这就是依赖倒置原则——把原本的高层建筑依赖底层建筑“倒置”过来，变成底层建筑依赖高层建筑。高层建筑决定需要什么，底层去实现这样的需求，但是高层并不用管底层是怎么实现的。这样就不会出现前面的“牵一发动全身”的情况。  
![](/img/in-post/msb_spring_notes/5.png)  
这里IoC Container可以直接隐藏具体的创建实例的细节，在我们来看它就像一个工厂：  
![](/img/in-post/2018-01-02-spring-notes-1/6.png)  
我们就像是工厂的客户。我们只需要向工厂请求一个Car实例，然后它就给我们按照Config创建了一个Car实例。我们完全不用管这个Car实例是怎么一步一步被创建出来。

IoC控制反转（Inversion of Control） 就是依赖倒置原则的一种代码设计的思路。具体采用的方法就是所谓的依赖注入（Dependency Injection）。  
在Spring中`org.springframework.context.ApplicationContext`接口代表IoC容器，它负责bean的初始化、配置和装配。可以通过xml、注解或者代码的方式来表示配置元数据，容器通过读取配置数据来知道它需要干什么。

![](/img/in-post/2018-01-02-spring-notes-1/2.png)

根据官方文档我们重写上面的xml配置文件，使用Spring来实现之前的功能。

    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
    	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    	xsi:schemaLocation="http://www.springframework.org/schema/beans
    		http://www.springframework.org/schema/beans/spring-beans.xsd">

    	<bean id="u" class="edu.heu.dao.impl.UserDaoImpl">
    	</bean>
    	<bean id="userService" class="edu.heu.service.UserService">
    		<property name="userDao" ref="u"/>
    	</bean>
    </beans>

**Test**

    public class UserServiceTest {
    	@Test
    	public void testAdd() throws IOException, Exception {
    		ApplicationContext applicationContext = new ClassPathXmlApplicationContext("beans.xml");
    		UserService service = (UserService) applicationContext.getBean("userService");
    		User u = new User();
    		u.setUserName("lee");
    		service.add(u);
    	}
    }

可以看到测试类中的代码与之前并没有太大变化，但是有一点需要注意，查看JDK文档可以发现，接口`ApplicationContext`和`BeanFactory`都是`ClassPathXmlApplicationContext`的父接口，那么如何选择呢？关于这个问题Spring文档中给出了明确说明：  
![](/img/in-post/2018-01-02-spring-notes-1/7.png)

因为`ApplicationContext`包含了`BeanFactory`所有的功能，除了少数几种情况下，推荐使用`ApplicationContext`。

#### 注入类型

主要有两种分别是 **setter注入** 和 **构造方法注入**，接口注入不常用。  
之前写过的其实就是setter注入，通过`property`标签。下面来看一下构造方法注入是怎么实现的，也很简单。

    <bean id="u" class="edu.heu.dao.impl.UserDaoImpl">
    </bean>
    <bean id="userService" class="edu.heu.service.UserService">
        <!-- <constructor-arg>
            <ref bean="u"/>
        </constructor-arg> -->
        <constructor-arg ref="u"></constructor-arg>
    </bean>

两种写法都可以，当构造方法中有多个参数时，为了避免歧义可以使用索引，类型、名字等属性指定。

> **setter注入** 方式可以解决循环依赖问题。什么是循环依赖呢？简单来说就是类A中依赖类B的一个实例，类B中又依赖类A，就会出现循环。  
通过构造器注入构成的循环依赖，这种依赖是无法解决的，只能抛出`BeanCreationException`异常表示循环依赖。  
setter方法也是Spring中的默认解决方法，可以解决循环依赖的问题。具体实现原理是：setter注入造成的依赖是通过Spring容器提前暴露刚完成构造器注入但并未完成其他步骤的bean完成。  
Spring是先将Bean对象实例化之后再设置对象属性的，此时Spring会将这个实例化结束的对象放到一个Map中，并且Spring提供了获取这个未设置属性的实例化对象引用的方法。 结合我们的实例来看，当Spring实例化了TestA、TestB、TestC后，紧接着会去设置对象的属性，此时TestA依赖TestB，就会去Map中取出存在里面的单例TestB对象，以此类推。  
对于`prototype`作用域的Bean，Spring容器无法完成依赖注入，因为Spring容器不进行缓存，因此无法提前暴露一个创建中的Bean。

所以推荐使用setter方法进行注入。下面看一下官方文档中的例子：

    public class ExampleBean {
    	private AnotherBean beanOne;
    	private YetAnotherBean beanTwo;
    	private int i;

    	public void setBeanOne(AnotherBean beanOne) {
    		this.beanOne = beanOne;
    	}

    	public void setBeanTwo(YetAnotherBean beanTwo) {
    		this.beanTwo = beanTwo;
    	}

    	public void setIntegerProperty(int i) {
    		this.i = i;
    	}
    }

对应xml：

    <bean id="exampleBean" class="examples.ExampleBean">
    <!-- setter injection using the nested ref element -->
        <property name="beanOne">
            <ref bean="anotherExampleBean"/>
        </property>
    <!-- setter injection using the neater ref attribute -->
        <property name="beanTwo" ref="yetAnotherBean"/>
        <property name="integerProperty" value="1"/>
    </bean>
    <bean id="anotherExampleBean" class="examples.AnotherBean"/>
    <bean id="yetAnotherBean" class="examples.YetAnotherBean"/>

#### Bean的作用范围

Bean的作用范围有以下几种：

![](/img/in-post/2018-01-02-spring-notes-1/8.png)

**默认是singleton**，和设计模式中的单例模式是一样的，在容器中只创建一个实例，大家一起用。

![](/img/in-post/2018-01-02-spring-notes-1/9.png)

另一种常用的是 **prototype**，每次注入或者调用容器的`getBean`方法都会创建一个实例。

![](/img/in-post/2018-01-02-spring-notes-1/10.png)

<bean id="accountService" class="com.foo.DefaultAccountService" scope="prototype"/>

有一点需要注意就是 **Spring并不能保证Bean的线程安全性**。  
Bean分两种，**有状态** 和 **无状态**，无状态的Bean适合使用单例模式创建，例如一些Service类的DAO类。  
但是避免不了在类中持有了成员变量，对于这些有状态的Bean单例模式并不能保证线程的安全性，这样的话最好使用ThreadLocal或者对操作加锁，更简单的方法就是将scope改为`prototype`。

#### Bean的延迟初始化

默认情况下Bean是在项目启动时全部初始化完成，但是如果想加快启动速度，就可以让Bean延迟初始化，也就是用到它的时候再创建，这个配置很少用到。

在xml中的配置方式：

    <bean id="lazy" class="com.foo.ExpensiveToCreateBean" lazy-init="true"/>
    <bean name="not.lazy" class="com.foo.AnotherBean"/>

也可以指定所有的Beans：

    <beans default-lazy-init="true">
    <!-- no beans will be pre-instantiated... -->
    </beans>

使用延迟初始化时，如果一个bean是另一个 **非延迟初始化单例bean所依赖的实例**，那么延迟初始化的设置对它无效。

#### Bean的生命周期回调

实现Bean初始化回调和销毁回调各有三种方法，分别是实现接口、xml配置文件和注解的方式。

* 实现`InitializingBean`和`DisposableBean`接口

        public class AnotherExampleBean implements InitializingBean {
                public void afterPropertiesSet() {
                        // do some initialization work
                }
        }

        public class AnotherExampleBean implements DisposableBean {
                public void destroy() {
                        // do some destruction work (like releasing pooled connections)
                }
        }

* 使用xml配置文件

        <bean id="exampleInitBean" class="examples.ExampleBean" init-method="init"/>

        public class ExampleBean {
                public void init() {
                        // do some initialization work
                }
        }

        <bean id="exampleInitBean" class="examples.ExampleBean" destroy-method="cleanup"/>

        public class ExampleBean {
                public void cleanup() {
                        // do some destruction work (like releasing pooled connections)
                }
        }

* 使用 `@PostConstruct` 和 `@PreDestroy` 注解

        public class CachingMovieLister {
        	@PostConstruct
        	public void populateMovieCache() {
        		// populates the movie cache upon initialization...
        	}

        	@PreDestroy
        	public void clearMovieCache() {
        		// clears the movie cache upon destruction...
        	}
        }

### 基于注解的配置

上面所用到的大多是基于xml的配置，这一节来看一下如何用Annotation的方式实现这些功能。

首先，使用注解之前要配置好xml：

    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/context
    http://www.springframework.org/schema/context/spring-context.xsd">
        <context:annotation-config/>
    </beans>

* @Autowired

    自动装配，默认是byType。

* @Qualifier

    和@Autowired搭配使用，指定名字，默认是类名首字母小写。

* @Resource

    和@Autowired+@Qualifier实现同样的功能，默认byName，不是Spring提供，在`javax.annotation.Resource`包中。

* @Component

    首先需要在配置文件中加入：

        <context:component-scan base-package="org.example"/>

    代替xml声明Bean，在Spring中，加上@Component、@Repository、@Service、@Controller四种注解的类都会被看作是由Spring管理的组件。其中Component是最基础的类型，其他三种有着各自的用途，相对于Component更具体，分别对应着持久层、业务层和表示层。在不同的层中可以用他们代替@Component。

* @Scope ——Bean的作用范围
* @PostConstruct @PreDestroy ——Bean的生命周期回调

关于注解更详细的说明可以查看官方文档，在写项目时更倾向于使用注解的方式。

### 参考资料
>《Spring Framework Reference》  
https://www.zhihu.com/question/23277575/answer/169698662  
https://blog.csdn.net/qq_38663729/article/details/80438680

---
