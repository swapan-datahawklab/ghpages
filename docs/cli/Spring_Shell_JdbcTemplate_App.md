**Swapan**: create a full spring shell jdbctemplate based application interactive

**ChatGPT**: Below is an example of a minimal but complete Spring Boot application that uses **Spring Shell** and **JdbcTemplate**. This sample showcases how to set up interactive commands that manipulate data in an in-memory H2 database. You can type commands directly in the shell (once the application starts) to store and retrieve data.

---

## Project Structure

```
spring-shell-jdbctemplate-app
 ┣─ pom.xml
 ┗─ src
    ┗─ main
       ┗─ java
          ┗─ com
             ┗─ example
                ┗─ shelljdbctemplate
                   ┣─ Application.java
                   ┣─ Person.java
                   ┣─ PersonRepository.java
                   ┗─ ShellCommands.java
```

---

## pom.xml

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" 
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example.shelljdbctemplate</groupId>
    <artifactId>spring-shell-jdbctemplate-app</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <name>Spring Shell JDBC Template Demo</name>
    <description>Spring Boot + Spring Shell + JdbcTemplate Example</description>

    <properties>
        <java.version>17</java.version>
        <spring.boot.version>3.0.5</spring.boot.version>
        <spring.shell.version>3.0.3</spring.shell.version>
    </properties>

    <dependencies>
        <!-- Spring Boot -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
            <version>${spring.boot.version}</version>
        </dependency>

        <!-- Spring Boot JDBC -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
            <version>${spring.boot.version}</version>
        </dependency>

        <!-- Spring Shell -->
        <dependency>
            <groupId>org.springframework.shell</groupId>
            <artifactId>spring-shell-starter</artifactId>
            <version>${spring.shell.version}</version>
        </dependency>

        <!-- H2 in-memory database -->
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>
        
    </dependencies>

    <build>
        <plugins>
            <!-- Spring Boot Maven Plugin -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>${spring.boot.version}</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## Application.java

```java
package com.example.shelljdbctemplate;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

## Person.java

```java
package com.example.shelljdbctemplate;

public class Person {
    private Long id;
    private String firstName;
    private String lastName;

    // Constructors
    public Person() {
    }

    public Person(Long id, String firstName, String lastName) {
        this.id = id;
        this.firstName = firstName;
        this.lastName = lastName;
    }

    // Getters and Setters
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

    // Optional convenience method
    @Override
    public String toString() {
        return "Person [id=" + id + ", firstName=" + firstName +
               ", lastName=" + lastName + "]";
    }
}
```

---

## PersonRepository.java

```java
package com.example.shelljdbctemplate;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.stereotype.Repository;

import javax.annotation.PostConstruct;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.List;

@Repository
public class PersonRepository {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    // Initialize the table for demonstration
    @PostConstruct
    public void init() {
        jdbcTemplate.execute("DROP TABLE IF EXISTS person");
        jdbcTemplate.execute("CREATE TABLE person (" +
                "id IDENTITY PRIMARY KEY, " +
                "first_name VARCHAR(255), " +
                "last_name VARCHAR(255))");
    }

    public int save(String firstName, String lastName) {
        return jdbcTemplate.update(
                "INSERT INTO person (first_name, last_name) VALUES (?, ?)",
                firstName, lastName
        );
    }

    public List<Person> findAll() {
        return jdbcTemplate.query("SELECT * FROM person", new PersonRowMapper());
    }

    public Person findById(Long id) {
        return jdbcTemplate.queryForObject(
                "SELECT * FROM person WHERE id = ?",
                new PersonRowMapper(),
                id
        );
    }

    public int update(Long id, String firstName, String lastName) {
        return jdbcTemplate.update(
                "UPDATE person SET first_name = ?, last_name = ? WHERE id = ?",
                firstName, lastName, id
        );
    }

    public int deleteById(Long id) {
        return jdbcTemplate.update("DELETE FROM person WHERE id = ?", id);
    }

