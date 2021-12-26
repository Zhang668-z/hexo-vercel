## 消息队列之RabbitMQ

### 一、消息中间件(消息队列)

#### 	1.1MQ

​		MQ：消息队列、消息中间件

​		可以实现进程间或者线程间的数据交互 通信。

​		可以实现进程间或者线程间的数据交互 通信。

​		目前市场主流的MQ：

​		ActiveMQ、RabbitMQ、ZeroMQ、RocketMQ、Kafka

​		实现进程间数据交互的方式：

​		1.面向服务开发

​		2.采用消息队列

​		3.采用WSDL

​			WebService

​		4.自定义通信协议

​			Socket、Nio、Netty等

#### 	1.2RabbitMQ

​		RabbitMQ是部署最广泛的开源信息队列
​		https://www.rabbitmq.com/	

![](E:/Typora/图片资料/01.png)

​		RabbitMQ在企业中使用最为广泛，主要用来实现异步通信、解耦。无论是单体项目还是微服务项目。

### 二、RabbMQit初体验

#### 2.1 安装RabbitMQ

```html
docker run -d --name rabbitmq -p 15671:15671 rabbitmq:management	
```
如果镜像不存在会自动下载
访问测试：localhost:15671
默认的账号密码：账号：guest 密码：guest

#### 2.2 SpringBoot整合Rabbit

##### 	1.依赖jar

##### 	2.编写配置

##### 	3.修改配置文件

##### 	4.编写消息发送者

##### 	5.编写消息监听器

##### 	6.运行测试项目

### 三、RabbitMQ消息模式

#### 3.1模式简介

##### RabbitMQ提供三种模式：

##### 1.普通消息 点对点消息

​	一个消息提供者对应一个消息接收者

##### 2.Worker模式

​	一个消息提供者对应多个消息接受者

##### 3.Exchange模式 	交换器

​	可以通过路由匹配实现消息的一对多  就是一个消息发送到多个队列

##### 交换器的类型：4种

​	1.Direct

​	2.fanout

​	3.topic

​	4.headers



#### 3.2普通消息

##### 3.2.1实现点对点消息：

##### 3.2.2代码演示：

##### 3.2.3实现步骤：

1.定义队列

2.发送消息---->到指定队列

3.消息监听---->监听指定的队列的消息变化



#### 3.3Worker模式

3.3.1  Worker模式是RabbitMQ给出的应对高并发下，发送消息大于消费消息频率的问题，防止消息过量推挤。

3.3.2  一个消息发送者对应多个消息消费者，多个消息消费者之间进行随机消费队列的消息

3.3.3  一个消息被消费一次

3.3.4  发送者的发送频率大于消费者的消费频率。

核心代码演示：

发送者：

```java
@RestController 
public class MqSendController02 {    
    @Autowired    
    private RabbitTemplate rabbitTemplate;    
    @Autowired    
    private IdGenerator generator;    
    @GetMapping("/api/mq2/sendmsg/{msg}")    
    public String sendMsg(@PathVariable("msg") String msg){        
        System.out.println("消息发送者--->"+msg);        
        MqMsg mqMsg=new MqMsg(generator.nextId(),"Worker模式",msg);   
        rabbitTemplate.convertAndSend("qname_912_worker",JSON.toJSONString(mqMsg));     
        return "OK";    
    } 
}
```

消费者01：

```java
@Component 
@RabbitListener(queues = "qname_912_worker")  //标记要去监听的队列 
public class MqConsumer02 {
	@RabbitHandler //标记对应的自动接收队列消息的函数    
	public void msg(String msg){        
    	if(msg!=null && msg.length()>0){            
        	MqMsg mqMsg= JSON.parseObject(msg,MqMsg.class);   
        	System.out.println("消费者001-->："+mqMsg);        
    	}    
	} 
}
```

#### 3.4 exchange交换器模式

Pub/Sub发布订阅：可以实现一个消息被消费两次 基于Exchange实现的发布订阅

提供4种类型的交换器：

1. fanout直接转发
2. direct路由匹配转发
3. topic模糊路由匹配转发 
4. headers通过消息头匹配转发消息



##### 3.4.1  fanout

![](E:/Typora/图片资料/02.png)



fanout模式：提供直接转发消息，可以将接受的消息直接转发到所有绑定的队列中

代码演示：

