**Swapan**: maven artifactory client to search for latest docker image in artifactory docker registry

**ChatGPT**: To search for the latest Docker image in an Artifactory Docker registry using a Maven-based client, you can use the Artifactory REST API, as Maven itself does not have built-in functionality to directly interact with Docker registries. Here’s a step-by-step guide on how to achieve this:

### 1. Set Up Your Maven Project

First, ensure you have a Maven project set up. You can create a new Maven project using the following command:

```bash
mvn archetype:generate -DgroupId=com.example -DartifactId=docker-image-search -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
```

### 2. Add Dependencies

In your `pom.xml`, add the necessary dependencies. You will need a library for HTTP requests, such as Apache HttpClient:

```xml
<dependencies>
    <dependency>
        <groupId>org.apache.httpcomponents</groupId>
        <artifactId>httpclient</artifactId>
        <version>4.5.13</version>
    </dependency>
    <dependency>
        <groupId>com.google.gson</groupId>
        <artifactId>gson</artifactId>
        <version>2.8.6</version>
    </dependency>
</dependencies>
```

### 3. Write the Java Code

Here’s an example of how to write the Java code to search for the latest Docker image in the Artifactory Docker registry:

```java
import org.apache.http.HttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;
import com.google.gson.JsonArray;
import com.google.gson.JsonObject;
import com.google.gson.JsonParser;

public class DockerImageSearcher {
    private static final String ARTIFACTORY_URL = "https://your-artifactory-url/artifactory";
    private static final String REPO = "your-docker-repo";
    private static final String IMAGE_NAME = "your-image-name";
    private static final String API_KEY = "your-api-key";

    public static void main(String[] args) {
        try {
            CloseableHttpClient client = HttpClients.createDefault();
            String url = ARTIFACTORY_URL + "/api/docker/" + REPO + "/v2/" + IMAGE_NAME + "/tags/list";
            HttpGet request = new HttpGet(url);
            request.setHeader("X-JFrog-Art-Api", API_KEY);

            HttpResponse response = client.execute(request);
            String jsonResponse = EntityUtils.toString(response.getEntity());

            JsonObject jsonObject = JsonParser.parseString(jsonResponse).getAsJsonObject();
            JsonArray tags = jsonObject.getAsJsonArray("tags");

            String latestTag = getLatestTag(tags);
            System.out.println("Latest Tag: " + latestTag);

            client.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static String getLatestTag(JsonArray tags) {
        String latestTag = null;
        // Implement your logic to determine the latest tag based on your criteria
        for (int i = 0; i < tags.size(); i++) {
            String tag = tags.get(i).getAsString();
            if (latestTag == null || tag.compareTo(latestTag) > 0) {
                latestTag = tag;
            }
        }
        return latestTag;
    }
}
```

### 4. Configure Authentication

Ensure that you replace the placeholder values in the code with your actual Artifactory URL, Docker repository name, image name, and API key.

### 5. Run Your Application

You can compile and run your application using Maven:

```bash
mvn clean install
mvn exec:java -Dexec.mainClass="com.example.DockerImageSearcher"
```

### 6. Notes

- The example assumes that tags are sorted lexicographically. You may need to implement a more sophisticated method to determine the latest version based on your versioning strategy.
- Ensure that you have appropriate permissions set in Artifactory to access the Docker repository and execute the API calls.

This setup will allow you to query the Artifactory Docker registry and retrieve the latest image tag for your specified Docker image.

**Swapan**: convert to aql query

**ChatGPT**: To search for the latest Docker image in an Artifactory Docker registry using AQL (Artifactory Query Language), you can construct an AQL query that filters and sorts the results to find the most recent image tag. Below is how you can do this.

### AQL Query for Latest Docker Image

Here's an AQL query that you can use to find the latest Docker image:

```aql
items.find({
  "repo": "your-docker-repo",
  "path": {"$match": "your-image-name/*"},
  "type": "docker"
}).sort({"$desc": ["created"]}).limit(1)
```

### Explanation of the Query

