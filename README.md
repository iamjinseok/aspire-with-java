# aspire-with-java

JAVA SpringBoot의 Backend와 Blazor Frontend를 연결하여 오케스트레이션 하는 Aspire 데모

## Demo
### Aspire 프로젝트 생성
- aspire start application 생성

- https 설정 해제, http로 실행하려면 ASPIRE_ALLOW_UNSECURED_TRANSPORT 설정을 추가
  ``` json
    {
      "$schema": "https://json.schemastore.org/launchsettings.json",
      "profiles": {
        "http": {
          "commandName": "Project",
          "dotnetRunMessages": true,
          "launchBrowser": true,
          "applicationUrl": "http://localhost:15056",
            "environmentVariables": {
                "ASPNETCORE_ENVIRONMENT": "Development",
                "DOTNET_ENVIRONMENT": "Development",
                "DOTNET_DASHBOARD_OTLP_ENDPOINT_URL": "http://localhost:19294",
                "DOTNET_RESOURCE_SERVICE_ENDPOINT_URL": "http://localhost:20297",
                "ASPIRE_ALLOW_UNSECURED_TRANSPORT": "true"
            }
        }
      }
    }
    ```
- 필요시 로그인 토큰 입력

### JAVA Spring Boot 생성
- springboot extension으로 실행
  - springboot : 3.2.5
  - language : JAVA
  - group id : kr.blazorstudy
  - artifact id : weather
  - package type : jar
  - java version : 17
    - 개인 PC에 설치된 것이 17
  - dependencies : Spring Web
  - 완료되면 새로 오픈

- Controller 생성 : Blazor web api Project에서와 같이 반환하도록 구현
  
  - WeatherController 파일 생성
    ``` java
    @RestController
    public class WeatherController {

      @GetMapping("/hello")
      public String hello() {
          return "Hello, Aspire Java!";
      }

      @GetMapping("/weatherforecast")
      public WeatherForecast[] weatherforecast() {

        var random = new Random();
        
        return new WeatherForecast[] {
            new WeatherForecast("2021-10-01", random.nextInt(45), "서울"),
            new WeatherForecast("2021-10-02", random.nextInt(45), "대구"),
            new WeatherForecast("2021-10-03", random.nextInt(45), "부산"),
            new WeatherForecast("2021-10-04", random.nextInt(45), "광주"),
            new WeatherForecast("2021-10-05", random.nextInt(45), "인천")
        };
      }
    }
    ```

  - WeatherForecast.java 파일 생성 : 클래스 자료형

    ``` java
    public class WeatherForecast {
      public String date;
      public int TemperatureC;
      public String Summary;
      public int TemperatureF;

      public WeatherForecast(String date, int TemperatureC, String Summary) {
        this.date = date;
        this.TemperatureC = TemperatureC;
        this.Summary = Summary;
        this.TemperatureF = 32 + (int)(TemperatureC / 0.5556);
      }
    }
    ```

  - gradle build assemble 이후 실행해 보기(동작여부 확인을 위해)
    ``` bash
    java -jar ./build/libs/weather-0.0.1-SNAPSHOT.jar
    ```
### Java Agent 다운로드
  - [OpenTelemetry 참고문서](https://opentelemetry.io/docs/languages/java/automatic/)

### Docker 이미지 생성
- 아래와 같이 Dockerfile 생성
  ```docker
  # 베이스 이미지 설정
  FROM mcr.microsoft.com/openjdk/jdk:17-ubuntu

  # 작업 디렉토리 설정
  WORKDIR /app

  # 빌드 파일 복사
  COPY ./build/libs/aspire-java-0.0.1-SNAPSHOT.jar /app/app.jar
  COPY ./opentelemetry-javaagent.jar /app

  ENV OTEL_BLRP_SCHEDULE_DELAY=OTEL_BLRP_SCHEDULE_DELAY
  ENV OTEL_BSP_SCHEDULE_DELAY=OTEL_BSP_SCHEDULE_DELAY
  ENV OTEL_DOTNET_EXPERIMENTAL_ASPNETCORE_DISABLE_URL_QUERY_REDACTION=OTEL_DOTNET_EXPERIMENTAL_ASPNETCORE_DISABLE_URL_QUERY_REDACTION
  ENV OTEL_DOTNET_EXPERIMENTAL_HTTPCLIENT_DISABLE_URL_QUERY_REDACTION=OTEL_DOTNET_EXPERIMENTAL_HTTPCLIENT_DISABLE_URL_QUERY_REDACTION
  ENV OTEL_DOTNET_EXPERIMENTAL_OTLP_EMIT_EVENT_LOG_ATTRIBUTES=OTEL_DOTNET_EXPERIMENTAL_OTLP_EMIT_EVENT_LOG_ATTRIBUTES
  ENV OTEL_DOTNET_EXPERIMENTAL_OTLP_EMIT_EXCEPTION_LOG_ATTRIBUTES=OTEL_DOTNET_EXPERIMENTAL_OTLP_EMIT_EXCEPTION_LOG_ATTRIBUTES
  ENV OTEL_DOTNET_EXPERIMENTAL_OTLP_RETRY=OTEL_DOTNET_EXPERIMENTAL_OTLP_RETRY
  ENV OTEL_EXPORTER_OTLP_ENDPOINT=OTEL_EXPORTER_OTLP_ENDPOINT
  ENV OTEL_EXPORTER_OTLP_HEADERS=OTEL_EXPORTER_OTLP_HEADERS
  ENV OTEL_EXPORTER_OTLP_PROTOCOL=OTEL_EXPORTER_OTLP_PROTOCOL
  ENV OTEL_METRIC_EXPORT_INTERVAL=OTEL_METRIC_EXPORT_INTERVAL
  ENV OTEL_RESOURCE_ATTRIBUTES=OTEL_RESOURCE_ATTRIBUTES
  ENV OTEL_SERVICE_NAME=OTEL_SERVICE_NAME
  ENV OTEL_TRACES_SAMPLER=OTEL_TRACES_SAMPLER

  # 포트 노출
  EXPOSE 8080

  # 애플리케이션 실행
  ENTRYPOINT ["java","-javaagent:/app/opentelemetry-javaagent.jar", "-jar","/app/app.jar"]
  ```

- docker image 만들기
  ```
  docker build -t aspire-weather:0.0.1 .
  ```

### Aspire 프로젝트에서 JAVA Backend 추가
- AppHost Project의 Program.cs 파일
  - java spring 생성 코드 추가
    ``` cs
    var javaService = builder.AddContainer("aspire-weather", "aspire-weather", "0.0.1")
                            .WithHttpEndpoint(name: "http", port: 8080, targetPort: 8080)
                            .WithOtlpExporter();
    ```
  - blazor frontend 연결 코드 변경
    ``` cs
    var frontend = builder.AddProject<Projects.aspire_java_Web>("webfrontend")
                            .WithExternalHttpEndpoints()
                            .WithReference(javaService.GetEndpoint("http"));
    ```

### Web 프로젝트에서 backend 서비스 설정
- Web Project의 Program.cs 파일
  - backend 서비스 지정
    ``` cs
    builder.Services.AddHttpClient<WeatherApiClient>(client => client.BaseAddress = new("http://aspire-weather"));
    ```
