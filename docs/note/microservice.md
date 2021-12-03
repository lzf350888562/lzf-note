Spring Cloud Alibaba由于没有纳入到Spring Cloud的主版本管理中，所以我们需要自己去引入其版本信息，

```
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Finchley.SR1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>0.2.1.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```



# spring cloud alibaba

版本对应

| Spring Boot | Spring Cloud | Spring Cloud Alibaba |
| :---------- | :----------- | :------------------- |
| 2.1.x       | Greenwich    | 0.9.x                |
| 2.0.x       | Finchley     | 0.2.x                |
| 1.5.x       | Edgware      | 0.1.x                |
| 1.5.x       | Dalston      | 0.1.x                |