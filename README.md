# TP-BPEL — Student enrollment: Orchestration & Choreography
## Name: KOWO MANFOUO ELIE MAKODA
## Class: INGE 4 ISI EN 
## School: SJI

Tools: JDK 11, Maven 3.9.x, Tomcat 9.0.x (with `ode.war` 1.3.8 in `webapps/`), ActiveMQ Artemis 2.33.x, SoapUI 5.7.x.
`$TOMCAT_HOME` points to the Tomcat folder.
## Starting the app 
* 1) Clone the repository
`git clone  https://github.com/eliemakoda/Tp-Bel-KOWOManfouo_ElieMakoda.git
`
* 2) unzip 
## Build

```bash
cd tp-bpel-enrollment
mvn clean install
```

## Part A — Orchestration

Deploy the four partner services:

```bash
cp services/*/target/*.war $TOMCAT_HOME/webapps/
```

Deploy the BPEL process on ODE:

```bash
cp -r bpel-orchestrator $TOMCAT_HOME/webapps/ode/WEB-INF/processes/enrollment
cp services/*/src/main/resources/*.wsdl $TOMCAT_HOME/webapps/ode/WEB-INF/processes/enrollment/
cp -r --parents services/*/src/main/resources/*.wsdl $TOMCAT_HOME/webapps/ode/WEB-INF/processes/
```

Start Tomcat:

```bash
$TOMCAT_HOME/bin/catalina.sh run
```

Check:

- http://localhost:8080/university-service/UniversityService?wsdl
- http://localhost:8080/bank-service/BankService?wsdl
- http://localhost:8080/residence-service/ResidenceService?wsdl
- http://localhost:8080/transport-service/TransportService?wsdl
- http://localhost:8080/ode/processes/EnrollmentService?wsdl

Test: open `enrollment-tests.xml` in SoapUI and run `EnrollmentTestSuite`.

## Part B — Choreography

Start the broker and create the topics:

```bash
~/artemis-broker/bin/artemis-service start
~/artemis-broker/bin/artemis address create --name EnrollmentRequested --multicast
~/artemis-broker/bin/artemis address create --name StudentRegistered   --multicast
~/artemis-broker/bin/artemis address create --name PaymentDecided      --multicast
~/artemis-broker/bin/artemis address create --name EnrollmentCompleted --multicast
```

Start the four adapters, each one in its own terminal:

```bash
cd choreography/university-event-adapter && mvn spring-boot:run
cd choreography/bank-event-adapter       && mvn spring-boot:run
cd choreography/residence-event-adapter  && mvn spring-boot:run
cd choreography/transport-event-adapter  && mvn spring-boot:run
```

Trigger the flow by publishing an `EnrollmentRequested` event:

```bash
curl -u admin:admin -H "Content-Type: application/json" -H "Origin: http://localhost" \
     -XPOST http://localhost:8161/console/jolokia \
     -d '{"type":"exec",
          "mbean":"org.apache.activemq.artemis:broker=\"0.0.0.0\",component=addresses,address=\"EnrollmentRequested\"",
          "operation":"sendMessage(java.util.Map,int,java.lang.String,boolean,java.lang.String,java.lang.String)",
          "arguments":[{},3,"{\"correlationId\":\"enr-001\",\"type\":\"EnrollmentRequested\",\"data\":{\"student\":{\"name\":\"Alice\",\"program\":\"ISI4\"},\"amount\":\"350000\"}}",false,"admin","admin"]}'
```

Use `"amount":"900000"` for the refused flow.

Observe the messages on each topic in the Artemis console: http://localhost:8161 (admin / admin).