- **repo**: This specifies the Docker repository you want to query.
- **path**: This is a wildcard search to match any tags under the specified image name.
- **type**: Indicates that we are searching for Docker images.
- **sort**: This sorts the results in descending order based on the `created` timestamp, ensuring the latest images are at the top.
- **limit**: This limits the results to the top 1 entry, which will be the latest image.

### Using the AQL Query in Java

You can modify your Java code to use this AQL query to find the latest Docker image tag. Here’s how you can do it:

```java
import org.apache.http.HttpResponse;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;
import com.google.gson.JsonArray;
import com.google.gson.JsonObject;
import com.google.gson.JsonParser;

public class DockerImageSearcher {
    private static final String ARTIFACTORY_URL = "https://your-artifactory-url/artifactory";
    private static final String AQL_QUERY = "items.find({\"repo\":\"your-docker-repo\",\"path\":{\"$match\":\"your-image-name/*\"},\"type\":\"docker\"}).sort({\"$desc\":[\"created\"]}).limit(1)";
    private static final String API_KEY = "your-api-key";

    public static void main(String[] args) {
        try {
            CloseableHttpClient client = HttpClients.createDefault();
            String url = ARTIFACTORY_URL + "/api/search/aql";
            HttpPost post = new HttpPost(url);
            post.setHeader("X-JFrog-Art-Api", API_KEY);
            post.setHeader("Content-Type", "text/plain");

            StringEntity entity = new StringEntity(AQL_QUERY);
            post.setEntity(entity);

            HttpResponse response = client.execute(post);
            String jsonResponse = EntityUtils.toString(response.getEntity());

            JsonArray results = JsonParser.parseString(jsonResponse).getAsJsonArray();
            if (results.size() > 0) {
                JsonObject latestImage = results.get(0).getAsJsonObject();
                String latestTag = latestImage.get("name").getAsString();
                System.out.println("Latest Tag: " + latestTag);
            } else {
                System.out.println("No images found.");
            }

            client.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### Key Changes

1. **AQL_QUERY**: This variable contains the AQL query as a string.
2. **HttpPost**: Changed from `HttpGet` to `HttpPost` since AQL queries require a POST request.
3. **Entity**: Set the AQL query as the body of the request.

### Running the Code

Ensure that you replace the placeholder values in the code with your actual Artifactory URL, Docker repository name, image name, and API key. You can then compile and run your application as described earlier.

This approach will effectively use AQL to find the latest Docker image in your specified Artifactory Docker registry.

**Swapan**: convert to spring-shell 3 and create a artifactory aql query package that is more generic

**ChatGPT**: User is working with Spring Shell 3.

**ChatGPT**: Model set context updated.

**ChatGPT**: To create a more generic Spring Shell application that queries an Artifactory AQL (Artifactory Query Language) using Spring Shell 3, you can follow these steps. This will involve setting up a Spring Boot project, defining a command to run AQL queries, and implementing a service to handle interactions with Artifactory.

### Step 1: Set Up Your Spring Boot Project

You can create a Spring Boot project with the necessary dependencies using Maven. In your `pom.xml`, include the following dependencies:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-shell</artifactId>
    </dependency>
    <dependency>
        <groupId>org.apache.httpcomponents</groupId>
        <artifactId>httpclient</artifactId>
        <version>4.5.13</version>
    </dependency>
    <dependency>
        <groupId>com.google.gson</groupId>
        <artifactId>gson</artifactId>
        <version>2.8.6</version>
    </dependency>
</dependencies>
```

### Step 2: Create the Artifactory AQL Query Service

Create a service class to handle AQL queries:

```java
package com.example.artifactory;

import org.apache.http.HttpResponse;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;
import com.google.gson.JsonArray;
import com.google.gson.JsonObject;
import com.google.gson.JsonParser;
import org.springframework.stereotype.Service;

@Service
public class ArtifactoryService {
    private final String artifactoryUrl;
    private final String apiKey;

    public ArtifactoryService(String artifactoryUrl, String apiKey) {
        this.artifactoryUrl = artifactoryUrl;
        this.apiKey = apiKey;
    }

    public JsonArray executeAqlQuery(String aqlQuery) throws Exception {
        try (CloseableHttpClient client = HttpClients.createDefault()) {
            String url = artifactoryUrl + "/api/search/aql";
            HttpPost post = new HttpPost(url);
            post.setHeader("X-JFrog-Art-Api", apiKey);
            post.setHeader("Content-Type", "text/plain");

            StringEntity entity = new StringEntity(aqlQuery);
            post.setEntity(entity);

            HttpResponse response = client.execute(post);
            String jsonResponse = EntityUtils.toString(response.getEntity());

            return JsonParser.parseString(jsonResponse).getAsJsonArray();
        }
    }
}
```

