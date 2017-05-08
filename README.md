# weblogic-with-arquillian-cube

## The Sample Application
This project presented here builds upon Steve Button's [weblogic-with-arquillian](https://github.com/buttso/weblogic-with-arquillian) project by modifying it to use [Arquillian-Cube](http://arquillian.org/arquillian-cube/). I will discuss the steps that were necessary to migrate from a test suite running against a local WebLogic installation to a test suite running against a Dockerized WebLogic instance. 

## Adding the Maven Dependencies
Importing the bill of material (BOM) dependencies in the dependencyManagement section of the pom ensures that consistent versions of the Arquillian and Arquillian-Cube and all transitive dependencies are used:
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.arquillian</groupId>
            <artifactId>arquillian-universe</artifactId>
            <version>${version.arquillian_universe}</version>
            <scope>import</scope>
            <type>pom</type>
        </dependency>
    </dependencies>
</dependencyManagement>
```

A dependency for arquillian-cube-docker needs to be added along with arquillian-junit since the tests are written for JUnit using the Arquillian test-runner:
```xml
<dependency>
    <groupId>org.arquillian.universe</groupId>
    <artifactId>arquillian-cube-docker</artifactId>
    <scope>test</scope>
    <type>pom</type>
</dependency>

<dependency>
    <groupId>org.arquillian.universe</groupId>
    <artifactId>arquillian-junit</artifactId>
    <scope>test</scope>
    <type>pom</type>
</dependency>
```

Finally, a *weblogic* profile is added to the project along with a dependency to arquillian-wls-remote-10.3.

Using Maven profiles allows you to define several containers and switch between them using a command-line switch. This allows you to test your application against different application servers or environments very easily.

The 10.3 container was chosen because it uses the WebLogic Deployer to deploy test applications, the 12.x containers use JMX style deployment that requires the application file to be accessible from the WebLogic process. Because the Docker container is truly a remote server, the test files produced by the Arquillian framework were not visible to the application server process.

```xml
<profiles>
     <profile>
         <id>weblogic</id>
         <activation>
             <activeByDefault>true</activeByDefault>
         </activation>
         <properties>
             <arquillian.launch>weblogic</arquillian.launch>
         </properties>
         <dependencies>
             <groupId>org.jboss.arquillian.container</groupId>
             <artifactId>arquillian-wls-remote-rest</artifactId>
             <version>${version.arquillian-wls-remote-rest}</version>
         </dependencies>
     </profile>
</profiles>     
```
## Configuring Arquillian
*arquillian.xml* contains two extensions and a container definition. The first is the *cube* extension, which allows you to specify configuration options for Arquillian-Cube. Here I am only specifying one such option, the *connectionMode*. In this case, this setting is configuring Arquillian-Cube to use an existing container if one is already running, and to leave the container running after the test completes. This is a useful feature if you're running tests on your local machine; it's faster on subsequent invocations and you can access the running container to read logs or even attach a debugger.
```xml
<extension qualifier="cube">
    <property name="connectionMode">STARTORCONNECTANDLEAVE</property>
</extension>
```  
The *docker* extension is used to configure options specific to Docker. *dockerContainersFile* is a reference to a *docker-compose* compatible file. The *docker-compose* tool is not used to create the services, rather the file is parsed and services are created using the [docker-java](https://github.com/docker-java/docker-java) library.

*serverUri* specifies the method by which Arquillian-Cube will communicate with Docker. If you do not specify this property it is inferred based on your platform. Setting this explicitly on the Mac overrode the default behavior of using boot2docker and worked with docker-engine directly.  

```xml
    <extension qualifier="docker">
        <property name="autoStartContainers">true</property>
        <property name="dockerContainersFile">docker-compose.yml</property>
        <property name="serverVersion">1.12</property>
        <property name="serverUri">unix:///var/run/docker.sock</property>
        <property name="tlsVerify">false</property>
    </extension>
```

Finally, the *weblogic* container definition which is used by the WebLogic Remote Arquillian Container specifies the configuration needed to connect to and deploy applications to the remote server. This container definition is the same as if WebLogic were running directly on the host:
```xml
<container qualifier="weblogic" default="true">
    <configuration>
        <property name="adminUrl">http://localhost:7001</property>
        <property name="adminUserName">weblogic</property>
        <property name="adminPassword">welcome1</property>
        <property name="target">AdminServer</property>
    </configuration>
</container>
```

*wlHome* refers to an installation of WebLogic on the Docker host. *jmxClientJarPath* also refers to a location on the Docker host and is used to construct a classpath for the Deployer. *wlthint3client.jar* was chosen because [wljmxclient.jar](https://docs.oracle.com/middleware/12212/wls/SACLT/basics.htm#SACLT123) does not provide support for the t3 protocol.  

## Docker configuration
The last change needed is the addition of the *docker-compose.yml* file referenced by *arquillian.xml*. I used an image built from the [Example of Image with WLS Domain](https://github.com/oracle/docker-images/tree/master/OracleWebLogic/samples/1221-domain) sample. If you needed to you could take advantage of Docker's extension mechanism to add configuration (such as a JMS Server or Datasource) as in the [1221-domain-with-resources](https://github.com/oracle/docker-images/tree/master/OracleWebLogic/samples/1221-domain-with-resources) sample image.

```yml
weblogic:
  image: 1221-domain:latest
  ports:
      - "7001:7001"
  environment:
      - PRODUCTION_MODE=dev
```
## Running the Test Applications
You can run the test application from an IDE or from the command-line by invoking Maven:

  ```
  $ mvn test
[INFO] Scanning for projects...
[INFO]                                                                         
[INFO] ------------------------------------------------------------------------
[INFO] Building wlsarq 1.0
[INFO] ------------------------------------------------------------------------

-------------------------------------------------------
 T E S T S
-------------------------------------------------------
Running buttso.demo.weblogic.wlsarq.IndexPageTest
CubeDockerConfiguration:
  serverVersion = 1.12
  serverUri = unix:///var/run/docker.sock
  tlsVerify = false
  dockerServerIp = localhost
  definitionFormat = COMPOSE
  autoStartContainers =
  clean = false
  removeVolumes = true
  dockerContainers = containers:
  weblogic:
    alwaysPull: false
    env: !!set {PRODUCTION_MODE=dev: null}
    image: 1221-domain:latest
    killContainer: false
    manual: false
    portBindings: !!set {7001/tcp: null}
    readonlyRootfs: false
    removeVolumes: true
networks: {}


Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 10.19 sec - in buttso.demo.weblogic.wlsarq.IndexPageTest
Running buttso.demo.weblogic.wlsarq.PingPongBeanTest
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 7.36 sec - in buttso.demo.weblogic.wlsarq.PingPongBeanTest

Results :

Tests run: 3, Failures: 0, Errors: 0, Skipped: 0

[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 20.222 s
[INFO] Finished at: 2017-04-12T17:04:52-04:00
[INFO] Final Memory: 20M/437M
[INFO] ------------------------------------------------------------------------

  ```
