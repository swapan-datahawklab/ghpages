**Swapan**: rest api to lookup properties stored in database spring boot

**ChatGPT**: To create a REST API in Spring Boot to look up properties stored in a database, you can follow these steps:

1. **Set Up Your Spring Boot Application**: Ensure that you have a basic Spring Boot application with dependencies for Spring Web, Spring Data JPA, and your database connector (e.g., MySQL, PostgreSQL).

2. **Create Your Entity Class**: This represents the table in your database where the properties are stored.

3. **Create Your Repository Interface**: This will provide basic CRUD operations.

4. **Create Your Service Class**: To handle the business logic.

5. **Create Your REST Controller**: To expose the API endpoint.

Here's how to implement this:

### Step 1: Add Dependencies
Add the necessary dependencies to your `pom.xml` or `build.gradle`.

**`pom.xml`**:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>com.h2database</groupId> <!-- Or your preferred database -->
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

### Step 2: Create the Entity Class
Create a class that maps to the database table.

**`Property.java`**:
```java
package com.example.demo.model;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

@Entity
public class Property {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String key;

    private String value;

    // Getters and setters
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getKey() {
        return key;
    }

    public void setKey(String key) {
        this.key = key;
    }

    public String getValue() {
        return value;
    }

    public void setValue(String value) {
        this.value = value;
    }
}
```

### Step 3: Create the Repository Interface
Create a repository interface for the `Property` entity.

**`PropertyRepository.java`**:
```java
package com.example.demo.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import com.example.demo.model.Property;
import java.util.Optional;

public interface PropertyRepository extends JpaRepository<Property, Long> {
    Optional<Property> findByKey(String key);
}
```

### Step 4: Create the Service Class
Create a service to handle the business logic.

**`PropertyService.java`**:
```java
package com.example.demo.service;

import com.example.demo.model.Property;
import com.example.demo.repository.PropertyRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.Optional;

@Service
public class PropertyService {

    @Autowired
    private PropertyRepository propertyRepository;

    public Optional<Property> getPropertyByKey(String key) {
        return propertyRepository.findByKey(key);
    }
}
```

### Step 5: Create the REST Controller
Create a controller to expose the REST endpoint.

**`PropertyController.java`**:
```java
package com.example.demo.controller;

import com.example.demo.model.Property;
import com.example.demo.service.PropertyService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Optional;

@RestController
@RequestMapping("/api/properties")
public class PropertyController {

    @Autowired
    private PropertyService propertyService;

    @GetMapping("/{key}")
    public ResponseEntity<Property> getPropertyByKey(@PathVariable String key) {
        Optional<Property> property = propertyService.getPropertyByKey(key);
        return property.map(ResponseEntity::ok)
                       .orElseGet(() -> ResponseEntity.notFound().build());
    }
}
```

### Step 6: Configure Your Database
Configure your `application.properties` or `application.yml` file with your database details.

**`application.properties`**:
```properties
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=password
spring.jpa.hibernate.ddl-auto=update
```

### Final Step: Run the Application
Run your Spring Boot application. You can access the endpoint by sending a `GET` request to:
```
http://localhost:8080/api/properties/{key}
```

This will return the property associated with the given key from the database.

**Swapan**: convert to oracle and use seperte database configuration clas

**ChatGPT**: To convert the configuration to use Oracle as the database and create a separate database configuration class, follow these steps:

1. **Add Oracle Database Dependency**
2. **Update the `application.properties`**
3. **Create a Database Configuration Class**

### Step 1: Add Oracle Database Dependency

Add the Oracle JDBC dependency to your `pom.xml`:

```xml
<dependency>
    <groupId>com.oracle.database.jdbc</groupId>
    <artifactId>ojdbc8</artifactId>
    <version>19.8.0.0</version> <!-- Ensure this version matches your setup -->
</dependency>
```

### Step 2: Update `application.properties`

Update your `application.properties` to remove the database configuration details, as these will be provided in a separate configuration class.

**`application.properties`**:
```properties
spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.Oracle12cDialect
```

### Step 3: Create a Separate Database Configuration Class

Create a class to handle the database configuration using `DataSource` and `EntityManagerFactory`.

**`OracleDataSourceConfig.java`**:
```java
package com.example.demo.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.orm.jpa.JpaTransactionManager;
import org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter;

import javax.sql.DataSource;
import java.util.Properties;

@Configuration
@EnableJpaRepositories(
        basePackages = "com.example.demo.repository",
        entityManagerFactoryRef = "entityManagerFactory",
        transactionManagerRef = "transactionManager"
)
public class OracleDataSourceConfig {

    @Value("${spring.datasource.url}")
    private String dbUrl;

    @Value("${spring.datasource.username}")
    private String dbUsername;

    @Value("${spring.datasource.password}")
    private String dbPassword;

    @Value("${spring.datasource.driver-class-name}")
    private String dbDriverClassName;

    @Bean
    @Primary
    public DataSource dataSource() {
        return DataSourceBuilder.create()
                .url(dbUrl)
                .username(dbUsername)
                .password(dbPassword)
                .driverClassName(dbDriverClassName)
                .build();
    }

    @Bean
    @Primary
    public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource());
        em.setPackagesToScan("com.example.demo.model");

        HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
        em.setJpaVendorAdapter(vendorAdapter);
        em.setJpaProperties(hibernateProperties());

        return em;
    }

    @Bean
    @Primary
    public PlatformTransactionManager transactionManager() {
        JpaTransactionManager transactionManager = new JpaTransactionManager();
        transactionManager.setEntityManagerFactory(entityManagerFactory().getObject());
        return transactionManager;
    }

    private Properties hibernateProperties() {
        Properties properties = new Properties();
        properties.setProperty("hibernate.dialect", "org.hibernate.dialect.Oracle12cDialect");
        properties.setProperty("hibernate.hbm2ddl.auto", "update");
        return properties;
    }
}
```