### Step 3: Create the Command for Spring Shell

Next, create a command class to interact with the Artifactory service:

```java
package com.example.artifactory;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.shell.standard.ShellComponent;
import org.springframework.shell.standard.ShellMethod;
import org.springframework.shell.standard.ShellMethodAvailability;

@ShellComponent
public class ArtifactoryCommands {
    private final ArtifactoryService artifactoryService;

    @Autowired
    public ArtifactoryCommands(ArtifactoryService artifactoryService) {
        this.artifactoryService = artifactoryService;
    }

    @ShellMethod("Get the latest Docker image tag from the specified repo and image name.")
    public String getLatestDockerImageTag(String repo, String imageName) {
        String aqlQuery = String.format("items.find({\"repo\":\"%s\",\"path\":{\"$match\":\"%s/*\"},\"type\":\"docker\"}).sort({\"$desc\":[\"created\"]}).limit(1)", repo, imageName);
        
        try {
            var results = artifactoryService.executeAqlQuery(aqlQuery);
            if (results.size() > 0) {
                JsonObject latestImage = results.get(0).getAsJsonObject();
                return "Latest Tag: " + latestImage.get("name").getAsString();
            } else {
                return "No images found.";
            }
        } catch (Exception e) {
            return "Error executing AQL query: " + e.getMessage();
        }
    }
}
```

### Step 4: Configure Application Properties

In your `application.properties`, set the Artifactory URL and API key:

```properties
artifactory.url=https://your-artifactory-url/artifactory
artifactory.api-key=your-api-key
```

### Step 5: Configure the Application

Finally, set up the Spring Boot application to read these properties:

```java
package com.example.artifactory;

import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
public class ArtifactoryAqlQueryApplication {

    public static void main(String[] args) {
        SpringApplication.run(ArtifactoryAqlQueryApplication.class, args);
    }

    @Bean
    public ArtifactoryService artifactoryService() {
        return new ArtifactoryService("your-artifactory-url/artifactory", "your-api-key");
    }
}
```

### Step 6: Running the Application

To run your application, use the following command:

```bash
mvn spring-boot:run
```

### Usage

Once your application is running, you can use the Spring Shell to execute the command:

```bash
getLatestDockerImageTag --repo your-docker-repo --imageName your-image-name
```

### Summary

This setup provides a generic way to query Docker images from an Artifactory Docker registry using AQL. You can extend the `ArtifactoryService` and `ArtifactoryCommands` classes to support more complex queries or additional features as needed.

**Swapan**: convert to command instead of shellmethod and convert only running as a console application

**ChatGPT**: To convert the Spring Shell application into a command-line console application without using Spring Shell’s `@ShellMethod`, we can implement a simple console input loop in the `main` method of the application. This involves reading input from the console, executing the AQL query, and printing the results directly.

Here’s how to set up your application accordingly:

### Step 1: Update the Artifactory Service

You can keep the `ArtifactoryService` class the same as before:

```java
package com.example.artifactory;

import org.apache.http.HttpResponse;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;
import com.google.gson.JsonArray;
import com.google.gson.JsonObject;
import com.google.gson.JsonParser;
import org.springframework.stereotype.Service;

@Service
public class ArtifactoryService {
    private final String artifactoryUrl;
    private final String apiKey;

    public ArtifactoryService(String artifactoryUrl, String apiKey) {
        this.artifactoryUrl = artifactoryUrl;
        this.apiKey = apiKey;
    }

    public JsonArray executeAqlQuery(String aqlQuery) throws Exception {
        try (CloseableHttpClient client = HttpClients.createDefault()) {
            String url = artifactoryUrl + "/api/search/aql";
            HttpPost post = new HttpPost(url);
            post.setHeader("X-JFrog-Art-Api", apiKey);
            post.setHeader("Content-Type", "text/plain");

            StringEntity entity = new StringEntity(aqlQuery);
            post.setEntity(entity);

            HttpResponse response = client.execute(post);
            String jsonResponse = EntityUtils.toString(response.getEntity());

            return JsonParser.parseString(jsonResponse).getAsJsonArray();
        }
    }
}
```

