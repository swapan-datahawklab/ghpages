**Swapan**: spring boot jdbctemplate application that can be run as both a spring shell console application or a spring rest web application 

**ChatGPT**: Below is a simple illustrative example of a Spring Boot application that:

1. Uses **Spring Shell** to provide a console-based interface.  
2. Exposes a **REST API** for the same functionality.  
3. Utilizes **JdbcTemplate** for data operations.  

You can run it in “shell mode” or “web mode” by passing different command-line arguments (for example, `shell` vs. `web`).  

---

## Project Structure

A minimal Maven-based project might look like this:

```
spring-boot-shell-rest-jdbc-template/
├── pom.xml
└── src
    └── main
        ├── java
        │   └── com
        │       └── example
        │           └── shellrest
        │               ├── Application.java
        │               ├── config
        │               │   ├── DataSourceConfig.java
        │               │   └── ShellConfig.java
        │               ├── controller
        │               │   └── CustomerController.java
        │               ├── domain
        │               │   └── Customer.java
        │               ├── repository
        │               │   └── CustomerRepository.java
        │               └── shell
        │                   └── CustomerCommands.java
        └── resources
            └── application.properties
```

Below are the core files. Adjust package names and group IDs as you wish.

---

## 1. `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" 
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>spring-boot-shell-rest-jdbc-template</artifactId>
    <version>1.0.0</version>

    <properties>
        <java.version>17</java.version>
        <spring.boot.version>3.0.5</spring.boot.version>
    </properties>

    <dependencies>
        <!-- Spring Boot Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>${spring.boot.version}</version>
        </dependency>

        <!-- Spring JDBC -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
            <version>${spring.boot.version}</version>
        </dependency>

        <!-- H2 in-memory database (for demonstration; use a real DB for production) -->
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- Spring Shell -->
        <dependency>
            <groupId>org.springframework.shell</groupId>
            <artifactId>spring-shell-starter</artifactId>
            <version>3.0.3</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Spring Boot Maven plugin -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>${spring.boot.version}</version>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## 2. `Application.java`

This class decides whether to run the application as a **Spring Shell** console application or as a **web** (REST) application, depending on command-line arguments.

```java
package com.example.shellrest;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        // If the first argument is "shell", run in shell mode; otherwise run as a web app
        if (args.length > 0 && "shell".equalsIgnoreCase(args[0])) {
            // Shell mode: run the shell-enabled context
            // The presence of spring-shell-starter plus auto configuration can handle that.
            SpringApplication.run(Application.class, args);
        } else {
            // Web mode: run the web application (port, etc.)
            SpringApplication.run(Application.class, args);
        }
    }
}
```

*Note:* In practice, Spring Shell also works in the same application context. By default, if you just do `SpringApplication.run(Application.class, args);` with the dependencies, Spring Shell environment is started. But typically, you might only want the shell environment in “shell mode.” This example uses a simple argument check to differentiate.

---

## 3. `DataSourceConfig.java`

Configures the data source and `JdbcTemplate`. If you rely on Spring Boot’s auto-configuration and `application.properties`, you might not even need this. But here is an explicit example:

```java
package com.example.shellrest.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.datasource.DriverManagerDataSource;

import javax.sql.DataSource;

@Configuration
public class DataSourceConfig {

    @Bean
    public DataSource dataSource() {
        // For demonstration, we use an in-memory H2 database.
        // For production, configure the JDBC URL, username, password, etc. in application.properties
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setUrl("jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE");
        dataSource.setUsername("sa");
        dataSource.setPassword("");
        dataSource.setDriverClassName("org.h2.Driver");
        return dataSource;
    }

    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

---

## 4. `application.properties`

For a local in-memory test, the minimal config can be:

```properties
# Logging
logging.level.org.springframework=INFO
logging.level.com.example.shellrest=DEBUG

