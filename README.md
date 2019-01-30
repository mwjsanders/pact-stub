# Service stub met behulp van Pact

Voor het lokaal testen van je applicatie kan het soms handig zijn om externe services waar je applicatie gebruik van maakt te mocken of te stubben. Op deze manier kan een applicatie geisoleerd getest worden zonder dat dit invloed heeft op de staat van eventuele test of productie applicaties. Ook is het dan niet nodig om lokaal een instantie van deze externe deployen. [Pact.io](http://pact.io) biedt een mogelijkheid om snel en eenvoudig een server stub in de lucht te brengen.

[Pact.io](http://pact.io) is een test tool voor het testen van het contract tussen een consumer en een provider van een service. Voor verdere details hiervoor kan je terecht op [https://docs.pact.io/](). In deze blog ga ik slechts in op een klein deel van Pact: De server stub. En dan hoe je deze stub moet opzetten en hoe je deze stub kan inzetten bij integratie tests.

## Een stub maken

Pact biedt een docker container die aan de hand van een *pact bestand*; de interacties die vastgelegd zijn in dit bestand kan naspelen. Een interactie is een combinatie van een request en een response. Ieder request dat de pact server stub ontvangt wordt geprobeerd te matchen op een request in het pact bestand. Indien er een match is wordt de bijbehorende response geretourneerd.

Hieronder een voorbeeld van een *pact bestand* waarin twee endpoint zijn gedefieerd:
- **/health** : een health check
- **/products** een rest endpoint voor het ophalen van een lijst producten

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

Met de volgende Dockerfile gebruiken we bovenstaande *pact bestand* en kopieren dit bestand genaamd *pact.json* in de docker container.
```Dockerfile
FROM pactfoundation/pact-stub-server:latest
COPY pact.json  /app/pacts/
ENTRYPOINT ["/app/pact-stub-server", "-d=/app/pacts", "-p=8080"]
EXPOSE 8080
```

Nu hoeven we alleen nog de docker container te bouwen en deze te starten

```cmd
$ docker build -t provider-stub .
$ docker run -p 8080:8080 provider-stub
```

Zodra de container gestart is kan je controleren of de stub het doet door in een browser de url `http://localhost:8080/products` te benaderen. Je moet dan een lijstje van producten terug krijgen.

##  Intergratie test met deze stub

Hoe je deze server stub kan inzetten tijdens het uitvoeren van integratie test wil ik graag demonstreren met een voorbeeld.
De [fabic8 maven plugin](https://maven.fabric8.io/) maakt het mogelijk om voor het uitvoeren van je integratie test een docker container te starten en deze aan het einde van de test weer te stoppen. In onderstaand voorbeeld laten we docker zelf bepalen op welke poort de stub beschikbaar is. De poort waarop de container gestart wordt wordt als environment variabele doorgegeven aan de applicatie.

docker-maven-plugin voert volgens onderstaande setup de volgende stappen uit voordat de integratie tests worden uitgevoerd:
- De docker container gebouwd voor de stub volgens het recept in de opgegeven Dockerfile.
- De gebouwde container wordt gestart. De poort waarop de stub te benaderen is wordt toegekend aan de property *provider-stub.port*. De maven failsafe plugin geeft deze property door als environemt variabele aan de testen applicatie.
- Er wordt gewacht tot de health check van de stub een status code 200 retourneerd.
Na het uitvoeren van de integratie test is deze plugin.

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

application.yaml
```
# For deployment local
spring:
  profiles: default
  
provider:
    url: http://localhost:8081/  
---
# For deployment in Docker (PROD) containers
spring:
  profiles: prod

provider:
    url: http://provider.prod:8080/

---
# For integration testing
spring:
  profiles: integration-test

provider:
    url: http://${dns.host.docker}:${provider-stub.port}/
---
```

```java
@SpringBootTest
@ActiveProfiles("integration-test")
class CapabilitiesServiceIT(){

	@Autowired
    private lateinit var capabilitiesService: CapabilitiesServiceXslt

    @Test
    fun getCapabilitiesTest() {
		...
    }
}
```