### Step 4: Update `application.properties`

Add Oracle-specific properties for database connection.

**`application.properties`**:
```properties
spring.datasource.url=jdbc:oracle:thin:@//localhost:1521/xe
spring.datasource.username=yourUsername
spring.datasource.password=yourPassword
spring.datasource.driver-class-name=oracle.jdbc.OracleDriver
```

### Explanation:

- **`OracleDataSourceConfig` Class**: This class sets up the `DataSource`, `EntityManagerFactory`, and `TransactionManager` beans, which are used by Spring Data JPA to connect to and interact with the Oracle database.
- **`hibernateProperties()`**: Sets up additional Hibernate properties.
- **`@Primary` Annotation**: Ensures that these beans are used as the primary configuration.

### Final Step: Run Your Application

Run your Spring Boot application, and it will use Oracle as the database for managing property lookups. Make sure your Oracle database is running and accessible with the provided credentials and connection details.

**Swapan**: convert to jdbctemplate

**ChatGPT**: To convert the application to use `JdbcTemplate` instead of Spring Data JPA, follow these steps:

1. **Remove JPA Dependencies**: Ensure your `pom.xml` or `build.gradle` does not include JPA-related dependencies.
2. **Create a `JdbcTemplate` Configuration Class**.
3. **Implement a DAO Class** to handle database operations.
4. **Create a REST Controller** to expose the API.

### Step 1: Update Dependencies

Ensure your `pom.xml` includes the necessary JDBC dependencies:

**`pom.xml`**:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>com.oracle.database.jdbc</groupId>
    <artifactId>ojdbc8</artifactId>
    <version>19.8.0.0</version>
</dependency>
```

### Step 2: Create a Database Configuration Class

Create a configuration class to set up `JdbcTemplate`.

**`JdbcTemplateConfig.java`**:
```java
package com.example.demo.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.datasource.DriverManagerDataSource;

import javax.sql.DataSource;

@Configuration
public class JdbcTemplateConfig {

    @Value("${spring.datasource.url}")
    private String dbUrl;

    @Value("${spring.datasource.username}")
    private String dbUsername;

    @Value("${spring.datasource.password}")
    private String dbPassword;

    @Value("${spring.datasource.driver-class-name}")
    private String dbDriverClassName;

    @Bean
    public DataSource dataSource() {
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setUrl(dbUrl);
        dataSource.setUsername(dbUsername);
        dataSource.setPassword(dbPassword);
        dataSource.setDriverClassName(dbDriverClassName);
        return dataSource;
    }

    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

### Step 3: Implement the DAO Class

Create a DAO class to handle database operations using `JdbcTemplate`.

**`PropertyDao.java`**:
```java
package com.example.demo.dao;

import com.example.demo.model.Property;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.stereotype.Repository;

import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.Optional;

@Repository
public class PropertyDao {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    private static final String SELECT_PROPERTY_BY_KEY = "SELECT * FROM property WHERE key = ?";

    public Optional<Property> getPropertyByKey(String key) {
        return jdbcTemplate.query(SELECT_PROPERTY_BY_KEY, new Object[]{key}, new PropertyRowMapper())
                .stream()
                .findFirst();
    }

    private static class PropertyRowMapper implements RowMapper<Property> {
        @Override
        public Property mapRow(ResultSet rs, int rowNum) throws SQLException {
            Property property = new Property();
            property.setId(rs.getLong("id"));
            property.setKey(rs.getString("key"));
            property.setValue(rs.getString("value"));
            return property;
        }
    }
}
```

### Step 4: Create the Entity Class

**`Property.java`**:
```java
package com.example.demo.model;

public class Property {

    private Long id;
    private String key;
    private String value;

    // Getters and setters
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getKey() {
        return key;
    }

    public void setKey(String key) {
        this.key = key;
    }

    public String getValue() {
        return value;
    }

    public void setValue(String value) {
        this.value = value;
    }
}
```

### Step 5: Create the REST Controller

**`PropertyController.java`**:
```java
package com.example.demo.controller;

import com.example.demo.dao.PropertyDao;
import com.example.demo.model.Property;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Optional;

@RestController
@RequestMapping("/api/properties")
public class PropertyController {

    @Autowired
    private PropertyDao propertyDao;

    @GetMapping("/{key}")
    public ResponseEntity<Property> getPropertyByKey(@PathVariable String key) {
        Optional<Property> property = propertyDao.getPropertyByKey(key);
        return property.map(ResponseEntity::ok)
                       .orElseGet(() -> ResponseEntity.notFound().build());
    }
}
```

### Step 6: Configure `application.properties`

Configure the Oracle database connection properties in `application.properties`.

**`application.properties`**:
```properties
spring.datasource.url=jdbc:oracle:thin:@//localhost:1521/xe
spring.datasource.username=yourUsername
spring.datasource.password=yourPassword
spring.datasource.driver-class-name=oracle.jdbc.OracleDriver
```

### Final Step: Run Your Application

Run your Spring Boot application. Your `JdbcTemplate`-based REST API will now be able to retrieve properties stored in the Oracle database using the `GET` endpoint:

```
http://localhost:8080/api/properties/{key}
```

