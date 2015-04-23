# Binding to Data Services with Spring Boot in Cloud Foundry

In this article we look at how to bind a Spring Boot application to data services (JDBC, NoSQL, messaging etc.) and the various sources of default and automatic behaviour in Cloud Foundry, providing some guidance about which ones to use and which ones will be active under what conditions. Spring Boot provides a lot of autoconfiguration and external binding features, some of which are relevant to Cloud Foundry, and many of which are not. Spring Cloud Connectors is a library that you can use in your application if you want to create your own components programmatically, but it doesn't do anything "magical" by itself. And finally there is the Cloud Foundry buildpack which has an "auto-reconfiguration" feature that tries to ease the burden of moving simple applications to the cloud. The key to correctly configuring middleware services, like JDBC or AMQP or Mongo, is to understand what each of these tools provides, how they influence each other at runtime, and and to switch parts of them on and off. The goal should be a smooth transition from local execution of an application on a developer's desktop to a test environment in Cloud Foundry, and ultimately to production in Cloud Foundry (or otherwise) with no changes in source code or packaging, per the [twelve-factor application](http://12factor.net) guidelines.

There is some [simple source code](https://github.com/dsyer/cloud-middleware-blog) accompanying this article. To use it you can clone the repository and import it into your favourite IDE. You will need to remove two dependencies from the complete project to get to the same point where we start discussing concrete code samples, namely `spring-boot-starter-cloud-connectors` and `auto-reconfiguration`.

## Layers of Autoconfiguration

Let's take a a simple app with `DataSource` (similar considerations apply to RabbitMQ, Mongo, Redis, Elasticsearch, etc.):

```java
@SpringBootApplication
public class CloudApplication {
	
	@Autowired
	private DataSource dataSource;
	
	public static void main(String[] args) {
		SpringApplication.run(CloudApplication.class, args);
	}

}
```

This is a complete application: the `DataSource` can be `@Autowired` because it is created for us by Spring Boot. The details of the `DataSource` (concrete class, JDBC driver, connection URL, etc.) depend on what is on the classpath. Let's assume that the application uses Spring JDBC via the `spring-boot-starter-jdbc` (or `spring-boot-starter-data-jpa`), so it has a `DataSource` implementation available from Tomcat (even if it isn't a web application), and this is what Spring Boot uses.

Consider what happens when:

* Classpath contains H2 (only) in addition to the starters: the `DataSource` is the Tomcat high-performance pool from `DataSourceAutoConfiguration` and it connects to an in memory database "testdb".

* Classpath contains H2 and MySQL: `DataSource` is still H2 (same as before) because we didn't provide any additional configuration for MySQL and Spring Boot can't guess the credentials for connecting.

* Add `spring-boot-starter-cloud-connectors` to the classpath: no change in `DataSource` because the Spring Cloud Connectors do not detect that they are running in a Cloud platform. The providers that come with the starter all look for specific environment variables, which they won't find unless you set them, or run the app in Cloud Foundry, Heroku, etc.

* Run the application in "cloud" profile with `spring.profiles.active=cloud`: no change yet, but this is one of the things that [Java buildpack](https://github.com/cloudfoundry/java-buildpack) is looking for as a "tell" that the app is running in Cloud Foundry. The "cloud" profile in Cloud Foundry is provided to Spring applications via the auto-reconfiguration jar that is added to the classpath by the Java buildpack.

* Run in "cloud" profile and provide some environment variables simulating running in Cloud Foundry and binding to a MySQL service:
```
VCAP_APPLICATION={"name":"application","instance_id":"FOO"}
VCAP_SERVICES={"mysql":[{"name":"mysql","tags":["mysql"],"credentials":{"uri":"jdbc:mysql://localhost/test"}}]}
```
(the "tags" provides a hint that we want to create a MySQL `DataSource`, the "uri" provides the location, and the "name" becomes a bean ID). Still nothing changes in the `DataSource` (it is still H2) because Spring Cloud Connectors by themselves don't create a `DataSource`. Spring Boot has some autoconfiguration for the Connectors, so if you looked at the beans in your application you would see a `CloudFactory`, but it's up to you to use that if you want it to create a `DataSource` for you (see below).

* Add the "auto-reconfiguration" JAR from the Java buildpack (Maven co-ordinates `org.cloudfoundry:auto-reconfiguration:1.7.0.RELEASE`). You can add it as a local dependency to simulate running an application in Cloud Foundry, but it wouldn't be normal to do this with a real application (this is just for experimenting with autoconfiguration). The auto-reconfiguration JAR now has everything it needs to create a `DataSource`, but it still doesn't (yet) because it detects that you already have a bean of type `CloudFactory`, one that was added by Spring Boot.

* Remove the explicit "cloud" profile. The profile will still be active when your app starts because the auto-reconfiguration JAR adds it back again. There is still no change to the `DataSource` because Spring Boot has created a `CloudFactory` for you.

* Remove the `spring-boot-starter-cloud-connectors` dependency, so that Spring Boot backs off creating a `CloudFactory`. The auto-reconfiguration JAR actually has its own copy of Spring Cloud Connectors (all the classes with different package names) and it now uses them to create a `DataSource` (in a `BeanFactoryPostProcessor`). Spring Boot also backs off its `DataSourceAutoConfiguration` and you have a different `DataSource`, finally binding to MySQL via the `VCAP_SERVICES`. There is no control over pool properties, but it does still use the Tomcat pool if available (no support for Hikari or DBCP2).

* Remove the auto-reconfiguration JAR and the `DataSource` reverts to H2.

> TIP: use web and actuator starters with `endpoints.health.sensitive=false` to inspect the `DataSource` quickly through "/health". You can also use the "/beans", "/env" and "/autoconfig" endpoints to see what is going in in the autoconfigurations and why.

> NOTE: Running in Cloud Foundry or including auto-reconfiguration JAR in classpath locally both activate the "cloud" profile (for the same reason). The `VCAP_*` env vars are the thing that makes Spring Cloud and/or the auto-reconfiguration JAR create beans.

The auto-reconfiguration JAR is always on the classpath in Cloud Foundry (by default) but it backs off creating any `DataSource` if it finds a `org.springframework.cloud.CloudFactory` bean (which is provided by Spring Boot). Thus the net effect of adding it to the classpath, if the Connectors are also present in a Spring Boot application, is only to enable the "cloud" profile automatically. You can see it making the decision to skip auto-reconfiguration in the application logs on startup:

```
015-04-14 15:11:11.765  INFO 12727 --- [           main] urceCloudServiceBeanFactoryPostProcessor : Skipping auto-reconfiguring beans of type javax.sql.DataSource
2015-04-14 15:11:57.650  INFO 12727 --- [           main] ongoCloudServiceBeanFactoryPostProcessor : Skipping auto-reconfiguring beans of type org.springframework.data.mongodb.MongoDbFactory
2015-04-14 15:11:57.650  INFO 12727 --- [           main] bbitCloudServiceBeanFactoryPostProcessor : Skipping auto-reconfiguring beans of type org.springframework.amqp.rabbit.connection.ConnectionFactory
2015-04-14 15:11:57.651  INFO 12727 --- [           main] edisCloudServiceBeanFactoryPostProcessor : Skipping auto-reconfiguring beans of type org.springframework.data.redis.connection.RedisConnectionFactory
...
etc.
```

## Create your own DataSource

The last section walked through most of the important autoconfiguration features in the various libraries. If you want to take control yourself, one thing you start with is to create your own instance of `DataSource`. You could do that, for instance, using a `DataSourceBuilder` which is a convenience class and comes as part of Spring Boot (it chooses an implementation based on the classpath):

```java
@SpringBootApplication
public class CloudApplication {
	
	@Bean
	public DataSource dataSource() {
		return DataSourceBuilder.create().build();
	}
	
	...

}
```

The `DataSource` as we've defined it is useless because it doesn't have a connection URL or any credentials, but that can easily be fixed. Let's run this application as if it was in Cloud Foundry: with the `VCAP_*` environment variables and the auto-reconfiguration JAR but not Spring Cloud Connectors on the classpath and no explicit "cloud" profile. The auto-reconfiguration JAR creates a `DataSource` `@Bean` and binds it to the `VCAP_SERVICES` because it found a service of type "mysql" (per the tags). It removes your `DataSource` completely and replaces it with a manually registered singleton (which doesn't show up in the "/beans" endpoint in Spring Boot).

You can disable the Spring Boot autoconfiguration and Java buildpack auto-reconfiguration by creating your own `Cloud` instance as a `@Bean`:

```java
@Bean
public Cloud cloud() {
  return new CloudFactory().getCloud();
}

@Bean
@ConfigurationProperties(DataSourceProperties.PREFIX)
public DataSource dataSource() {
  return cloud().getSingletonServiceConnector(DataSource.class, null);
}
```

(If you have the Connectors on the classpath you can disable the Spring Boot autoconfiguration of the `CloudFactory` with `spring.cloud.enabled=false`, or by defining a `@Bean` of type `Cloud`. The buildpack auto-reconfiguration backs off if it finds that `@Bean` as well.)

Pros: The auto-reconfiguration JAR backed off so there is only one `DataSource`, and it can be tweaked using `application.properties` via `spring.datasource.*` properties, per the [Spring Boot User Guide](http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/).

Cons: It doesn't work without `VCAP_*` environment variables. It also relies on user remembering to ceate the `Cloud` as a `@Bean` in order to disable the autoconfiguration.

Summary: we are still not in a comfortable place (an app that doesn't run without some intricate wrangling of environment variables is not much use in practice).

## Dual Running: Local with H2, in the Cloud with MySQL

As mentioned briefly above, there is a local configuration file option in Spring Cloud Connectors, but it's awkward to set up despite being boiler plate, and you also have to somehow switch it off when you are in a real cloud platform. The last point there is really the important one because you end up needing a local file to run locally, but only running locally, and it can't be packaged with the rest of the application code (for instance violates the twelve factor guidelines).

So to move forward with our explicit `@Bean` definition it's probably better to stick to mainstream Spring and Spring Boot features, e.g. using the "cloud" profile to guard the explicit creation of a `DataSource`:

```java
@Configuration
@Profile("cloud")
public class DataSourceConfiguration {

  @Bean
  public Cloud cloud() {
    return new CloudFactory().getCloud();
  }

  @Bean
  @ConfigurationProperties(DataSourceProperties.PREFIX)
  public DataSource dataSource() {
    return cloud().getSingletonServiceConnector(DataSource.class, null);
  }

}
```

With this we have a solution that works smoothly both locally and in Cloud Foundry. Locally Spring Boot will create a `DataSource` with an H2 embedded database. In Cloud Foundry it will bind to a singleton service of type `DataSource`. It also has the benefit of working with any platform supported by Spring Cloud Connectors, so the same code will run on Heroku and Cloud Foundry, for instance.

If you don't want to use H2 or some other in-memory database locally, then you can't really avoid having to configure it (Spring Boot can guess a lot from the URL, but it will need that at least). So at a minimum you need to set some `spring.datasource.*` properties (the URL for instance), but that isn't hard to do, and you can easily set different values in different environments using additional profiles. Thus with no changes to code you can be running locally with an embedded or non-embedded database, and also in the cloud with the non-embedded version.

## Manually Creating a Local and a Cloud DataSource

If you like creating `DataSource` beans, and you want to do it both locally and in the cloud, you can use 2 profiles ("cloud" and "local"), for example:

```java
@Configuration
@Profile("local")
public class LocalDataSourceConfiguration {
	
	@Bean
    @ConfigurationProperties(DataSourceProperties.PREFIX)
	public DataSource dataSource() {
		return DataSourceBuilder.create().build();
	}

}

@Configuration
@Profile("cloud")
public class CloudDataSourceConfiguration {

  @Bean
  public Cloud cloud() {
    return new CloudFactory().getCloud();
  }

  @Bean
  @ConfigurationProperties(DataSourceProperties.PREFIX)
  public DataSource dataSource() {
    return cloud().getSingletonServiceConnector(DataSource.class, null);
  }

}
```

Now you just need a way to switch the "local" one on when the "cloud" one is off, and the "cloud" one off when the "local" one is on. For example:

```java
public static void main(String[] args) {
  SpringApplicationBuilder builder = new SpringApplicationBuilder(
      CloudApplication.class);
  try {
    new CloudFactory().getCloud();
  }
  catch (CloudException e) {
    builder.profiles("local");
  }
  builder.run(args);
}
```

or, if we know that we are only going to run in Cloud Foundry, this would work:

```java
public static void main(String[] args) {
  SpringApplicationBuilder builder = new SpringApplicationBuilder(
      CloudApplication.class);
  Environment environment = new StandardEnvironment();
  if (!environment.acceptsProfiles("cloud") || environment.getProperty("VCAP_APPLICATION")!=null) {
    builder.profiles("local");
  }
  builder.run(args);
}
```

The "local" `DataSource` is actually identical to the autoconfigured one in this simple example, so you wouldn't do this unless you needed to, e.g. to create a custom concrete `DataSource` of a type not supported by Spring Boot. You might think it's all getting a bit complicated, but in fact Spring Boot is not making it any harder, we are just dealing with the consequences of needing to control the `DataSource` construction in 2 environments.

## A Purely Declarative Approach

If you prefer not to write Java code, or don't want to use Spring Cloud Connectors, it is also completely feasible to do everything for most applications via Spring Boot autoconfiguration and external properties (or YAML) files. For example Spring Boot creates a `DataSource` for you if it finds the right stuff on the classpath, and it can be completely controlled through `application.properties`, including all the granular fetaures on the `DataSource` that you need in production (like pool sizes and validation queries). So all you need is a way to discover the location and credentials for the service from the environment. Spring Boot also provides a `VcapApplicationListener` which translates Cloud Foundry `VCAP_*` environment variables into usable property sources in the Spring `Environment`. Thus, for instance, a `DataSource` configuration might look like this:

```properties
spring.datasource.url: ${vcap.services.mysql.credentials.uri:jdbc:h2:mem:testdb}
spring.datasource.username: ${vcap.services.mysql.credentials.username:sa}
spring.datasource.password: ${vcap.services.mysql.credentials.password:}
spring.datasource.testOnBorrow: true
```

The "mysql" part of the property names is the service name in Cloud Foundry (so it is set by the user). And of course the same pattern applies to all kinds of services, not just a JDBC `DataSource`. The only limitation of this approach is if the application needs to configure beans that are not provided by Spring Boot out of the box (e.g. if you need 2 `DataSources`), in which case you have to write Java code anyway, and may or may not choose to use properties files to parameterize it. Generally speaking it is good practice to use external configuration and in particular `@ConfigurationPropertues` since they allow maximum flexibility, for instance to override using System properties or environment variables at runtime.

> Note: similar features are provided by the Cloud Foundry buildpack, which provides `cloud.services.*` instead of `vcap.services.*`, so you actually end up with more than one way to do this. 

## Mixed Declarative and Explicit Bean Definition

You can also mix the two approaches: declare a single `@Bean` definition so that you control the construction of the object, but bind additional configuration to it using `@ConfigurationProperties` (and do the same locally and in Cloud Foundry). Example:

```java
@Configuration
public class LocalDataSourceConfiguration {
	
	@Bean
    @ConfigurationProperties(DataSourceProperties.PREFIX)
	public DataSource dataSource() {
		return DataSourceBuilder.create().build();
	}

}
```

(where the `DataSourceBuilder` would be replaced with whatever fancy logic you need for your use case). And the `application.properties` would be the same as above, with whatever additional properties you need for your production settings.

## A Third Way: Discover the Credentials and Bind Manually

Another approach that lends itself to platform and environment independence is to declare explicit bean definitions for the `@ConfigurationProperties` beans that Spring Boot uses to bind its autoconfigured connectors. For instance, to set the default values for a `DataSource` you can declare a `@Bean` of type `DataSourceProperties`:

```java
@Bean
@Primary
public DataSourceProperties dataSourceProperties() {
    DataSourceProperties properties = new DataSourceProperties();
    properties.setInitialize(false);
    return properties;
}
```

This sets a default value for the "initialize" flag, and allows other properties to be bound from `application.properties` (or other external properties). Combine this with the Spring Cloud Connectors and you can control the binding of the credentials when a cloud service is detected:

```java
@Bean
@Primary
public DataSourceProperties dataSourceProperties() {
    DataSourceProperties properties = new DataSourceProperties();
    properties.setInitialize(false);
    try {
      CloudFactory factory = new CloudFactory();
      List<ServiceInfo> infos = factory.getCloud().getServiceInfos(RelationalServiceInfo.class);
      if (infos.size()==1) {
        RelationalServiceInfo info = (RelationalServiceInfo) infos.get(0);
        properties.setUrl(info.getJdbcUrl());
        properties.setUsername(info.getUserName());
        properties.setPassword(info.getPassword());
      }
    } catch (CloudException e) { // ignore
    }
    return properties;
}
```

It ends up being quite a lot of code, and is quite unnecessary in this simple use case, but might be handy if you have more complicated bindings, or need to implement some logic to choose a `DataSource` at runtime.

Spring Boot has similar `*Properties` beans for the other middleware you might commonly use (e.g. `RabbitProperties`, `RedisProperties`, `MongoProperties`). An instance of such a bean marked as `@Primary` is enough to reset the defaults for the autoconfigured connector.

## Summary of Autoconfiguration and Provided Behaviour

* Spring Boot provides `DataSource` (also RabbitMQ or Redis `ConnectionFactory`, Mongo etc.) if it finds all the right stuff on the classpath. Using the "spring-boot-starter-*" dependencies is sufficient to activate the behaviour.

* Spring Boot also provides an autowirable `CloudFactory` if it finds Spring Cloud Connectors on the classpath (but switches off only if it finds a `@Bean` of type `Cloud`).

* The Cloud Foundry Java buildpack detects a Spring Boot application and activates the "cloud" profile, unless it is already active. Adding "org.cloudfoundry:auto-reconfiguration" (current release 1.7.0.RELEASE) does the same thing if you want to try it locally.

* Through the auto-reconfiguration JAR, the buildpack also kicks in and creates a `DataSource` (ditto RabbitMQ, Redis, Mongo etc.) if it does *not* find a `CloudFactory` bean or a `Cloud` bean (amongst others). So including Spring Cloud Connectors in a Spring Boot application switches off this part of the "auto-reconfiguration" behaviour (the bean creation).

* Spring Boot binds `application.properties` (and other sources of external properties) to `@ConfigurationProperties` beans, including but not limited to the ones that it autoconfigures. You can use this feature to tweak pool properties and other settings that need to be different in production environments.

## General Advice and Conclusion

We have seen quite a few options and autoconfigurations in this short article, and we've only really used thee libraries (Spring Boot, Spring Cloud Connectors, and the Cloud Foundry buildpack auto-reconfiguration JAR) and one platform (Cloud Foundry), not counting local deployment. The buildpack features are really only useful for very simple applications because there is no flexibility to tune the connections in production. That said it is a nice thing to be able to do when prototyping. There are only three main approaches if you want to achieve the goal of deploying the same code locally and in the cloud, yet still being able to make necessary tweaks in production: 

1. Use Spring Cloud Connectors to explicitly create `DataSource` and other middleware connections and protect those `@Beans` with `@Profile("cloud")`. The approach always works, but leads to more code than you might need for many applications.

2. Use the Spring Boot default autoconfiguration and declare the cloud bindings using `application.properties` (or in YAML).

3. Use Spring Cloud Connectors to discover the credentials, and bind them to the Spring Boot `@ConfigurationProperties` as default values if present.

The two approaches are actually not incompatible, and can be mixed using `@ConfigurationProperties` to provide profile-specific overrides of default configuration (e.g. for setting up connection pools in a different way in a production environment). If you have a relatively simple Spring Boot application, the only way to choose between the two approaches is probably personal taste. If you have a non-Spring Boot application then the explicit `@Bean` approach will win, and it will also win if you plan to deploy your application in more than one cloud platform (e.g. Heroku and Cloud Foundry).
