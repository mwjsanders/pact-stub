# Service stub met behulp van Pact

Voor het lokaal testen van je applicatie kan het soms handig zijn om externe services waar je applicatie gebruik van maakt te mocken of te stubben. Op deze manier kan een applicatie gei# Service stub met behulp van Pact

Voor het lokaal testen van je applicatie kan het soms handig zijn om externe services waar je applicatie gebruik van maakt te mocken of te stubben. Op deze manier kan een applicatie geïsoleerd getest worden zonder dat dit invloed heeft op de toestand van eventuele afhankelijke test of productie applicaties. Ook is het dan niet nodig om lokaal een instantie van deze externe services te deployen. [Pact.io](http://pact.io) biedt een mogelijkheid om snel en eenvoudig een server stub op te zetten.

[Pact.io](http://pact.io) is een test tool voor het testen van het contract tussen een consumer en een provider van een service. Voor verdere details hiervoor wil ik je verwijzen naar [https://docs.pact.io/](). In deze blog ga ik slechts in op een klein deel van Pact: De server stub. En dan met name hoe je deze stub moet opzetten en hoe je deze stub kan inzetten bij integratie tests.

## Een service stub maken

Pact biedt een docker container die aan de hand van vooraf gedefinieerde interacties een service kan starten en deze interacties kan naspelen. Een interactie is een combinatie van een request en een response en deze worden in een *pact bestand* opgeslagen. Ieder request dat deze service ontvangt wordt geprobeerd te matchen tegen een request in het pact bestand. Indien er een match is wordt de bijbehorende response geretourneerd.

Hieronder een voorbeeld van een *pact bestand* waarin twee interacties met de volgende request paden zijn gedefieerd:
- **/health**: een health check
- **/products**: een rest endpoint voor het ophalen van een lijst producten

```json
{
  "consumer": {
    "name": "some-consumer"
  },
  "provider": {
    "name": "some-provider"
  },
  "interactions": [
    {
      "description": "health check",
      "provider_state": "Some state",
      "request": {
        "method": "GET",
        "path": "/health"
      },
      "response": {
        "status": 200,
        "headers": {
          "Content-Type": "text/plain"
        },
        "body": "OK"
      }
    },
    {
      "description": "find all rpoducts",
      "provider_state": "Some state",
      "request": {
        "method": "GET",
        "path": "/products"
      },
      "response": {
        "status": 200,
        "headers": {
          "Content-Type": "application/json"
        },
        "body":
        {
          "Products": {
            "Product": [
              {
                "CatalogueID": "101",
                "Name": "Widget",
                "Price": "10.99",
                "Manufacturer": "Company A",
                "InStock": "Yes"
              },
              {
                "CatalogueID": "300",
                "Name": "Fooble",
                "Price": "2.00",
                "Manufacturer": "Company B",
                "InStock": "No"
              }
            ]
          }
		}
      }
    }
  ]
}
```
De Service stub heeft alleen dit bestand nodig een service te starten. De volgende Dockerfile gebruikt het *pact bestand* en kopieert dit bestand genaamd *pact.json* in de docker container.
```Dockerfile
FROM pactfoundation/pact-stub-server:latest
COPY pact.json  /app/pacts/
ENTRYPOINT ["/app/pact-stub-server", "-d=/app/pacts", "-p=8080"]
EXPOSE 8080
```
Nu hoeven we alleen nog onze eigen variant van de docker container te bouwen en deze te starten:
```cmd
$ docker build -t provider-stub .
$ docker run -p 8080:8080 provider-stub
```

Zodra de container gestart is kan je controleren of de stub het doet door in een browser de url `http://localhost:8080/products` te benaderen. Je moet dan een lijstje van producten terug krijgen dat in de pact file is gedefinieerd.

##  Intergratie test met deze stub

Hoe je deze server stub kan inzetten voor het uitvoeren van integratie tests van de consumerende applicatie wil ik graag demonstreren met een aantal code snippets uit een werkende applicatie. Deze applicatie noemen we voor het gemak de consumer en is een Spring Boot applicatie geschreven in Kotlin die middels Maven gebouwd wordt. De consmer is diegene die de provider gaat bevragen.
De [fabic8 maven plugin](https://maven.fabric8.io/) maakt het mogelijk om vooraf aan het uitvoeren van de integratie test van de consumer een docker container tijdelijk te starten. In dit voorbeeld bepaalt docker zelf op welke poort de provider stub beschikbaar wordt, in tegenstelling tot het eerder genoemde docker run commando waar de interne port gemapt wordt naar port 8080.

De docker-maven-plugin voert volgens onderstaande setup in de pom.xml de volgende stappen uit:
- De docker container wordt gebouwd voor de stub volgens het recept in de opgegeven Dockerfile. De eerder genoemde Dockerfile kan hiervoor gebruikt worden.
- De gebouwde container wordt gestart. De poort waarop de stub te benaderen is wordt toegekend aan de property *provider-stub.port*. De maven failsafe plugin maakt deze properties beschikbaar als environemt variabele zodat deze later in de consumer gebruikt kunne worden om de client te configureren.
- Er wordt gewacht tot de health check van de stub een status code 200 retourneert.
Na het uitvoeren van de integratie test wordt de container van de stub weer gestopt.

```xml
<project>
  ...
  <properties>
    <docker.fabric8.docker-maven-plugin>0.26.0</docker.fabric8.docker-maven-plugin>
    <dns.host.docker>localhost</dns.host.docker>
    <!-- Default port waarop stub beschikbaar is-->
    <provider-stub.port>8081</provider-stub.port>
  </properties>
  ...
  <build>
    <plugins>
      ...
            <!-- integration testing-->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-failsafe-plugin</artifactId>
                <executions>
                    <execution>
                        <id>failsafe-integration-test</id>
                        <phase>integration-test</phase>
                        <goals>
                            <goal>integration-test</goal>
                            <goal>verify</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                	<!-- Maak de volgende environment variabele beschikbaar voor de applicatie-->
                    <environmentVariables>
                        <catalog-service.port>${provider-stub.port}</catalog-service.port>
                        <dns.host.docker>${dns.host.docker}</dns.host.docker>
                    </environmentVariables>
                </configuration>
            </plugin>
            <plugin>
                <groupId>io.fabric8</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>${docker.fabric8.docker-maven-plugin}</version>
                <executions>
                    <execution>
                        <id>build</id>
                        <phase>pre-integration-test</phase>
                        <goals>
                            <goal>build</goal>
                            <goal>start</goal>
                        </goals>
                        <configuration>
                            <images>
                                <image>
                                    <name>provider-stub:0.0.1-it</name>
                                    <alias>provider-stub</alias>
                                    <build>
                                        <!-- verwijzing naar de Dockerfile van de pact stub -->
                                        <dockerFileDir>${project.basedir}/stub/</dockerFileDir>
                                        <tags>
                                            <tag>0.0.1-it</tag>
                                        </tags>
                                    </build>
                                    <run>
                                        <ports>
                                            <port>provider-stub.port:8080</port>
                                        </ports>
                                        <wait>
                                            <!-- preconditie voordat de integratie test uitgevoerd kan worden -->
                                            <http>
                                                <url>http://${dns.host.docker}:${provider-stub.port}/health</url>
                                                <method>GET</method>
                                                <status>200</status>
                                            </http>
                                            <time>5000</time>
                                        </wait>
                                    </run>
                                </image>
                            </images>
                        </configuration>
                    </execution>
                    <execution>
                        <id>remove-provider-stub</id>
                        <phase>post-integration-test</phase>
                        <goals>
                            <goal>stop</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
    </plugins>
  </build>
</project>
```

De consumer heeft een client die kan communiceren met de provider. De client class zal een dergelijke opzet hebben. Het endpoint van de provider (hieronder terug te vinden als variabele *providerUrl*) wordt middels een property geset.

```java
@Service
class ProviderClient : IProvider {

    @Value("\${provider.url}")
    private lateinit var providerUrl: String

    override fun getProducts(): Set<Products> {
    	...
    }
}
```
De integratie test maakt gebruik van deze client om producten op te halen en hier eventueel in de test iets mee te doen. Doordat de class naam eindigt op IT zal deze herkent worden als integratie test en zal de maven plugin de stub server starten voor het uitvoeren van deze test. Tevens wordt een profiel *integratie-test* geladen voor de juiste configuratie van de client zodat deze met de stub gaat communiceren.
```java
@SpringBootTest
@ActiveProfiles("integration-test")
class ProviderClientIT(){

	@Autowired
    private lateinit var providerClient: IProvider

    @Test
    fun getCapabilitiesTest() {
		val products =  providerClient.getProducts()
        ...
        ... // a lot of assertions
    }
}
```
De Spring Boot configuratie bevat het profiel *integration-test* dat in de bovenstaande integratie test gebruikt wordt. Hier zie hoe de *provider.url* property geset wordt. Deze wordt dus gevuld met de twee variabelen die vanuit de maven plugin geset worden.
```yaml
spring:
  profiles: default
provider:
    url: http://localhost:8081/
---
spring:
  profiles: prod
provider:
    url: http://www.some-provider.com/
---
spring:
  profiles: integration-test
provider:
    url: http://${dns.host.docker}:${provider-stub.port}/
---
```
## Conclusie
Pact biedt een eenvoudige manier om service's stubben. Dit kan handig zijn voor het testen van lokaal gedeployde applicaties die afhankelijkheden hebben met externe services waar mogelijk geen toegang toe is. Het mooie is dat diezelfde stub voor het lokaal testen gebruikt kan worden voor de integratie.soleerd getest worden zonder dat dit invloed heeft op de toestand van eventuele afhankelijke test of productie applicaties. Ook is het dan niet nodig om lokaal een instantie van deze externe services te deployen. [Pact.io](http://pact.io) biedt een mogelijkheid om snel en eenvoudig een server stub op te zetten.

[Pact.io](http://pact.io) is een test tool voor het testen van het contract tussen een consumer en een provider van een service. Voor verdere details hiervoor wil ik je verwijzen naar [https://docs.pact.io/](). In deze blog ga ik slechts in op een klein deel van Pact: De server stub. En dan met name hoe je deze stub moet opzetten en hoe je deze stub kan inzetten bij integratie tests.

## Een service stub maken

Pact biedt een docker container die aan de hand van vooraf gedefineerde interacties een service kan starten en deze interacties kan naspelen. Een interactie is een combinatie van een request en een response en deze worden in een *pact bestand* opgeslagen. Ieder request dat deze service ontvangt wordt geprobeerd te matchen tegen een request in het pact bestand. Indien er een match is wordt de bijbehorende response geretourneerd.

Hieronder een voorbeeld van een *pact bestand* waarin twee interacties met de volgende request paden zijn gedefieerd:
- **/health**: een health check
- **/products**: een rest endpoint voor het ophalen van een lijst producten

```json
{
  "consumer": {
    "name": "some-consumer"
  },
  "provider": {
    "name": "some-provider"
  },
  "interactions": [
    {
      "description": "health check",
      "provider_state": "Some state",
      "request": {
        "method": "GET",
        "path": "/health"
      },
      "response": {
        "status": 200,
        "headers": {
          "Content-Type": "text/plain"
        },
        "body": "OK"
      }
    },
    {
      "description": "find all rpoducts",
      "provider_state": "Some state",
      "request": {
        "method": "GET",
        "path": "/products"
      },
      "response": {
        "status": 200,
        "headers": {
          "Content-Type": "application/json"
        },
        "body":
        {
          "Products": {
            "Product": [
              {
                "CatalogueID": "101",
                "Name": "Widget",
                "Price": "10.99",
                "Manufacturer": "Company A",
                "InStock": "Yes"
              },
              {
                "CatalogueID": "300",
                "Name": "Fooble",
                "Price": "2.00",
                "Manufacturer": "Company B",
                "InStock": "No"
              }
            ]
          }
		}
      }
    }
  ]
}
```
De Service stub heeft alleen dit bestand nodig een service te starten. Met de volgende Dockerfile gebruiken we het *pact bestand* en kopieren dit bestand genaamd *pact.json* in de docker container.
```Dockerfile
FROM pactfoundation/pact-stub-server:latest
COPY pact.json  /app/pacts/
ENTRYPOINT ["/app/pact-stub-server", "-d=/app/pacts", "-p=8080"]
EXPOSE 8080
```
Nu hoeven we alleen nog onze eigen variant van de docker container te bouwen en deze te starten:
```cmd
$ docker build -t provider-stub .
$ docker run -p 8080:8080 provider-stub
```

Zodra de container gestart is kan je controleren of de stub het doet door in een browser de url `http://localhost:8080/products` te benaderen. Je moet dan een lijstje van producten terug krijgen dat in de pact file is gedefineerd.

##  Intergratie test met deze stub

Hoe je deze server stub kan inzetten voor het uitvoeren van integratie tests van de consumerende applicatie wil ik graag demonstreren met een aantal code snippets uit een werkende applicatie. Deze applicatie noemen we voor het gemak de consumer en is een Spring Boot applicatie geschreven in Kotlin die middels Maven gebouwd wordt. De consmer is diegene die de provider gaat bevragen.
De [fabic8 maven plugin](https://maven.fabric8.io/) maakt het mogelijk om vooraf aan het uitvoeren van de integratie test van de consumer een docker container tijdelijk te starten. In dit voorbeeld bepaalt docker zelf op welke poort de provider stub beschikbaar wordt, in tegenstelling tot het eerder genoemde docker run commando waar de interne port gemapt wordt naar port 8080.

De docker-maven-plugin voert volgens onderstaande setup in de pom.xml de volgende stappen uit:
- De docker container wordt gebouwd voor de stub volgens het recept in de opgegeven Dockerfile. De eerder genoemde Dockerfile kan hiervoor gebruikt worden.
- De gebouwde container wordt gestart. De poort waarop de stub te benaderen is wordt toegekend aan de property *provider-stub.port*. De maven failsafe plugin maakt deze properties beschikbaar als environemt variabele zodat deze later in de consumer gebruikt kunne worden om de client te configureren.
- Er wordt gewacht tot de health check van de stub een status code 200 retourneerd.
Na het uitvoeren van de integratie test wordt de container van de stub weer gestopt.

```xml
<project>
  ...
  <properties>
    <docker.fabric8.docker-maven-plugin>0.26.0</docker.fabric8.docker-maven-plugin>
    <dns.host.docker>localhost</dns.host.docker>
    <!-- Default port waarop stub beschikbaar is-->
    <provider-stub.port>8081</provider-stub.port>
  </properties>
  ...
  <build>
    <plugins>
      ...
            <!-- integration testing-->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-failsafe-plugin</artifactId>
                <executions>
                    <execution>
                        <id>failsafe-integration-test</id>
                        <phase>integration-test</phase>
                        <goals>
                            <goal>integration-test</goal>
                            <goal>verify</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                	<!-- Maak de volgende environment variabele beschikbaar voor de applicatie-->
                    <environmentVariables>
                        <catalog-service.port>${provider-stub.port}</catalog-service.port>
                        <dns.host.docker>${dns.host.docker}</dns.host.docker>
                    </environmentVariables>
                </configuration>
            </plugin>
            <plugin>
                <groupId>io.fabric8</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>${docker.fabric8.docker-maven-plugin}</version>
                <executions>
                    <execution>
                        <id>build</id>
                        <phase>pre-integration-test</phase>
                        <goals>
                            <goal>build</goal>
                            <goal>start</goal>
                        </goals>
                        <configuration>
                            <images>
                                <image>
                                    <name>provider-stub:0.0.1-it</name>
                                    <alias>provider-stub</alias>
                                    <build>
                                        <!-- verwijzing naar de Dockerfile van de pact stub -->
                                        <dockerFileDir>${project.basedir}/stub/</dockerFileDir>
                                        <tags>
                                            <tag>0.0.1-it</tag>
                                        </tags>
                                    </build>
                                    <run>
                                        <ports>
                                            <port>provider-stub.port:8080</port>
                                        </ports>
                                        <wait>
                                            <!-- preconditie voordat de integratie test uitgevoerd kan worden -->
                                            <http>
                                                <url>http://${dns.host.docker}:${provider-stub.port}/health</url>
                                                <method>GET</method>
                                                <status>200</status>
                                            </http>
                                            <time>5000</time>
                                        </wait>
                                    </run>
                                </image>
                            </images>
                        </configuration>
                    </execution>
                    <execution>
                        <id>remove-provider-stub</id>
                        <phase>post-integration-test</phase>
                        <goals>
                            <goal>stop</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
    </plugins>
  </build>
</project>
```

De consumer heeft een client die kan communiceren met de provider. De client class zal een dergelijke opzet hebben. Het endpoint van de provider (hieronder terug te vinden als variabele *providerUrl*) wordt middels een property geset.

```java
@Service
class ProviderClient : IProvider {

    @Value("\${provider.url}")
    private lateinit var providerUrl: String

    override fun getProducts(): Set<Products> {
    	...
    }
}
```
De integratie test maakt gebruik van deze client om producten op te halen en hier eventueel in de test iets mee te doen. Doordat de class naam eindigt op IT zal deze herkent worden als integratie test en zal de maven plugin de stub server starten voor het uitvoeren van deze test. Tevens wordt een profiel *integratie-test* geladen voor de juiste configuratie van de client zodat deze met de stub gaat communiceren.
```java
@SpringBootTest
@ActiveProfiles("integration-test")
class ProviderClientIT(){

	@Autowired
    private lateinit var providerClient: IProvider

    @Test
    fun getCapabilitiesTest() {
		val products =  providerClient.getProducts()
        ...
        ... // a lot of assertions
    }
}
```
De Spring Boot configuratie bevat het profiel *integration-test* dat in de bovenstaande integratie test gebruikt wordt. Hier zie hoe de *provider.url* property geset wordt. Deze wordt dus gevuld met de twee variabelen die vanuit de maven plugin geset worden.
```yaml
spring:
  profiles: default
provider:
    url: http://localhost:8081/
---
spring:
  profiles: prod
provider:
    url: http://www.some-provider.com/
---
spring:
  profiles: integration-test
provider:
    url: http://${dns.host.docker}:${provider-stub.port}/
---
```
## Conclusie
Pact biedt een eenvoudige manier om service's stubben. Dit kan handig zijn voor het testen van lokaal gedeployde applicaties die afhankelijkheden hebben met externe services waar mogelijk geen toegang toe is. Het mooie is dat diezelfde stub voor het lokaal testen gebruikt kan worden voor de integratie.

