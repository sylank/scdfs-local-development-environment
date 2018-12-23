# Local development environment for Spring Dataflow Server

## Environment setup
This repo contains a docker based local development environment for Spring Cloud Dataflow Server. It conains the major dependencies:
* Kafka[https://kafka.apache.org/] message broker server
* Zookeeper[https://zookeeper.apache.org/] coordination service for Kafka
* Redis[https://redis.io/] cache databas to save stream data
* Skipper[https://cloud.spring.io/spring-cloud-skipper/] server to run and manage stream applications

Requirements:
* Docker
* Docker-Compose
* Tip: Redis Desktop Manager
* Tip: IntelliJ IDEA CE

It is easy to use:
```shell
.\compose.sh
```

**Note:** you will not reach the Kafka, Zookeeper, Redis service because we do not bint the ports to host due to `expose` keyword in compose file. That services is reachable inside the docker network. If you want to reach them outside use `ports` keyword but check use different ports.

## Container check
We need to check the proper working of the docker. Some linux distribution can denie some service. For example the **apparmor** can denie the port binding. Sometimes the docker can not resolve DNS names. 

Use jimho's tcp-echo server to check your containers can bind port on your host OS. If you can not see BindingExceptions or Permission denied exceptions your container have access to the host OS ports.

To avoid binding exceptions:
```
docker run jimho/tcp-echo
```

After start you will have to download maven resources so you need 'internet' access. Check your connection with byrnedo's curl image.

To check internet access:
```
docker run byrnedo/alpine-curl https://www.google.com
```

## Dataflow Server setup
In this step we will clone and build the Spring Cloud Dataflow Server. Please find the link[https://cloud.spring.io/spring-cloud-dataflow/] to read something about the project.

Clone the repo:
```
git clone https://github.com/spring-cloud/spring-cloud-dataflow.git
```

Step inside the directory and build it with maven wrapper:
```
./mvnw clean install or ./mvnw clean install -DskipTests 
```

When the build finish successfully, jar files will be generated, so you can run the modules with java commands. We will use the `spring-cloud-dataflow-shell` to manage the server (do not affraid you can use TAB to auto-complete commands).

If the build ended you have to setup some environment variables for the stream applications and for the server too:
```
spring.cloud.dataflow.applicationProperties.stream.spring.cloud.stream.kafka.binder.brokers=kafka:9092;
spring.cloud.dataflow.applicationProperties.stream.spring.cloud.stream.kafka.binder.zkNodes=zookeeper:2181;
spring.cloud.skipper.client.serverUri=http://127.0.0.1:7577/api
```
**Note:** the `kafka` and the `zookeeper` keywords are used in the docker network, docker will resolve them to real IP adresses, if you don't use docker environment change them to `localhost` or another address.

You can export it in yout current shell session or you set the variables at IntelliJ's Run/Debug Configurations

Now you can start the server:
```
java -jar spring-cloud-dataflow-server/target/spring-cloud-dataflow-server-<version>.jar
```

It is running on `http://localhost:9393` and you can reach the dashboard UI at `http://localhost:9393/dashboard/index.html`

If you have question, project Gitter channel: https://gitter.im/spring-cloud/spring-cloud-dataflow

### Install apps
You have to install (register) apps to create streams. This is just a setup documentation so we will not take deep dive. You can use maven local or remote urls and local files on your local file system.

Up to date application list: https://cloud.spring.io/spring-cloud-stream-app-starters/

#### Bulk import
You can import a bunch of applications with bulk application import.
```
dataflow:>app import --uri http://bit.ly/Darwin-SR3-stream-applications-kafka-maven
```

#### Maven
It is better to use this[https://docs.spring.io/spring-cloud-dataflow/docs/current/reference/htmlsingle/#spring-cloud-dataflow-register-stream-apps] guideline.

Shell command:
```
dataflow:>app register --name mysource --type source --uri maven://com.example:mysource:0.0.1-SNAPSHOT
```

Schema:
```
maven://<groupId>:<artifactId>[:<extension>[:<classifier>]]:<version>
```

#### Local
You can use your local file system but it is complecated when you use docker environment because of skipper and docker and host file system. It is better to read the app register documentation[https://docs.spring.io/spring-cloud-dataflow/docs/current/reference/htmlsingle/#spring-cloud-dataflow-register-stream-apps].

Some useful command:
```
dataflow:>app register --name myprocessor --type processor --uri file:///Users/example/myprocessor-1.2.3.jar
```

Or import if from a http url:
```
dataflow:>app register --name mysink --type sink --uri http://example.com/mysink-2.0.1.jar
```

### Test stream
Now we need to create a stream. Create one with the following commands:

Create a stream:
```
dataflow:>stream create --name test --definition "time|log"
```

Deploy the stream:
```
dataflow:>stream deploy --name test
```

Check the skipper logs and check the log files inside the docker container. 

Example Skipper logs:
```
kipper           | 2018-12-23 11:28:37.311  INFO 1 --- [eTaskExecutor-3] o.s.c.s.s.s.StateMachineConfiguration    : Entering state StateMachineState [getIds()=[INSTALL], toString()=AbstractState [id=INSTALL, pseudoState=null, deferred=[], entryActions=[], exitActions=[], stateActions=[], regions=[], submachine=INSTALL_INSTALL INSTALL_EXIT  /  / uuid=8895ae6d-d965-48eb-b586-6ca7c111ec16 / id=test], getClass()=class org.springframework.statemachine.state.StateMachineState]
skipper           | 2018-12-23 11:28:37.314  INFO 1 --- [eTaskExecutor-3] o.s.c.s.s.s.StateMachineConfiguration    : Entering state ObjectState [getIds()=[INSTALL_INSTALL], getClass()=class org.springframework.statemachine.state.ObjectState, hashCode()=1229772397, toString()=AbstractState [id=INSTALL_INSTALL, pseudoState=org.springframework.statemachine.state.DefaultPseudoState@15fae9c0, deferred=[], entryActions=[org.springframework.cloud.skipper.server.statemachine.InstallInstallAction@1aa6e3c0], exitActions=[], stateActions=[], regions=[], submachine=null]]
skipper           | 2018-12-23 11:28:37.741  INFO 1 --- [eTaskExecutor-3] o.s.c.d.spi.local.LocalAppDeployer       : Command to be executed: /opt/openjdk/bin/java -jar /root/.m2/repository/org/springframework/cloud/stream/app/time-source-kafka/2.0.1.RELEASE/time-source-kafka-2.0.1.RELEASE.jar
skipper           | 2018-12-23 11:28:37.746  INFO 1 --- [eTaskExecutor-3] o.s.c.d.spi.local.LocalAppDeployer       : Deploying app with deploymentId test.time-v1 instance 0.
skipper           |    Logs will be in /tmp/spring-cloud-deployer-6406189366663531439/test-1545564517607/test.time-v1
skipper           | 2018-12-23 11:28:37.849  INFO 1 --- [eTaskExecutor-3] o.s.c.d.spi.local.LocalAppDeployer       : Command to be executed: /opt/openjdk/bin/java -jar /root/.m2/repository/org/springframework/cloud/stream/app/log-sink-kafka/2.0.2.RELEASE/log-sink-kafka-2.0.2.RELEASE.jar
skipper           | 2018-12-23 11:28:37.850  INFO 1 --- [eTaskExecutor-3] o.s.c.d.spi.local.LocalAppDeployer       : Deploying app with deploymentId test.log-v1 instance 0.
skipper           |    Logs will be in /tmp/spring-cloud-deployer-6406189366663531439/test-1545564517747/test.log-v1
skipper           | 2018-12-23 11:28:37.920  INFO 1 --- [eTaskExecutor-3] o.s.s.support.LifecycleObjectSupport     : stopped org.springframework.statemachine.support.DefaultStateMachineExecutor@57081d5
skipper           | 2018-12-23 11:28:37.921  INFO 1 --- [eTaskExecutor-3] o.s.s.support.LifecycleObjectSupport     : stopped INSTALL_INSTALL INSTALL_EXIT  /  / uuid=8895ae6d-d965-48eb-b586-6ca7c111ec16 / id=test
skipper           | 2018-12-23 11:28:37.922  INFO 1 --- [eTaskExecutor-3] o.s.c.s.s.s.StateMachineConfiguration    : Entering state ObjectState [getIds()=[INITIAL], getClass()=class org.springframework.statemachine.state.ObjectState, hashCode()=745230356, toString()=AbstractState [id=INITIAL, pseudoState=org.springframework.statemachine.state.DefaultPseudoState@4255d564, deferred=[], entryActions=[], exitActions=[org.springframework.cloud.skipper.server.statemachine.ResetVariablesAction@70f822e], stateActions=[], regions=[], submachine=null]]
skipper           | 2018-12-23 11:28:37.922  INFO 1 --- [eTaskExecutor-3] o.s.c.s.s.s.SkipperStateMachineService   : setting future value org.springframework.cloud.skipper.domain.Release@51321fef
skipper           | 2018-12-23 11:28:37.924  INFO 1 --- [eTaskExecutor-3] o.s.s.support.LifecycleObjectSupport     : started org.springframework.statemachine.support.DefaultStateMachineExecutor@57081d5
skipper           | 2018-12-23 11:28:37.925  INFO 1 --- [eTaskExecutor-3] o.s.s.support.LifecycleObjectSupport     : started INSTALL_INSTALL INSTALL_EXIT  /  / uuid=8895ae6d-d965-48eb-b586-6ca7c111ec16 / id=test
```

This is the important line:
```
skipper           | 2018-12-23 11:28:37.746  INFO 1 --- [eTaskExecutor-3] o.s.c.d.spi.local.LocalAppDeployer       : Deploying app with deploymentId test.time-v1 instance 0.
skipper           |    Logs will be in /tmp/spring-cloud-deployer-6406189366663531439/test-1545564517607/test.time-v1

```
So we need to print out the ` /tmp/spring-cloud-deployer-6406189366663531439/test-1545564517607/test.time-v1` files content.

List the running containers and step inside:
```
$ docker ps

$ docker exec -it <skipper_container_id> /bin/bash
```
Then `cd` into log file directory. You have to print out the `stdout_0.log` content. Check the `source` and the `sink` too.

### Build your own app
Check the applications source to create your own application, you can check them here[https://github.com/spring-cloud-stream-app-starters]. You can generate your app with (Spring Initializr)[https://github.com/spring-cloud-stream-app-starters].

### Run and debug from IntelliJ
Import the project as maven application and you can start the pre-configured Dataflow Server from `spring-cloud-dataflow-server/src/main/java/org/springframework/cloud/dataflow/server/single/DataFlowServerApplication.java`. Don't forget to set the environment variables.