# If you want to rely on spring.datasource.* config, you could do it here:
# spring.datasource.url=jdbc:h2:mem:testdb
# spring.datasource.driver-class-name=org.h2.Driver
# spring.datasource.username=sa
# spring.datasource.password=
```

---

## 5. `Customer.java` (Domain Object)

```java
package com.example.shellrest.domain;

public class Customer {
    private Long id;
    private String firstName;
    private String lastName;

    public Customer() {}

    public Customer(Long id, String firstName, String lastName) {
        this.id = id;
        this.firstName = firstName;
        this.lastName = lastName;
    }

    // Getters and setters
    
    public Long getId() {
        return id;
    }
    public void setId(Long id) {
        this.id = id;
    }
    public String getFirstName() {
        return firstName;
    }    
    public void setFirstName(String firstName) {
        this.firstName = firstName;
    }    
    public String getLastName() {
        return lastName;
    }    
    public void setLastName(String lastName) {
        this.lastName = lastName;
    }
}
```

---

## 6. `CustomerRepository.java`

A simple repository using `JdbcTemplate` to perform basic CRUD. On startup, we can create a simple table:

```java
package com.example.shellrest.repository;

import com.example.shellrest.domain.Customer;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.stereotype.Repository;
import javax.annotation.PostConstruct;
import java.util.List;

@Repository
public class CustomerRepository {

    private final JdbcTemplate jdbcTemplate;

    public CustomerRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    // Create a simple table on startup
    @PostConstruct
    public void initDb() {
        jdbcTemplate.execute("""
                CREATE TABLE IF NOT EXISTS customers (
                  id BIGINT AUTO_INCREMENT PRIMARY KEY,
                  first_name VARCHAR(50),
                  last_name VARCHAR(50)
                )
                """);
    }

    public List<Customer> findAll() {
        String sql = "SELECT id, first_name, last_name FROM customers";
        return jdbcTemplate.query(sql, rowMapper());
    }

    public Customer findById(Long id) {
        String sql = "SELECT id, first_name, last_name FROM customers WHERE id = ?";
        return jdbcTemplate.queryForObject(sql, rowMapper(), id);
    }

    public int save(Customer customer) {
        String sql = "INSERT INTO customers (first_name, last_name) VALUES (?, ?)";
        return jdbcTemplate.update(sql, customer.getFirstName(), customer.getLastName());
    }

    public int update(Customer customer) {
        String sql = "UPDATE customers SET first_name=?, last_name=? WHERE id=?";
        return jdbcTemplate.update(sql, customer.getFirstName(), customer.getLastName(), customer.getId());
    }

    public int delete(Long id) {
        String sql = "DELETE FROM customers WHERE id=?";
        return jdbcTemplate.update(sql, id);
    }

    private RowMapper<Customer> rowMapper() {
        return (rs, rowNum) -> {
            Customer c = new Customer();
            c.setId(rs.getLong("id"));
            c.setFirstName(rs.getString("first_name"));
            c.setLastName(rs.getString("last_name"));
            return c;
        };
    }
}
```

---

## 7. `CustomerController.java` (REST)

A Spring REST controller that exposes CRUD endpoints:

```java
package com.example.shellrest.controller;

import com.example.shellrest.domain.Customer;
import com.example.shellrest.repository.CustomerRepository;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/customers")
public class CustomerController {

    private final CustomerRepository repository;

    public CustomerController(CustomerRepository repository) {
        this.repository = repository;
    }

    @GetMapping
    public List<Customer> findAll() {
        return repository.findAll();
    }

    @GetMapping("/{id}")
    public Customer findById(@PathVariable Long id) {
        return repository.findById(id);
    }

    @PostMapping
    public ResponseEntity<String> create(@RequestBody Customer customer) {
        int result = repository.save(customer);
        return (result == 1)
            ? ResponseEntity.ok("Customer created successfully.")
            : ResponseEntity.badRequest().body("Failed to create customer.");
    }