### Step 2: Modify the Main Application Class

Update the main application class to read from the console and execute the AQL query based on user input:

```java
package com.example.artifactory;

import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

import java.util.Scanner;

@SpringBootApplication
public class ArtifactoryAqlQueryApplication {

    private final ArtifactoryService artifactoryService;

    public ArtifactoryAqlQueryApplication(ArtifactoryService artifactoryService) {
        this.artifactoryService = artifactoryService;
    }

    public static void main(String[] args) {
        SpringApplication.run(ArtifactoryAqlQueryApplication.class, args);
    }

    @Bean
    public CommandLineRunner run() {
        return args -> {
            Scanner scanner = new Scanner(System.in);
            System.out.println("Enter Artifactory Repository and Image Name (format: repo imageName):");
            while (scanner.hasNextLine()) {
                String input = scanner.nextLine();
                String[] parts = input.split(" ");
                if (parts.length != 2) {
                    System.out.println("Invalid input. Please enter in format: repo imageName");
                    continue;
                }

                String repo = parts[0];
                String imageName = parts[1];

                String aqlQuery = String.format("items.find({\"repo\":\"%s\",\"path\":{\"$match\":\"%s/*\"},\"type\":\"docker\"}).sort({\"$desc\":[\"created\"]}).limit(1)", repo, imageName);

                try {
                    var results = artifactoryService.executeAqlQuery(aqlQuery);
                    if (results.size() > 0) {
                        JsonObject latestImage = results.get(0).getAsJsonObject();
                        System.out.println("Latest Tag: " + latestImage.get("name").getAsString());
                    } else {
                        System.out.println("No images found.");
                    }
                } catch (Exception e) {
                    System.out.println("Error executing AQL query: " + e.getMessage());
                }

                System.out.println("Enter another repo and image name or type 'exit' to quit:");
            }
            scanner.close();
        };
    }

    @Bean
    public ArtifactoryService artifactoryService() {
        return new ArtifactoryService("your-artifactory-url/artifactory", "your-api-key");
    }
}
```

### Step 3: Update the Application Properties

Ensure your `application.properties` is configured correctly:

```properties
artifactory.url=https://your-artifactory-url/artifactory
artifactory.api-key=your-api-key
```

### Running the Console Application

To run your application, execute:

```bash
mvn spring-boot:run
```

### Usage Instructions

1. The application will prompt you to enter the Artifactory repository and image name in the format `repo imageName`.
2. It will then execute the AQL query to fetch the latest Docker image tag and display the result.
3. You can continue entering repository and image names or type `exit` to quit the application.

### Summary

This implementation converts the Spring Shell application to a console application, allowing you to execute AQL queries interactively from the command line. You can easily extend this console application to support additional functionality as needed.

**Swapan**: use spring shell instead of the command line runner

**ChatGPT**: If you want to use Spring Shell while ensuring that the application runs as a console application, you can still take advantage of Spring Shell’s features while simplifying the setup. Here’s how to implement your application using Spring Shell, which allows you to interact with the application in a console-like environment.

### Step 1: Set Up Your Spring Boot Project

Ensure your `pom.xml` includes the necessary dependencies for Spring Boot and Spring Shell, along with the HTTP client and Gson:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-shell</artifactId>
    </dependency>
    <dependency>
        <groupId>org.apache.httpcomponents</groupId>
        <artifactId>httpclient</artifactId>
        <version>4.5.13</version>
    </dependency>
    <dependency>
        <groupId>com.google.gson</groupId>
        <artifactId>gson</artifactId>
        <version>2.8.6</version>
    </dependency>
