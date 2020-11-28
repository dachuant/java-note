```
server:
  port: 8888
spring:
  application:
    name: service-config
  profiles:
    active: native
  cloud:
    config:
      server:
        native:
          search-locations: classpath:/service-hi/
```

