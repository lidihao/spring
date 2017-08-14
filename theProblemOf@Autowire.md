# 使用@Autowired自动注入时,出现空指针问题

标签（空格分隔）： spring框架

---
1 .Spring java配置及注解注入方法出现空指针异常的原因

在使用Spring进行自动注入的过程中，只会对通过读取Spring的配置文件或者配置类后产生的实例进行自动注入。 
手动new出来的实例是无法获得在Spring中注册过得实例，这是 因为手动new 的实例并不是Spring 在初始化过程中注册的实例。

通过下面的例子来解释一下问题所在： 
定义一个接口类MessageService

package hello;

public interface MessageService {
    String getMessage();
}
定义一个实现类MessagePrinter，使用@Autowired通过Type的方式在这个Bean中自动装配service
package hello;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

public class MessagePrinter {

    final private MessageService service;

    @Autowired
    public MessagePrinter(MessageService service) {
        this.service = service;
    }

    public void printMessage() {
       System.out.println(this.service.getMessage());                        
    }
}
最后是测试类，使用@Configuration告诉Spring这个类是一个配置类，并且通过@ComponentScan自动扫描同一个包内的其他文件，将符合条件的类自动注册到Spring的容器中。然后使用@Bean实现了一个MessageService的接口。
package hello;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.*;

@Configuration
@ComponentScan
public class Application {

    @Bean
    MessageService mockMessageService() {
        return new MessageService() {
            public String getMessage() {
              return "Hello World!";
            }
        };
    }

  public static void main(String[] args) {
      ApplicationContext context = 
          new AnnotationConfigApplicationContext(Application.class);
          System.out.println(context.containsBean("messageService"));                              
      MessagePrinter printer = new MessagePrinter();
      printer.printMessage();
  }
}
通过这个测试类的main方法，使用AnnotationConfigApplicationContext（）方法加载了Application.class用来初始化Spring完成对所有组件的注册。
此时我们containsBean()的方法验证了一下是否所有实例都已经注册在Spring中了。然后新建一个MessagePrinter并打印结果，运行结果如下：

true 
java.lang.NullPointerException

at hello.MessagePrinter.show(MessagePrinter.java:16) 
at hello.Application.main(Application.java:24)

发生了空指针异常。 
根据异常的反馈发现错误在这一句：

System.out.println(this.service.getMessage()); 
也就是没有找到MessageService service 这个实例。

这就很奇怪了，从打印的true中可以看出MessageService 已经成功注册到Spring容器中，可是却出现了空指针异常这是为什么呢？

经过仔细看Spring官方的例子发现了问题所在： 
当通过new的方式创建一个MessagePrinter对象的时候，虽然期望使用了注解@Autowired对这个对象进行装配，但是Spring是不会这么做的，因为Spring不会对任意一个MessagePrinter进行自动装配，只有MessagePrinter也是一个在Spring中注册过的Bean，才会获得自动装配的功能。

理解了这个问题，修改一下前面的代码：

在MessagePrinter前面增加一个@Component来注册一个Bean

package hello;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class MessagePrinter {

    final private MessageService service;

    @Autowired
    public MessagePrinter(MessageService service) {
        this.service = service;
    }

    public void printMessage() {
        System.out.println(this.service.getMessage());
    }
}
运行后发现结果依然异常：

true 
java.lang.NullPointerException 
at hello.MessagePrinter.show(MessagePrinter.java:17)

at hello.Application.main(Application.java:24)

这是因为，Spring默认都是单例的，new出来的对象，Spring依然不会对它进行装配，只有通过Spring创建的对象才会获得自动装配的功能，所以再改一下最后的测试代码：

把new的printer改为从ApplicationContext对象中创建对实例：

package hello;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.*;

@Configuration
@ComponentScan
public class Application {

    @Bean
    MessageService mockMessageService() {
        return new MessageService() {
            public String getMessage() {
              return "Hello World!";
            }
        };
    }

  public static void main(String[] args) {
      ApplicationContext context = 
          new AnnotationConfigApplicationContext(Application.class);
System.out.println(context.containsBean("messageService"));
      MessagePrinter printer = context.getBean(MessagePrinter.class);
      printer.printMessage();
  }
}
最后来看看结果，终于看到了久违的“Hello World!” 
true

Hello World!