</dependencies>
```

### Step 2: Create the Artifactory AQL Query Service

This service class will handle the AQL query execution:

```java
package com.example.artifactory;

import org.apache.http.HttpResponse;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;
import com.google.gson.JsonArray;
import com.google.gson.JsonObject;
import com.google.gson.JsonParser;
import org.springframework.stereotype.Service;

@Service
public class ArtifactoryService {
    private final String artifactoryUrl;
    private final String apiKey;

    public ArtifactoryService(String artifactoryUrl, String apiKey) {
        this.artifactoryUrl = artifactoryUrl;
        this.apiKey = apiKey;
    }

    public JsonArray executeAqlQuery(String aqlQuery) throws Exception {
        try (CloseableHttpClient client = HttpClients.createDefault()) {
            String url = artifactoryUrl + "/api/search/aql";
            HttpPost post = new HttpPost(url);
            post.setHeader("X-JFrog-Art-Api", apiKey);
            post.setHeader("Content-Type", "text/plain");

            StringEntity entity = new StringEntity(aqlQuery);
            post.setEntity(entity);

            HttpResponse response = client.execute(post);
            String jsonResponse = EntityUtils.toString(response.getEntity());

            return JsonParser.parseString(jsonResponse).getAsJsonArray();
        }
    }
}
```

### Step 3: Create the Spring Shell Commands

Now, create a command class to define your Spring Shell commands:

```java
package com.example.artifactory;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.shell.standard.ShellComponent;
import org.springframework.shell.standard.ShellMethod;
import org.springframework.shell.standard.ShellOption;

@ShellComponent
public class ArtifactoryCommands {
    private final ArtifactoryService artifactoryService;

    @Autowired
    public ArtifactoryCommands(ArtifactoryService artifactoryService) {
        this.artifactoryService = artifactoryService;
    }

    @ShellMethod("Get the latest Docker image tag from the specified repository and image name.")
    public String getLatestDockerImageTag(
            @ShellOption(help = "The repository name") String repo,
            @ShellOption(help = "The Docker image name") String imageName) {
        String aqlQuery = String.format("items.find({\"repo\":\"%s\",\"path\":{\"$match\":\"%s/*\"},\"type\":\"docker\"}).sort({\"$desc\":[\"created\"]}).limit(1)", repo, imageName);

        try {
            var results = artifactoryService.executeAqlQuery(aqlQuery);
            if (results.size() > 0) {
                JsonObject latestImage = results.get(0).getAsJsonObject();
                return "Latest Tag: " + latestImage.get("name").getAsString();
            } else {
                return "No images found.";
            }
        } catch (Exception e) {
            return "Error executing AQL query: " + e.getMessage();
        }
    }
}
```

### Step 4: Configure Application Properties

Make sure your `application.properties` is set correctly:

```properties
artifactory.url=https://your-artifactory-url/artifactory
artifactory.api-key=your-api-key
```

### Step 5: Update the Main Application Class

Your main application class should simply run the Spring Boot application:

```java
package com.example.artifactory;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ArtifactoryAqlQueryApplication {
    public static void main(String[] args) {
        SpringApplication.run(ArtifactoryAqlQueryApplication.class, args);
    }
}
```

### Running the Application

To run your application, execute:

```bash
mvn spring-boot:run
```

### Usage Instructions

Once your application is running, you can use the Spring Shell commands:

```bash
getLatestDockerImageTag --repo your-docker-repo --imageName your-image-name
```

### Summary

This implementation utilizes Spring Shell to create a console-based interaction environment where you can execute commands to query Docker images from an Artifactory registry using AQL. You have a clean separation of concerns between the command logic and the service handling HTTP requests, making the code easy to maintain and extend.

**Swapan**: use \@command instead of shellmethod

**ChatGPT**: To implement the Spring Shell application using the `@Command` annotation instead of `@ShellMethod`, you'll follow a similar structure but will change how commands are defined. The `@Command` annotation allows you to create commands that are more flexible and can have additional metadata.

Here’s how you can modify the application to use the `@Command` annotation.

### Step 1: Update Dependencies

Ensure your `pom.xml` includes the necessary dependencies:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-shell</artifactId>
    </dependency>
    <dependency>
        <groupId>org.apache.httpcomponents</groupId>
        <artifactId>httpclient</artifactId>
        <version>4.5.13</version>
    </dependency>
    <dependency>
        <groupId>com.google.gson</groupId>
        <artifactId>gson</artifactId>
        <version>2.8.6</version>
    </dependency>
</dependencies>
```