    static class PersonRowMapper implements RowMapper<Person> {
        @Override
        public Person mapRow(ResultSet rs, int rowNum) throws SQLException {
            Person p = new Person();
            p.setId(rs.getLong("id"));
            p.setFirstName(rs.getString("first_name"));
            p.setLastName(rs.getString("last_name"));
            return p;
        }
    }
}
```

---

## ShellCommands.java

```java
package com.example.shelljdbctemplate;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.shell.standard.ShellComponent;
import org.springframework.shell.standard.ShellMethod;

import java.util.List;

@ShellComponent
public class ShellCommands {

    @Autowired
    private PersonRepository personRepository;

    @ShellMethod(key = "add-person", value = "Add a new person: Usage -> add-person <firstName> <lastName>")
    public String addPerson(String firstName, String lastName) {
        personRepository.save(firstName, lastName);
        return "Person added: " + firstName + " " + lastName;
    }

    @ShellMethod(key = "list-all", value = "List all persons in the database.")
    public List<Person> listAll() {
        return personRepository.findAll();
    }

    @ShellMethod(key = "find-person", value = "Find person by ID: Usage -> find-person <id>")
    public Person findPerson(Long id) {
        return personRepository.findById(id);
    }

    @ShellMethod(key = "update-person", value = "Update person: Usage -> update-person <id> <firstName> <lastName>")
    public String updatePerson(Long id, String firstName, String lastName) {
        int updated = personRepository.update(id, firstName, lastName);
        if (updated == 1) {
            return "Person updated: " + id + " " + firstName + " " + lastName;
        }
        return "No person found with id " + id;
    }

    @ShellMethod(key = "delete-person", value = "Delete person by ID: Usage -> delete-person <id>")
    public String deletePerson(Long id) {
        int deleted = personRepository.deleteById(id);
        if (deleted == 1) {
            return "Person deleted with id " + id;
        }
        return "No person found with id " + id;
    }
}
```

---

## How to Run

1. **Build and package**:

   ```bash
   mvn clean package
   ```
   
2. **Run the Spring Boot application**:

   ```bash
   java -jar target/spring-shell-jdbctemplate-app-1.0.0-SNAPSHOT.jar
   ```
   
3. **Interact with the Shell**:

   After the application starts, you’ll see a shell prompt (by default something like `shell:>`). You can type any of the shell commands (listed below) to interact with the application.

   Example commands:
   
   - **Add Person**:
     ```
     shell:> add-person John Doe
     ```
   - **List All**:
     ```
     shell:> list-all
     ```
   - **Find by ID**:
     ```
     shell:> find-person 1
     ```
   - **Update Person**:
     ```
     shell:> update-person 1 Jane Smith
     ```
   - **Delete Person**:
     ```
     shell:> delete-person 1
     ```

---

## Explanation

1. **Spring Boot + Spring Shell**:
   - The `@SpringBootApplication` annotation in `Application.java` enables Spring Boot’s auto-configuration.
   - The `spring-shell-starter` dependency automatically sets up a command-line shell environment.

2. **JdbcTemplate**:
   - The `PersonRepository.java` demonstrates how to use `JdbcTemplate` for basic CRUD operations: `save`, `findAll`, `findById`, `update`, and `deleteById`.
   - We use an in-memory H2 database; on app startup, we drop and recreate the `person` table (for demo purposes).

3. **Shell Commands**:
   - In `ShellCommands.java`, we create interactive commands annotated by `@ShellMethod`.
   - Each command calls the corresponding method in `PersonRepository`.
   - Return types in the command methods (e.g., `List<Person>`) are printed directly in the shell environment.

This provides a **fully functional** starter application to practice using **Spring Shell** with **JdbcTemplate** in an **interactive** CLI environment. Happy coding!

**ChatGPT**: **Piecing together dependencies**

I’m setting up a Spring Shell project with JDBC Template, planning to include dependencies like spring shell, spring boot, and H2 as the database.

