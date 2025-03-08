**Swapan**: pull and push to artifactory docker registry with jib maven

**ChatGPT**: Using Jib, a tool for building container images in Java, you can configure your Maven project to pull and push Docker images to an Artifactory Docker registry without requiring Docker to be installed on your system. Below are the steps to achieve this:

---

### **1. Add Jib Plugin to Your Maven Project**
Include the Jib Maven plugin in your `pom.xml`:

```xml
<plugin>
    <groupId>com.google.cloud.tools</groupId>
    <artifactId>jib-maven-plugin</artifactId>
    <version>3.4.0</version> <!-- Use the latest version -->
</plugin>
```

---

### **2. Configure Jib for Artifactory**
Update your `pom.xml` with configuration details for your Artifactory Docker registry.

```xml
<plugin>
    <groupId>com.google.cloud.tools</groupId>
    <artifactId>jib-maven-plugin</artifactId>
    <version>3.4.0</version>
    <configuration>
        <!-- Your Artifactory Docker registry details -->
        <to>
            <image>your-artifactory-registry.com/repository/image-name:tag</image>
            <auth>
                <username>${docker.username}</username>
                <password>${docker.password}</password>
            </auth>
        </to>
        <from>
            <image>base-image:tag</image> <!-- Example: openjdk:17 -->
        </from>
    </configuration>
</plugin>
```

---

### **3. Secure Authentication with a `settings.xml`**
Instead of hardcoding credentials, use Maven's `settings.xml` to manage sensitive information:

1. Locate the `settings.xml` file (usually in `~/.m2/settings.xml`).

2. Add your Artifactory credentials:

```xml
<servers>
    <server>
        <id>artifactory-docker</id>
        <username>your-username</username>
        <password>your-password</password>
    </server>
</servers>
```

3. Reference the server ID in the Jib plugin configuration:

```xml
<auth>
    <username>${env.DOCKER_USERNAME}</username>
    <password>${env.DOCKER_PASSWORD}</password>
</auth>
```

---

### **4. Pull the Base Image**
To specify a custom base image (e.g., from Artifactory):

```xml
<from>
    <image>your-artifactory-registry.com/repository/base-image:tag</image>
    <auth>
        <username>${docker.username}</username>
        <password>${docker.password}</password>
    </auth>
</from>
```

If using a public image, you can omit the `auth` block for the `from` configuration.

---

### **5. Build and Push the Image**
Use Maven to build and push the image to your Artifactory Docker registry:

- **Build only:**  
  ```bash
  mvn compile jib:buildTar
  ```

  This will create a `tar` file of the Docker image in your `target` directory.

- **Push to registry:**  
  ```bash
  mvn compile jib:build
  ```

  This command pushes the image directly to your Artifactory Docker registry.

---

### **6. Testing the Image**
After pushing, you can pull the image from your Artifactory Docker registry to test it:

```bash
docker pull your-artifactory-registry.com/repository/image-name:tag
docker run -p 8080:8080 your-artifactory-registry.com/repository/image-name:tag
```

---

### **Optional: Using Profiles for Multi-Environment Configurations**
To support multiple environments, you can define Maven profiles in your `pom.xml`:

```xml
<profiles>
    <profile>
        <id>staging</id>
        <properties>
            <jib.to.image>staging-artifactory-registry.com/repository/image-name:tag</jib.to.image>
        </properties>
    </profile>
    <profile>
        <id>production</id>
        <properties>
            <jib.to.image>production-artifactory-registry.com/repository/image-name:tag</jib.to.image>
        </properties>
    </profile>
</profiles>
```

Run with a specific profile:

```bash
mvn compile jib:build -Pstaging
```

---

This setup allows seamless integration of Jib with your Artifactory Docker registry for pulling and pushing container images.

**Swapan**: show a complete pom.xml

**ChatGPT**: Here is an example of a complete `pom.xml` configured for a Java application using Jib to push Docker images to an Artifactory Docker registry:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>my-app</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <name>MyApp</name>
    <description>Example Java application with Jib and Artifactory</description>

    <properties>
        <java.version>17</java.version>
        <docker.username>my-docker-username</docker.username>
        <docker.password>my-docker-password</docker.password>
    </properties>

    <dependencies>
        <!-- Add your dependencies here -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
            <version>3.1.4</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Jib Plugin -->
            <plugin>
                <groupId>com.google.cloud.tools</groupId>
                <artifactId>jib-maven-plugin</artifactId>
                <version>3.4.0</version>
                <configuration>
                    <from>
                        <image>openjdk:17-jdk-slim</image>
                    </from>
                    <to>
                        <image>artifactory.example.com/docker-repo/my-app:1.0.0</image>
                        <auth>
                            <username>${docker.username}</username>
                            <password>${docker.password}</password>
                        </auth>
                    </to>
                    <container>
                        <mainClass>com.example.MyApp</mainClass>
                        <ports>
                            <port>8080</port>
                        </ports>
                        <creationTime>USE_CURRENT_TIMESTAMP</creationTime>
                    </container>
                </configuration>
            </plugin>
        </plugins>
    </build>

    <profiles>
        <profile>
            <id>staging</id>
            <properties>
                <docker.username>staging-user</docker.username>
                <docker.password>staging-pass</docker.password>
            </properties>
        </profile>
        <profile>
            <id>production</id>
            <properties>
                <docker.username>production-user</docker.username>
                <docker.password>production-pass</docker.password>
            </properties>
        </profile>
    </profiles>
