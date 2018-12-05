1.背景
现代互联网充斥着各种攻击、病毒、钓鱼、欺诈等手段，层出不穷。对于一个公司而已最基本的财富无非是代码和数据，“配置属性加密”的应用场景假设如果攻击者通过某些手段拿到部分敏感代码或配置，甚至是全部源代码和配置时，我们的基础设施账号依然不被泄漏。当然手段多种多种多样，比如以某台中毒的内网机器为肉机，对其他电脑进行ARP攻击抓去通信数据进行分析，或者获取某个账号直接拿到源代码或者配置，等等诸如此类。

2.思路
采用比较安全的对称加密算法；
对基础设施账号密码等关键信息进行加密；
构建时、运行时传入密钥，在加载属性前进行解密；
开发环境可以将密钥放置在代码中，测试、灰度、生产等环境放置在构建脚本或者启动脚本中；
如果自动化部署甚至可以有专门的程序来管理这些密钥（目前没有，暂不考虑）；

值得思考的问题？

当内网被攻破，当源码泄漏，当zkui被攻破，当开发账号泄漏，当运维账号泄漏，我们如何保证生产环境的基础设施安全，如何防范关键信息泄漏？

3.技术框架
Jasypt是一个优秀的加密库，支持密码、Digest认证、文本、对象加密，此外密码加密复合RFC2307标准。http://www.jasypt.org/download.html
ulisesbocchio/jasypt-spring-boot，集成Spring Boot，在程序引导时对属性进行解密。https://github.com/ulisesbocchio/jasypt-spring-boot
4.处理过程
1.添加maven依赖引入jasypt

<!-- https://mvnrepository.com/artifact/com.github.ulisesbocchio/jasypt-spring-boot-starter -->
<dependency>
    <groupId>com.github.ulisesbocchio</groupId>
    <artifactId>jasypt-spring-boot-starter</artifactId>
    <version>1.8</version>
</dependency>

java -cp jasypt-1.9.2.jar org.jasypt.intf.cli.JasyptPBEStringDecryptionCLI input="bWgq4JW/jLTBJ5kuUrR0e3s0JWEt5E7W" password=ACMP10171215 algorithm=PBEWithMD5AndDES
	  
java -cp jasypt-1.9.2.jar org.jasypt.intf.cli.JasyptPBEStringEncryptionCLI input="bWgq4JW/jLTBJ5kuUrR0e3s0JWEt5E7W" password=ACMP10171215 algorithm=PBEWithMD5AndDES



2.配置加密参数 

jasypt:
  encryptor:
    #这里可以理解成是加解密的时候使用的密钥
    password: password
写一个测试方法，这里直接在单元测试里面来实现给密码加密，得到字符串密码

@Autowired
StringEncryptor stringEncryptor;
 
@Test
public void encryptPwd() {
    String result = stringEncryptor.encrypt("yourpassword");
    System.out.println("=================="); 
    System.out.println(result); 
    System.out.println("==================");
}
把得到的密文写到需要使用到的地方，加密后的字符串需要放到ENC里面，格式如下：
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/test?useUnicode=true&characterEncoding=utf8&autoReconnect=true&allowMultiQueries=true&zeroDateTimeBehavior=convertToNull&tinyInt1isBit=false
    username: root
    password: ENC(4TyrSSgQd2DCHnXVwkdKMQ==)
    driver-class-name: com.mysql.jdbc.Driver

5.启动时加载命令模式
 通过命令行运行 jasypt-1.9.2.jar 包命令来加密解密：


1. 在jar包所在目录打开命令行，运行如下加密命令：

java -cp jasypt-1.9.2.jar org.jasypt.intf.cli.JasyptPBEStringEncryptionCLI input="root" password=security algorithm=PBEWithMD5AndDES
运行结果如下：
----ENVIRONMENT-----------------
 
Runtime: Oracle Corporation Java HotSpot(TM) 64-Bit Server VM 25.91-b14 
 
----ARGUMENTS-------------------
 
algorithm: PBEWithMD5AndDES
input: root
password: security
 
----OUTPUT----------------------
 
i00VogiiZ1FpZR9McY7XNw==
 2. 使用刚才加密出来的结果进行解密，执行如下解密命令：

java -cp jasypt-1.9.2.jar org.jasypt.intf.cli.JasyptPBEStringDecryptionCLI input="i00VogiiZ1FpZR9McY7XNw==" password=security algorithm=PBEWithMD5AndD
运行结果如下：

----ENVIRONMENT-----------------
 
Runtime: Oracle Corporation Java HotSpot(TM) 64-Bit Server VM 25.91-b14 
 
----ARGUMENTS-------------------
 
algorithm: PBEWithMD5AndDES
input: i00VogiiZ1FpZR9McY7XNw==
password: security
 
----OUTPUT----------------------
 
root
过程中了解到对Spring Boot的支持是最简单的。如果要是 Spring 项目的话，需要做一些xml中的配置，参考：http://blog.csdn.net/gdfsbingfeng/article/details/16886805