配置代码：

```java
@Configuration 
public class RabbitMQConfig {    
    //演示fanout    
    public String qname03=&quot;qname_912_fanout01&quot;    
    public String qname04=&quot;qname_912_fanout02&quot;    
    
    //创建队列    
    @Bean    
    public Queue createQ3(){        
        return new Queue(qname03);    
    }
    
    @Bean    
    public Queue createQ4(){        
        return new Queue(qname04);    }    
    
    //交换器    
    public String exname01=&quot;ex_912_fanout&quot;    
    @Bean    
    public Exchange createEx1(){        
        return new FanoutExchange(exname01);    
    }
    
    //将队列和交换器进行绑定    
    @Bean    public Binding createBd1(FanoutExchange ex){        
        return BindingBuilder.bind(createQ3()).to(ex);    
    }
    
    //将队列和交换器进行绑定    
    @Bean    public Binding createBd2(FanoutExchange ex){        
        return BindingBuilder.bind(createQ4()).to(ex);    
    }

}
```

发送消息：

```java
@RestController 
public class MqSendFanout_Api {    
    @Autowired    
    private RabbitTemplate rabbitTemplate;    
    
    @GetMapping("api/exfanout/sendmsg/{msg}")    
    public String sm(@PathVariable String msg){       			                            
        rabbitTemplate.convertAndSend("ex_912_fanout",null,msg);                           
        return "OK";                                               
     } 
}
```

消费者 监听队列：

```java
@Component 
@RabbitListener(queues = "qname_912_fanout01")  //标记要去监听的队列 
public class MqFanoutConsumer01 {
    @RabbitHandler //标记对应的自动接收队列消息的函数    
    public void msg(String msg){        
    	System.out.println("fanout--01-->消费者："+msg);    
    } 
}

```





##### 3.4.2 direct

转发消息 可以根据路由关键字进行匹配。 只往匹配成功的队列中发送数据

![](E:/Typora/图片资料/03.png)

代码演示：

RabbitMQ配置：

```java
//创建队列 
private String qname05="qname_912_direct01"; 
private String qname06="qname_912_direct02"; 
private String qname07="qname_912_direct03"; 

@Bean public Queue createQ5(){    
    return new Queue(qname05);
} 

@Bean public Queue createQ6(){    
    return new Queue(qname06); 
} 

@Bean public Queue createQ7(){    
    return new Queue(qname07); 
} 

//创建交换器 
public String exname02="ex_912_direct"; 
@Bean public Exchange createEx2(){    
    return new DirectExchange(exname02); 
} 
//绑定交换器和队列  指定转发关键字（路由） 
@Bean public Binding createBd3(DirectExchange ex){    
    return BindingBuilder.bind(createQ5()).to(ex).with("log"); 
} 

@Bean public Binding createBd4(DirectExchange ex){    
    return BindingBuilder.bind(createQ6()).to(ex).with("log"); } 

@Bean public Binding createBd5(DirectExchange ex){    
    return BindingBuilder.bind(createQ7()).to(ex).with("error"); 
}
```

发送者：

```java
@RestController 
public class MqSendDirect_Api {    
	@Autowired    
	private RabbitTemplate rabbitTemplate;    						        				
    @GetMapping("api/exdirect/sendmsg/{msg}/{t}")    
    public String sm(@PathVariable String msg,@PathVariable String t){        
        //参数说明：1.交换器名称 2.路由关键字 匹配关键字 完整匹配 3.消息        
        rabbitTemplate.convertAndSend("ex_912_direct",t,msg);        
        return "OK";    
    } 
}
```



##### 3.4.3topic

通过路由匹配消息进行转发 支持模糊匹配

>*匹配一个单词
>
>#匹配多个或零个单词

代码演示：

RabbitMQ的配置：

```java
//演示Topic 
//创建队列 
private String qname08="qname_912_topic01"; 
private String qname09="qname_912_topic02"; 
private String qname10="qname_912_topic03"; 
@Bean public Queue createQ8(){    
    return new Queue(qname08); 
} 

@Bean public Queue createQ9(){    
    return new Queue(qname09); 
} 

@Bean public Queue createQ10(){    
    return new Queue(qname10); 
}

//创建交换器 
public String exname03="ex_912topic"; 
@Bean public Exchange createEx3(){    
    return new TopicExchange(exname03); 
} 

//绑定交换器和队列  指定转发关键字（路由） 
@Bean public Binding createBd6(TopicExchange ex){    
    return BindingBuilder.bind(createQ8()).to(ex).with("log.#"); 
} 

@Bean public Binding createBd7(TopicExchange ex){    
    return BindingBuilder.bind(createQ9()).to(ex).with("tea.*"); 
}

@Bean public Binding createBd8(TopicExchange ex){    
    return BindingBuilder.bind(createQ10()).to(ex).with("stu"); 
}
```