</project>
```

---

### Key Sections:

1. **Properties:**
   - `docker.username` and `docker.password` are placeholders for your Artifactory Docker registry credentials. You can use environment variables or Maven's `settings.xml` for better security.

2. **Dependencies:**
   - Add dependencies like `spring-boot-starter` as needed for your project.

3. **Jib Configuration:**
   - `from`: The base image (e.g., `openjdk:17-jdk-slim`).
   - `to`: The target image location in your Artifactory Docker registry.
   - `auth`: Configures authentication for pushing the image.

4. **Profiles:**
   - Define `staging` and `production` profiles to manage environment-specific credentials.

5. **Main Class:**
   - Replace `com.example.MyApp` with your application's main class.

---

### Build and Push
Run these commands to build and push your Docker image:

- **For default settings:**
  ```bash
  mvn compile jib:build
  ```

- **For a specific profile (e.g., staging):**
  ```bash
  mvn compile jib:build -Pstaging
  ```

**Swapan**: how to do image versioning?

**ChatGPT**: Image versioning is a crucial practice for tracking and managing different versions of your container images, ensuring stability and compatibility across environments. Here’s how you can implement image versioning effectively with Jib and Maven:

---

### **1. Use Maven Project Version**
The simplest way is to use the Maven project version for the Docker image tag.

#### Update the Jib Configuration:
In the `pom.xml` file, reference the `${project.version}` property in the `to.image` tag:

```xml
<configuration>
    <to>
        <image>artifactory.example.com/docker-repo/my-app:${project.version}</image>
    </to>
</configuration>
```

#### Update Maven Version:
Update the version in your `pom.xml` manually or using the Maven Versions Plugin:

- **Set a new version:**
  ```bash
  mvn versions:set -DnewVersion=1.0.1
  ```

- **Commit the change:**
  ```bash
  git commit -am "Update version to 1.0.1"
  ```

---

### **2. Use Git Commit Hash or Tag**
Incorporate a Git commit hash or tag as part of the version.

#### Generate the Commit Hash Dynamically:
Use the `git-commit-id-plugin` to fetch the Git commit hash:

Add this plugin to your `pom.xml`:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>pl.project13.maven</groupId>
            <artifactId>git-commit-id-plugin</artifactId>
            <version>5.0.0</version>
        </plugin>
    </plugins>
</build>
```

Then, update the Jib configuration to use the hash:

```xml
<configuration>
    <to>
        <image>artifactory.example.com/docker-repo/my-app:${project.version}-${git.commit.id.abbrev}</image>
    </to>
</configuration>
```

When you build the project, the image tag will include the project version and the abbreviated Git commit hash, e.g., `1.0.0-ab123cd`.

---

### **3. Use Timestamps**
Add a timestamp to the image version for better traceability.

#### Add Timestamp to Jib Configuration:
Use Maven's `build-helper-maven-plugin` to generate a timestamp:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>build-helper-maven-plugin</artifactId>
            <version>3.3.0</version>
            <executions>
                <execution>
                    <id>add-timestamp</id>
                    <phase>validate</phase>
                    <goals>
                        <goal>timestamp-property</goal>
                    </goals>
                    <configuration>
                        <name>timestamp</name>
                        <pattern>yyyyMMddHHmmss</pattern>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

Then, update the Jib configuration:

```xml
<configuration>
    <to>
        <image>artifactory.example.com/docker-repo/my-app:${project.version}-${timestamp}</image>
    </to>
</configuration>
```

---

### **4. Multi-Environment Versioning**
Define environment-specific tags by combining profiles and versioning strategies.

#### Example Profiles in `pom.xml`:
```xml
<profiles>
    <profile>
        <id>dev</id>
        <properties>
            <image.tag>${project.version}-dev</image.tag>
        </properties>
    </profile>
    <profile>
        <id>staging</id>
        <properties>
            <image.tag>${project.version}-staging</image.tag>
        </properties>
    </profile>
    <profile>
        <id>production</id>
        <properties>
            <image.tag>${project.version}</image.tag>
        </properties>
    </profile>
</profiles>
```

#### Update Jib Configuration:
Reference the `image.tag` property:

```xml
<configuration>
    <to>
        <image>artifactory.example.com/docker-repo/my-app:${image.tag}</image>
    </to>
</configuration>
```

#### Build for Specific Environments:
Run Maven with the desired profile:

- Development:
  ```bash
  mvn compile jib:build -Pdev
  ```

- Staging:
  ```bash
  mvn compile jib:build -Pstaging
  ```

- Production:
  ```bash
  mvn compile jib:build -Pproduction
  ```

---

### **5. Use CI/CD Pipelines for Dynamic Tags**
Leverage your CI/CD pipeline to dynamically tag images:

- Use environment variables for branch names, build numbers, or tags.
- Pass these variables to Maven during the build:

  ```bash
  mvn compile jib:build -Dimage.tag=1.0.0-build123
  ```

  Update Jib configuration:

  ```xml
  <configuration>
      <to>
          <image>artifactory.example.com/docker-repo/my-app:${image.tag}</image>
      </to>
  </configuration>
  ```

---

### Example Final Jib Configuration
Here’s a complete example using a combination of versioning strategies:

```xml
<configuration>
    <from>
        <image>openjdk:17-jdk-slim</image>
    </from>
    <to>
        <image>artifactory.example.com/docker-repo/my-app:${project.version}-${git.commit.id.abbrev}-${timestamp}</image>
        <auth>
            <username>${docker.username}</username>
            <password>${docker.password}</password>
        </auth>
    </to>
    <container>
        <mainClass>com.example.MyApp</mainClass>
        <ports>
            <port>8080</port>
        </ports>
        <creationTime>USE_CURRENT_TIMESTAMP</creationTime>
    </container>
</configuration>
```

With this configuration, your image version might look like this:
`1.0.0-ab123cd-20231203104500`.

This approach ensures traceability and clarity in versioning across environments and builds.

