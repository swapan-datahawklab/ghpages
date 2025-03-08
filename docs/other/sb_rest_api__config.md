**Swapan**: spring boot rest api with postgres db jdbctemplate micro benchmark

**ChatGPT**: To create a Spring Boot REST API with PostgreSQL database access using `JdbcTemplate` and conduct a micro-benchmark, follow these steps:

### 1. **Set Up Spring Boot Project**

You can use Spring Initializr to create the project.

- Go to [Spring Initializr](https://start.spring.io/).
- Choose the following:
  - **Project**: Maven
  - **Language**: Java
  - **Spring Boot Version**: 3.x.x (latest stable)
  - **Dependencies**: 
    - Spring Web
    - Spring Data JDBC (or JdbcTemplate)
    - PostgreSQL Driver

Download the project, unzip it, and import it into your IDE.

### 2. **Configure PostgreSQL Database**

In `src/main/resources/application.properties`, configure PostgreSQL connection details:

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/your_db_name
spring.datasource.username=your_db_username
spring.datasource.password=your_db_password
spring.datasource.driver-class-name=org.postgresql.Driver

spring.jpa.hibernate.ddl-auto=update
```

### 3. **Create Model Class**

Define your model, which represents the entity (table) in the database.

```java
public class Employee {
    private Long id;
    private String name;
    private String department;
    private Double salary;

    // Getters and Setters
}
```

### 4. **Create Repository with JdbcTemplate**

Use `JdbcTemplate` to perform database operations.

```java
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.stereotype.Repository;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.List;

@Repository
public class EmployeeRepository {

    private final JdbcTemplate jdbcTemplate;

    public EmployeeRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    private RowMapper<Employee> rowMapper = new RowMapper<Employee>() {
        @Override
        public Employee mapRow(ResultSet rs, int rowNum) throws SQLException {
            Employee employee = new Employee();
            employee.setId(rs.getLong("id"));
            employee.setName(rs.getString("name"));
            employee.setDepartment(rs.getString("department"));
            employee.setSalary(rs.getDouble("salary"));
            return employee;
        }
    };

    public List<Employee> findAll() {
        String sql = "SELECT * FROM employees";
        return jdbcTemplate.query(sql, rowMapper);
    }

    public void save(Employee employee) {
        String sql = "INSERT INTO employees (name, department, salary) VALUES (?, ?, ?)";
        jdbcTemplate.update(sql, employee.getName(), employee.getDepartment(), employee.getSalary());
    }

    // Other CRUD methods
}
```

### 5. **Create REST Controller**

Expose the repository through a REST API.

```java
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/employees")
public class EmployeeController {

    private final EmployeeRepository employeeRepository;

    public EmployeeController(EmployeeRepository employeeRepository) {
        this.employeeRepository = employeeRepository;
    }

    @GetMapping
    public List<Employee> getAllEmployees() {
        return employeeRepository.findAll();
    }

    @PostMapping
    public void createEmployee(@RequestBody Employee employee) {
        employeeRepository.save(employee);
    }
}
```

### 6. **Micro-Benchmarking Setup**

Use `Spring Boot Actuator` to capture metrics or use an external benchmarking tool like `Apache JMeter` or `wrk`. Here, we’ll focus on using `Java System.nanoTime()` for basic micro-benchmarking of a database query:

```java
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/employees")
public class BenchmarkController {

    private final EmployeeRepository employeeRepository;

    public BenchmarkController(EmployeeRepository employeeRepository) {
        this.employeeRepository = employeeRepository;
    }

    @GetMapping("/benchmark")
    public String runBenchmark() {
        long startTime = System.nanoTime();
        
        // Call the DB operation (findAll)
        List<Employee> employees = employeeRepository.findAll();
        
        long endTime = System.nanoTime();
        long duration = (endTime - startTime) / 1000000; // Time in milliseconds
        
        return "Query executed in " + duration + " ms";
    }
}
```

### 7. **Running the Application**

- Build and run the application using your IDE or command line:
  ```bash
  mvn spring-boot:run
  ```

- Use `Postman` or `curl` to interact with the API:
  - To get all employees:
    ```bash
    curl -X GET http://localhost:8080/api/employees
    ```
  - To create a new employee:
    ```bash
    curl -X POST http://localhost:8080/api/employees -H "Content-Type: application/json" -d '{"name": "John", "department": "HR", "salary": 50000}'
    ```

### 8. **Benchmarking Results**

- For quick benchmarking, you can hit the endpoint `/api/employees/benchmark` which will return the execution time for the query in milliseconds.
  
To perform more comprehensive benchmarking, you can consider using tools like:

- **Apache JMeter**: For load and stress testing.
- **wrk**: For HTTP benchmarking and testing under load.

This setup gives you the foundation for creating a micro-benchmark of your Spring Boot REST API using JdbcTemplate and PostgreSQL.