### Step 2: Create the Artifactory AQL Query Service

This service class will remain unchanged:

```java
package com.example.artifactory;

import org.apache.http.HttpResponse;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;
import com.google.gson.JsonArray;
import com.google.gson.JsonObject;
import com.google.gson.JsonParser;
import org.springframework.stereotype.Service;

@Service
public class ArtifactoryService {
    private final String artifactoryUrl;
    private final String apiKey;

    public ArtifactoryService(String artifactoryUrl, String apiKey) {
        this.artifactoryUrl = artifactoryUrl;
        this.apiKey = apiKey;
    }

    public JsonArray executeAqlQuery(String aqlQuery) throws Exception {
        try (CloseableHttpClient client = HttpClients.createDefault()) {
            String url = artifactoryUrl + "/api/search/aql";
            HttpPost post = new HttpPost(url);
            post.setHeader("X-JFrog-Art-Api", apiKey);
            post.setHeader("Content-Type", "text/plain");

            StringEntity entity = new StringEntity(aqlQuery);
            post.setEntity(entity);

            HttpResponse response = client.execute(post);
            String jsonResponse = EntityUtils.toString(response.getEntity());

            return JsonParser.parseString(jsonResponse).getAsJsonArray();
        }
    }
}
```

### Step 3: Create Spring Shell Commands with `@Command`

Now, create a command class to define your Spring Shell commands using `@Command`:

```java
package com.example.artifactory;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.shell.standard.commands.Command;
import org.springframework.shell.standard.ShellComponent;
import org.springframework.shell.standard.ShellOption;

@ShellComponent
public class ArtifactoryCommands {

    private final ArtifactoryService artifactoryService;

    @Autowired
    public ArtifactoryCommands(ArtifactoryService artifactoryService) {
        this.artifactoryService = artifactoryService;
    }

    @Command("getLatestDockerImageTag")
    public String getLatestDockerImageTag(
            @ShellOption(help = "The repository name") String repo,
            @ShellOption(help = "The Docker image name") String imageName) {
        String aqlQuery = String.format("items.find({\"repo\":\"%s\",\"path\":{\"$match\":\"%s/*\"},\"type\":\"docker\"}).sort({\"$desc\":[\"created\"]}).limit(1)", repo, imageName);

        try {
            var results = artifactoryService.executeAqlQuery(aqlQuery);
            if (results.size() > 0) {
                JsonObject latestImage = results.get(0).getAsJsonObject();
                return "Latest Tag: " + latestImage.get("name").getAsString();
            } else {
                return "No images found.";
            }
        } catch (Exception e) {
            return "Error executing AQL query: " + e.getMessage();
        }
    }
}
```

### Step 4: Configure Application Properties

Ensure your `application.properties` is set correctly:

```properties
artifactory.url=https://your-artifactory-url/artifactory
artifactory.api-key=your-api-key
```

### Step 5: Update the Main Application Class

Your main application class should remain unchanged:

```java
package com.example.artifactory;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ArtifactoryAqlQueryApplication {
    public static void main(String[] args) {
        SpringApplication.run(ArtifactoryAqlQueryApplication.class, args);
    }
}
```

### Running the Application

To run your application, execute:

```bash
mvn spring-boot:run
```

### Usage Instructions

Once your application is running, you can use the Spring Shell commands:

```bash
getLatestDockerImageTag --repo your-docker-repo --imageName your-image-name
```

### Summary

This implementation uses the `@Command` annotation in Spring Shell to define commands for querying the latest Docker image tags from an Artifactory registry. This approach maintains a clean structure, leveraging Spring Shell's capabilities for creating a user-friendly command-line interface. You can easily extend the application with additional commands as needed.

