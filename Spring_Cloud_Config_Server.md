## 配置微服务：挑战 ##

在传统单体式应用中管理配置很直接，配置文件基本位于应用所在服务器上。如果需要更新配置文件，只需要修改属性文件并重启应用就可以。但是对于微服务来说，事情稍微有些复杂。。

微服务由很多小的匿名服务构成，每个都有其配置文件，分布在不同的多个服务器的多个服务中。在生产环境，每个服务都可能有多个实例，配置管理更是很复杂的任务。

云应用会更加复杂，云环境倾向于规模不确定的扩展式应用，这种不确定性给升级和确保正确配置带来挑战。

## 集中管理配置 ##

顾名思义，每个实例运行时从配置集中管理库中获得其配置文件。Spring通过Spring Cloud的子项目Spring Cloud Config完成这一任务。可以通过它创建通过REST API提供服务的Spring Boot 应用。服务通过REST API获得应用配置文件，此文件是保存在Git库中，因此可以进行版本控制。Spring Cloud Config可以选择本地库或者远程库。生产环境中，最好通过访问Git私有库。

## 示例 ##

本篇文章将会配置一个可以从Github上拉配置的服务，将演示一个银行账号服务如何使用其它服务提供的配置。参考如下架构图。源代码可以从[github](https://github.com/briansjavablog/micro-services-spring-cloud-config)上获得。

【图一】

## 创建Cloud Config Service ##

```
@EnableConfigServer
@SpringBootApplication
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

## 配置POM ##

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0                         http://maven.apache.org/xsd/maven-4.0.0.xsd">
   <modelVersion>4.0.0</modelVersion>
   <groupId>com.briansjavablog.microservices</groupId>
   <artifactId>config-server</artifactId>
   <version>0.0.1-SNAPSHOT</version>
   <packaging>jar</packaging>
   <name>config-server</name>
   <description>Demo config server provides centralised configuration for various micro services</description>
   <parent>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-parent</artifactId>
      <version>2.0.3.RELEASE</version>
   </parent>
   <properties>
      <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
      <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
      <java.version>1.8</java.version>
      <spring-cloud.version>Finchley.RELEASE</spring-cloud.version>
   </properties>
   <dependencies>
      <dependency>
         <groupId>org.springframework.cloud</groupId>
         <artifactId>spring-cloud-config-server</artifactId>
      </dependency>
      <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-devtools</artifactId>
         <scope>runtime</scope>
      </dependency>
   </dependencies>
   <dependencyManagement>
      <dependencies>
         <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
         </dependency>
      </dependencies>
   </dependencyManagement>
   <build>
      <plugins>
         <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
         </plugin>
      </plugins>
   </build>
</project>-

```

## 对Config Service进行配置 ##

```
spring.application.name=config-server
server.port=8888
# URI of GIT repo containing properties
spring.cloud.config.server.git.uri=https://github.com/briansjavablog/micro-services-spring-cloud-config
# path to properties from root of repo 
spring.cloud.config.server.git.searchPaths: configuration
logging.level.org.springframework.web=INFO
```

- 第一行，spring.application.name指定应用名。尽管不是必须的，但是最佳实践建议通过这一步定义出现在endpoint上的名字。
- 第二行，server.port指定admin应用运行在哪个端口上
- 第五行，spring.cloud.config.server.git.uri指定包含配置文件的路径。
- 第八航，spring.cloud.config.server.git.seachPaths指定配置文件的绝对路径。

## 运行Cloud Config Service ##

现在可以运行一个测试，可以在Eclipse或者命令行启动应用，并在端口8888上看到它。

【图二】

## 测试 ##

用默认属性调用 *http://localhost:8888/bank-account-service/default*. 

收到请求后，Config Service使用GIT URI复制远程的库。参看下图:

【图三】

Bank-account-service.properties返回配置信息，并以JSON方式显示如下：

```
{
   "name": "bank-account-service",
   "profiles": ["default"],
   "label": null,
   "version": "7b0732778b442726f8dd0bf7d1a36fc00f15c5b8",
   "state": null,
   "propertySources": [{
      "name": "https://github.com/briansjavablog/micro-services-spring-cloud-config/configuration/bank-account-service.properties",
      "source": {
          "bank-account-service.minBalance": "99",
          "bank-account-service.maxBalance": "200"
      }
   }]
}
```

## 创建Bank Account Service ##

现在创建一个bank account service调用以上属性。Bank Account Service有两个服务，一个是创建账号，一个是接受账号。

```
@RestController
@Slf4j
public class BankAccountController {
  @Autowired
  public BankAccountService bankAccountService;
  @PostMapping("/bank-account")
  public ResponseEntity << ? > createBankAccount(@RequestBody BankAccount bankAccount, HttpServletRequest request) throws URISyntaxException {
    bankAccountService.createBankAccount(bankAccount);
    log.info("created bank account {}", bankAccount);
    URI uri = new URI(request.getRequestURL() + "bank-account/" + bankAccount.getAccountId());
    return ResponseEntity.created(uri).build();
  }
  @GetMapping("/bank-account/{accountId}")
  public ResponseEntity<BankAccount> getBankAccount(@PathVariable("accountId") String accountId) {
    BankAccount account = bankAccountService.retrieveBankAccount(accountId);
    log.info("retrieved bank account {}", account);
    return ResponseEntity.ok(account);
  }
}
```

生成一个新账号时，服务会通过一系列最大最小值检查账号是否有结余。

```
/**
 * Add account to cache
 * 
 * @param account
 */
public void createBankAccount(BankAccount account) {
  /* check balance is within allowed limits */
  if(account.getAccountBlance().doubleValue() >= config.getMinBalance() &&
      account.getAccountBlance().doubleValue() <= config.getMaxBalance()) {
    log.info("Account balance [{}] is is greater than lower bound [{}] and less than upper bound [{}]",
        account.getAccountBlance(), config.getMinBalance(), config.getMaxBalance());
    accountCache.put(account.getAccountId(), account);
  }
  else {
    log.info("Account balance [{}] is outside of lower bound [{}] and upper bound [{}]",
        account.getAccountBlance(), config.getMinBalance(), config.getMaxBalance());
    throw new InvalidAccountBalanceException("Bank Account Balance is outside of allowed thresholds");
  }
}
```

最大最小值可以配置，并从输入的Configuration对象读入。

```
@Service
@Slf4j
public class BankAccountService {
  @Autowired
  private Configuration config;
```

Configuration对象的定义如下：

```
@Component
@ConfigurationProperties(prefix="bank-account-service")
public class Configuration {
  @Setter
  @Getter
  private Double minBalance;
  @Setter
  @Getter
  private Double maxBalance;
}
```

Github的截屏显示@Configuration类中的名字和属性值

【图四】

## Bank Account Service本地配置 ##

使用Bank Account Service配置之前， 还是有一些需要本地配置的地方：

```
spring.application.name=bank-account-service
server.port=8080
spring.config.cloud.uri=htp://localhost:8888
spring.cloud.config.profile=uat
management.endpoints.web.exposure.include=*
```

最终调用命令：

```
http://localhost:8888/bank-account-service/uat.
```

## 测试Bank Account Service ##

运行以下命令生成一个bank account:

```
curl -i -H "Content-Type: application/json" -X POST -d '{"accountId":"B12345","accountName":"Joe Bloggs","accountType":"CURRENT_ACCOUNT","accountBlance":1250.38}' localhost:8080/bank-account
```

注意现在的账号余额是1250.38英镑，处在UAT定义账户操作范围之内。

【图五】

从日志中可以看出，账号成功创建，最小最大值分别是501.0和15002.0，符合UAT定义范围。

## 更新服务配置 ##

通过git上传配置，并且能够在客户端启动时候自动获得最新配置信息。但是如果运行中，如何更新配置信息呢？幸运的是Spring Boot提供了一个机制可以在运行实例中更新配置的机制。

```
curl localhost:8080/actuator/refresh -d {} -H "Content-Type: application/json
```

从截屏可以看到，/refresh被调用，可以通过UAT来接收最新属性。

【图六】

## 自动更新服务 ##

尽管可以手动更新配置，但是我们也可以写一个shell自动获取更新。这样可以大大减少把精力投入更新配置的过程中。

## 总结 ##

本篇文章中使用Spring Cloud Config创建一个基于Github的集中配置管理机制。也介绍了管理不同环境下属性的方法。有任何问题，可以和本博主联系。

[原文地址](https://dzone.com/articles/configuring-micro-services-spring-cloud-config-ser)

