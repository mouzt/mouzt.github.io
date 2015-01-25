---
layout: post
title: "Spring静态变量注入"
tagline: "Spring静态变量注入"
description: "Spring静态变量注入"
tags: [Spring, java ]
---
{% include JB/setup %}

##Spring静态注入两种实现方式

###第一种方法

    @Component
    public class SecretInfoHandler {

        private static InfoDecryptClient infoDecryptClient;

        private static InfoEncryptClient infoEncryptClient;

        @Autowired(required = true)
        public SecretInfoHandler(@Qualifier("infoDecryptClient") InfoDecryptClient infoDecryptClient,
            @Qualifier("infoEncryptClient") InfoEncryptClient infoEncryptClient) {

            SecretInfoHandler.infoDecryptClient = infoDecryptClient;
            SecretInfoHandler.infoEncryptClient = infoEncryptClient;

        }

    }

###第二种方法

    @Autowired
    public void setInfoDecryptClient(InfoDecryptClient infoDecryptClient) {
        SecretInfoHandler.infoDecryptClient = infoDecryptClient;
    }

    @Autowired
    public void setInfoEncryptClient(InfoEncryptClient infoEncryptClient) {
        SecretInfoHandler.infoEncryptClient = infoEncryptClient;
    }

###@Qualifier

@Qualifier("XXX") 中的 XX是 Bean 的名称，所以 @Autowired 和 @Qualifier 结合使用时，自动注入的策略就从 byType 转变成 byName 了。
@Autowired 可以对成员变量、方法以及构造函数进行注释，而 @Qualifier 的标注对象是成员变量、方法入参、构造函数入参。

####自定义@Qualifier

例如定义一个交通工具类：Vehicle，以及它的子类Bus和Sedan。
如果用@Autowired来找Vehicle的话，会有两个匹配的选项Bus和Sedan。为了限定选项，可以象下面这样。
  @Autowired
  @Qualifier("car")
  private Vehicle vehicle;
如果要频繁使用@Qualifier("car")并且想让它变得更有意义，我们可以自定义一个@Qualifier。
  @Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.TYPE})
  @Retention(RetentionPolicy.RUNTIME)
  @Qualifier
  public @interface Car{
  }

  @Autowired
  @Car
  private Vehicle vehicle;
最后在Sedan类加上注释。
  @Car
  public class Sedan implements Vehicle{
  }