    @PutMapping("/{id}")
    public ResponseEntity<String> update(@PathVariable Long id, @RequestBody Customer updatedCustomer) {
        updatedCustomer.setId(id);
        int result = repository.update(updatedCustomer);
        return (result == 1)
            ? ResponseEntity.ok("Customer updated successfully.")
            : ResponseEntity.badRequest().body("Failed to update customer.");
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<String> delete(@PathVariable Long id) {
        int result = repository.delete(id);
        return (result == 1)
            ? ResponseEntity.ok("Customer deleted successfully.")
            : ResponseEntity.badRequest().body("Failed to delete customer.");
    }
}
```

---

## 8. `CustomerCommands.java` (Spring Shell Commands)

This class provides the “shell” commands you can interact with in console mode. Once the app is started in shell mode, you can type these commands to interact with customers in the DB.

```java
package com.example.shellrest.shell;

import com.example.shellrest.domain.Customer;
import com.example.shellrest.repository.CustomerRepository;
import org.springframework.shell.standard.ShellComponent;
import org.springframework.shell.standard.ShellMethod;

import java.util.List;

@ShellComponent
public class CustomerCommands {

    private final CustomerRepository repository;

    public CustomerCommands(CustomerRepository repository) {
        this.repository = repository;
    }

    @ShellMethod("List all customers.")
    public List<Customer> listCustomers() {
        return repository.findAll();
    }

    @ShellMethod("Find a customer by ID.")
    public Customer getCustomer(Long id) {
        return repository.findById(id);
    }

    @ShellMethod("Add a new customer (usage: addCustomer <firstName> <lastName>)")
    public String addCustomer(String firstName, String lastName) {
        Customer customer = new Customer(null, firstName, lastName);
        int rows = repository.save(customer);
        return (rows > 0) ? "Customer added." : "Error adding customer.";
    }

    @ShellMethod("Update an existing customer (usage: updateCustomer <id> <firstName> <lastName>)")
    public String updateCustomer(Long id, String firstName, String lastName) {
        Customer customer = new Customer(id, firstName, lastName);
        int rows = repository.update(customer);
        return (rows > 0) ? "Customer updated." : "Error updating customer.";
    }

    @ShellMethod("Delete a customer by ID.")
    public String deleteCustomer(Long id) {
        int rows = repository.delete(id);
        return (rows > 0) ? "Customer deleted." : "Error deleting customer.";
    }
}
```

---

## How to Run

1. **Shell Mode**  
   ```bash
   mvn clean package
   java -jar target/spring-boot-shell-rest-jdbc-template-1.0.0.jar shell
   ```
   - You will see a shell prompt like `shell:>`.  
   - Try commands like:  
     - `listCustomers`  
     - `addCustomer John Doe`  
     - `listCustomers`  
     - `getCustomer 1`  
     - `deleteCustomer 1`

2. **Web (REST) Mode**  
   ```bash
   java -jar target/spring-boot-shell-rest-jdbc-template-1.0.0.jar
   ```
   - By default, it will start on port 8080.  
   - Example endpoints:  
     - `GET http://localhost:8080/api/customers`  
     - `POST http://localhost:8080/api/customers` with JSON body like:
       ```json
       {
         "firstName": "Jane",
         "lastName": "Smith"
       }
       ```  
     - `GET http://localhost:8080/api/customers/1`

That’s it! You have a **single Spring Boot application** that can be run as either a **Spring Shell** console or a **RESTful web service**, and uses **JdbcTemplate** for data persistence. Adjust to your needs (e.g., use a real database or incorporate Docker, etc.).

**ChatGPT**: **Crafting the foundation**

I’m outlining the creation of a Spring Boot application utilizing JDBCTemplate, capable of operating as both a Spring Shell console and a REST-based application.

**Crafting the launch**

I’m coding a main class to toggle between Spring Shell and Web modes using conditional logic based on command-line arguments, and preparing for a minimal configuration setup.

