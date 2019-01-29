# Stubbing services using pact

##Pact

[Pact.io](http://pact.io)

pact.json
```json
{
  "consumer": {
    "name": "pdok-ngr-metadata"
  },
  "provider": {
    "name": "catalog-service"
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
      "description": "find all branches",
      "provider_state": "Some state",
      "request": {
        "method": "GET",
        "path": "/catalog-service/v0/branch"
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
  }
}
```

Docker.file
```
FROM pactfoundation/pact-stub-server:latest
COPY pact.json  /app/pacts/
ENTRYPOINT ["/app/pact-stub-server", "-d=/app/pacts", "-p=8080"]
EXPOSE 8080
```



##  Using this stub in an integration test



pom.xml
```xml
<project>
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
                    <environmentVariables>
                        <catalog-service.port>${catalog-service.port}</catalog-service.port>
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
                                    <name>catalog-service-stub:0.0.1-it</name>
                                    <alias>catalog-service</alias>
                                    <build>
                                        <dockerFileDir>${project.basedir}/stub/</dockerFileDir>
                                        <tags>
                                            <tag>0.0.1-it</tag>
                                        </tags>
                                    </build>
                                    <run>
                                        <ports>
                                            <port>catalog-service.port:8080</port>
                                        </ports>
                                        <wait>
                                            <http>
                                                <url>http://${dns.host.docker}:${catalog-service.port}/health</url>
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
                        <id>remove-catalog-service-stub</id>
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
```yml
# For deployment local
spring:
  profiles: default
  
---
# For deployment in Docker (PROD) containers
spring:
  profiles: prod

provider:
    url: http://catalog.pdok-int:8080/catalog-service/v0

---
# For integration testing
spring:
  profiles: integration-test

provider:
    url: http://${dns.host.docker}:${catalog-service.port}/catalog-service/v0
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