消息发送者：

```java
@RestController 
public class MqSendTopic_Api {    
    @Autowired    
    private RabbitTemplate rabbitTemplate;    
    @GetMapping("api/extopic/sendmsg/{msg}/{t}")    
    public String sm(@PathVariable String msg,@PathVariable String t){        
        //参数说明：1.交换器名称 2.路由关键字 匹配关键字 完整匹配 3.消息        
        rabbitTemplate.convertAndSend("ex_912topic",t,msg);        
        return "OK";    
    } 
}
```

### 四、RabbitMQ实现延迟消息

#### 4.1实现延迟处理

多种方法：

1.基于定时任务Spring Task轮巡

2.Redis失效监听

3.RabbitMQ延迟消息（借助RabbitMQ的死信队列实现）



#### 4.2RabbitMQ死信

死信：RabbitMQ可以为队列中的消息设置有效期 如果在有效期内没有进行消息消费 那么此消息会成为死信消息。可以借助交换器自动的将死信消息发到指定死信队列

实际开发中，就可以借助死信实现延迟消息投递

例如：订单超时自动处理 自动评价等需要一定时间来处理的



#### 4.3基于RabbitMQ实现延迟队列

配置：

```java
@Configuration 
public class MyRabbitConfig {    
    //定义交换器 fanout    
    public String ex01="ex_order_912";
     public String ex02="ex_dead_912";    
    //定义队列    
    public String qn01="q_maxtime_912";
    //带有效期的队列 默认30秒    
    public String qn02="q_dead_912";
    
    //死信队列    
    //死信交换器的路由名称    
    public String rk01="dead_order_912";
 
    @Bean    
    public Queue createq1(){        
        Map<String,Object> params=new HashMap<>();        
        //30秒        
        params.put("x-message-ttl",30000); //设置队列中消息的有效期        
        //死信交换器的名称        
        params.put("x-dead-letter-exchange","ex_dead_912");        
        //设置死信交换器的匹配路由        
        params.put("x-dead-letter-routing-key","dead_order_912");
 
        //return new Queue(qn01,true,false,true,params);        
        return QueueBuilder.durable(qn01).withArguments(params).build();    
    }
 
    @Bean    
    public Queue createq2(){        
        return new Queue(qn02);    
    }    
    //创建交换器    
    @Bean    
    public Exchange createex01(){        
        return new FanoutExchange(ex01);    
    }    
    //创建绑定    
    @Bean    
    public Binding createBd01(FanoutExchange ex){        
        return BindingBuilder.bind(createq1()).to(ex);    
    }
    
    //创建死信交换器    
    @Bean    
    public Exchange cteateex02(){        
        return new DirectExchange(ex02);    
    }    
    //创建绑定    
    @Bean    
    public Binding createBd02(DirectExchange ex){        
        return BindingBuilder.bind(createq2()).to(ex).with(rk01);    
    } 
}
```



发送消息：

```java
@RestController 
public class MeDeadController {    
    @Autowired    
    private RabbitTemplate rabbitTemplate;
 
    @GetMapping("/api/mqdead/sendmsg/{msg}")    
    public String ms(@PathVariable String msg){        
        System.out.println("发送延迟消息--->"+msg+"--->"+System.currentTimeMillis()/1000);   
        rabbitTemplate.convertAndSend("ex_order_912",null,msg);        
        return "死信消息";    
    } 
}
```

消息消费：

```java
@Component 
@RabbitListener(queues = "q_dead_912")  
//标记要去监听的队列 
public class MqDlxConsumer01 {
 
    @RabbitHandler //标记对应的自动接收队列消息的函数    
    public void msg(String msg){        
       System.out.println("延迟消息--01-->消费者："+msg+"-->"+System.currentTimeMillis()/1000
    } 
}
```









