**Swapan**: aql query on docker image to get artifactory properties

**ChatGPT**: **Navigating Artifactory**

I’m working through how to use AQL to extract Docker image properties stored in JFrog Artifactory, emphasizing the query's construction and REST API integration.

**ChatGPT**: To retrieve properties of Docker images stored in JFrog Artifactory using the Artifactory Query Language (AQL), you can construct and execute an AQL query tailored to Docker repositories. Below is a step-by-step guide on how to achieve this, including example queries and explanations.

## Understanding AQL for Docker Images

Artifactory stores Docker images as a collection of artifacts, including layers, manifests, and configuration files. Properties can be associated with these artifacts to store metadata such as version information, environment tags, or custom attributes.

AQL allows you to query these artifacts and their associated properties efficiently. To target Docker images specifically, you typically query within Docker repositories and filter based on Docker-specific attributes.

## Constructing the AQL Query

### Basic Structure

An AQL query generally consists of the following components:

1. **Items**: Defines the scope of the search (e.g., within specific repositories).
2. **Properties**: Filters based on artifact properties.
3. **Fields**: Specifies which fields to return in the results.

### Example Scenario

Suppose you have a Docker repository named `docker-local`, and you want to retrieve properties of Docker images named `my-app` with a specific tag.

### Sample AQL Query

Here's an example AQL query to fetch properties of a Docker image:

```json
items.find({
  "repo": "docker-local",
  "path": "my-app",
  "name": "my-app",
  "type": "file",
  "properties": {
    "$and": [
      {
        "key": "yourPropertyKey",
        "value": "yourPropertyValue"
      }
    ]
  }
}).include("name", "repo", "path", "properties")
```

### Explanation of the Query Components

- **repo**: Specifies the repository to search in (`docker-local`).
- **path**: The path within the repository where the Docker image resides (`my-app`).
- **name**: The name of the Docker image artifact (`my-app`).
- **type**: The type of artifact (`file` is typical for Docker layers and manifests).
- **properties**: Filters artifacts that have specific properties. Replace `yourPropertyKey` and `yourPropertyValue` with your actual property key and value.
- **include**: Specifies which fields to include in the response (`name`, `repo`, `path`, and `properties`).

### Retrieving All Properties for a Docker Image

If you want to retrieve all properties associated with a specific Docker image without filtering by property key/value, you can simplify the query:

```json
items.find({
  "repo": "docker-local",
  "path": "my-app",
  "name": "my-app",
  "type": "file"
}).include("name", "repo", "path", "properties")
```

### Filtering by Docker Image Tag

To filter Docker images by a specific tag, you can include the `docker.manifest.name` and `docker.manifest.tag` properties:

```json
items.find({
  "repo": "docker-local",
  "type": "file",
  "properties": {
    "$and": [
      {
        "key": "docker.manifest.name",
        "value": "my-app"
      },
      {
        "key": "docker.manifest.tag",
        "value": "v1.0.0"
      }
    ]
  }
}).include("name", "repo", "path", "properties")
```

### Executing the AQL Query

You can execute the AQL query using Artifactory's REST API. Here's how you can do it using `curl`:

```bash
curl -u your-username:your-password -X POST "https://your-artifactory-instance/artifactory/api/search/aql" -H "Content-Type: text/plain" -d '
items.find({
  "repo": "docker-local",
  "path": "my-app",
  "name": "my-app",
  "type": "file",
  "properties": {
    "$and": [
      {
        "key": "yourPropertyKey",
        "value": "yourPropertyValue"
      }
    ]
  }
}).include("name", "repo", "path", "properties")
'
```

**Replace the following placeholders:**
- `your-username`: Your Artifactory username.
- `your-password`: Your Artifactory password or API key.
- `https://your-artifactory-instance`: The URL of your Artifactory server.
- Adjust the JSON payload as per your specific query requirements.

### Parsing the Response

The response will be a JSON object containing the list of artifacts that match the query, along with their specified fields and properties. You can parse this JSON to extract and utilize the properties as needed.

## Additional Tips

- **Property Operators**: AQL supports various operators like `$eq`, `$match`, `$and`, `$or`, etc., allowing complex queries based on multiple property conditions.
  
- **Performance Considerations**: For large repositories, ensure your queries are as specific as possible to reduce execution time and resource usage.

- **Testing Queries**: Use the Artifactory AQL Editor available in the Artifactory UI to test and refine your queries before integrating them into scripts or applications.

- **Documentation**: Refer to the [JFrog Artifactory AQL Documentation](https://www.jfrog.com/confluence/display/JFROG/Artifactory+Query+Language) for more detailed information on crafting advanced queries.

## Example: Retrieving All Properties for All Docker Images in a Repository

If you need to retrieve properties for all Docker images within the `docker-local` repository, you can use the following AQL:

```json
items.find({
  "repo": "docker-local",
  "type": "file",
  "property": {}
}).include("name", "repo", "path", "properties")
```

This query fetches all file-type artifacts in the `docker-local` repository along with their properties.

## Conclusion

Using AQL to query Docker image properties in Artifactory provides a powerful way to manage and retrieve metadata associated with your Docker artifacts. By constructing precise AQL queries, you can efficiently extract the information you need for automation, reporting, or integration with other systems.

If you have specific requirements or encounter issues, feel free to provide more details, and I can assist further!

**Swapan**: how to use .include with .sort

**ChatGPT**: **Clarifying query components**

I'm working through how to use .include and .sort clauses in an AQL query. This involves retrieving specific fields and sorting results by field names and direction, likely for Docker image properties.

**ChatGPT**: Certainly! Combining the `.include` and `.sort` clauses in your Artifactory Query Language (AQL) queries allows you to both specify which fields to retrieve and define the order in which the results are returned. This can be particularly useful when working with Docker images in Artifactory, enabling you to efficiently organize and extract the desired information.

Below, I'll walk you through how to use `.include` with `.sort` in AQL, providing detailed explanations and example queries to illustrate their usage.

## Understanding `.include` and `.sort` in AQL

### `.include`
The `.include` clause specifies which fields you want to retrieve in the query results. By default, AQL returns a comprehensive set of fields, but using `.include` can optimize performance by fetching only the necessary data.

**Common Fields to Include:**
- `name`: The name of the artifact.
- `repo`: The repository where the artifact is stored.
- `path`: The path within the repository.
- `created`: The creation date of the artifact.
- `modified`: The last modified date.
- `properties`: The properties associated with the artifact.

### `.sort`
The `.sort` clause defines the order in which the query results are returned. You can sort the results based on one or more fields, in either ascending (`asc`) or descending (`desc`) order.

**Common Sort Fields:**
- `created`: Sort by creation date.
- `modified`: Sort by last modified date.
- `name`: Sort alphabetically by artifact name.
- `size`: Sort by artifact size.

## Combining `.include` with `.sort`

When combining `.include` with `.sort`, the typical structure of an AQL query is as follows:

```aql
items.find({
  /* Your search criteria */
})
.include("field1", "field2", "field3")
.sort({
  "fieldToSortBy": "asc" | "desc",
  /* Additional sort fields if needed */
})
```

### Key Points:
- **Order Matters:** The `.include` and `.sort` clauses should be placed after the `find` block.
- **Multiple Sorts:** You can sort by multiple fields by adding more key-value pairs within the `.sort` block.
- **Ascending vs. Descending:** Specify `asc` for ascending order and `desc` for descending order.

## Example Scenarios

### 1. Retrieve Specific Fields and Sort by Creation Date

Suppose you want to fetch the `name`, `repo`, `path`, and `properties` of Docker images in the `docker-local` repository, sorted by their creation date in descending order (newest first).

```aql
items.find({
  "repo": "docker-local",
  "type": "file",
  "path": "my-app"
})
.include("name", "repo", "path", "properties")
.sort({
  "created": "desc"
})
```

**Explanation:**
- **Search Criteria (`find`):**
  - `repo`: Targets the `docker-local` repository.
  - `type`: Specifies that we're looking for file-type artifacts (typical for Docker images).
  - `path`: Narrows down to the `my-app` path within the repository.
- **Fields to Include (`include`):**
  - Retrieves only the `name`, `repo`, `path`, and `properties` fields.
- **Sorting (`sort`):**
  - Sorts the results by the `created` field in descending order, showing the most recently created images first.

### 2. Retrieve All Properties and Sort by Name

If you want to retrieve all properties of Docker images in the `docker-local` repository and sort the results alphabetically by the artifact name:

```aql
items.find({
  "repo": "docker-local",
  "type": "file",
  "path": "my-app"
})
.include("name", "repo", "path", "properties")
.sort({
  "name": "asc"
})
```

**Explanation:**
- **Search Criteria (`find`):** Same as the previous example.
- **Fields to Include (`include`):** Same as the previous example.
- **Sorting (`sort`):**
  - Sorts the results by the `name` field in ascending order (A to Z).

### 3. Retrieve Specific Properties and Sort by Custom Property

Assume you have a custom property named `version` associated with your Docker images and you want to sort the results based on this property.

```aql
items.find({
  "repo": "docker-local",
  "type": "file",
  "path": "my-app",
  "property": {
    "version": {"$match": "*"}
  }
})
.include("name", "repo", "path", "properties")
.sort({
  "properties.version": "asc"
})
```

**Explanation:**
- **Search Criteria (`find`):**
  - In addition to previous criteria, it filters artifacts that have a `version` property.
- **Fields to Include (`include`):** Same as previous examples.
- **Sorting (`sort`):**
  - Sorts the results based on the `version` property in ascending order.
  - **Note:** Sorting by a property requires using the `properties.` prefix followed by the property key.

## Executing the AQL Query via Artifactory's REST API

To execute an AQL query that includes both `.include` and `.sort`, you can use the Artifactory REST API. Here's how you can do it using `curl`:

### Example: Retrieve Specific Fields Sorted by Creation Date

```bash
curl -u your-username:your-password -X POST "https://your-artifactory-instance/artifactory/api/search/aql" -H "Content-Type: text/plain" -d '
items.find({
  "repo": "docker-local",
  "type": "file",
  "path": "my-app"
})
.include("name", "repo", "path", "properties")
.sort({
  "created": "desc"
})
'
```

**Replace the following placeholders:**
- `your-username`: Your Artifactory username.
- `your-password`: Your Artifactory password or API key.
- `https://your-artifactory-instance`: The base URL of your Artifactory server.

### Example: Retrieve Specific Properties Sorted by Custom Property

```bash
curl -u your-username:your-password -X POST "https://your-artifactory-instance/artifactory/api/search/aql" -H "Content-Type: text/plain" -d '
items.find({
  "repo": "docker-local",
  "type": "file",
  "path": "my-app",
  "property": {
    "version": {"$match": "*"}
  }
})
.include("name", "repo", "path", "properties")
.sort({
  "properties.version": "asc"
})
'
```

**Notes:**
- **Authentication:** It's recommended to use API keys or access tokens instead of plain passwords for better security.
- **Content-Type:** Ensure the `Content-Type` header is set to `text/plain` when sending the AQL query in the request body.
- **Formatting:** The AQL query is enclosed within single quotes (`'`) to preserve the JSON structure.

## Parsing the Response

The response from the AQL query will be a JSON object containing the list of artifacts that match the query, along with the specified fields. Here's an example of what the response might look like:

```json
{
  "results": [
    {
      "repo": "docker-local",
      "path": "my-app",
      "name": "my-app-1.0.0.tar",
      "properties": [
        {
          "key": "version",
          "value": "1.0.0"
        },
        {
          "key": "docker.manifest.name",
          "value": "my-app"
        },
        {
          "key": "docker.manifest.tag",
          "value": "v1.0.0"
        }
      ]
    },
    {
      "repo": "docker-local",
      "path": "my-app",
      "name": "my-app-1.0.1.tar",
      "properties": [
        {
          "key": "version",
          "value": "1.0.1"
        },
        {
          "key": "docker.manifest.name",
          "value": "my-app"
        },
        {
          "key": "docker.manifest.tag",
          "value": "v1.0.1"
        }
      ]
    }
    // More results...
  ],
  "range": {
    "start_pos": 0,
    "end_pos": 1,
    "total": 2
  }
}
```

**Key Components:**
- **results:** An array of artifacts matching the query.
- **repo, path, name:** The included fields for each artifact.
- **properties:** An array of properties associated with each artifact.
- **range:** Information about the range of results returned (useful for pagination).

You can parse this JSON response using your preferred programming language or tool to extract and utilize the information as needed.

## Additional Tips

1. **Testing Your Queries:**
   - Use the [JFrog Artifactory AQL Editor](https://www.jfrog.com/confluence/display/JFROG/Artifactory+Query+Language) available in the Artifactory UI to test and refine your queries before integrating them into scripts or applications.

2. **Optimizing Performance:**
   - **Be Specific:** The more specific your `find` criteria, the faster the query will execute.
   - **Limit Results:** Use the `.limit` clause to restrict the number of results returned, which can enhance performance, especially with large datasets.

3. **Handling Large Results:**
   - **Pagination:** Use `.offset` and `.limit` to paginate through large result sets.
   - **Streaming:** For very large queries, consider processing results in a streamed fashion to avoid memory issues.

4. **Advanced Sorting:**
   - **Multiple Sort Fields:** You can sort by multiple fields by adding additional key-value pairs in the `.sort` clause.

   **Example: Sort by `version` ascending, then by `created` descending**
   ```aql
   .sort({
     "properties.version": "asc",
     "created": "desc"
   })
   ```

5. **Combining with Other Clauses:**
   - **Filters:** Combine `.include` and `.sort` with other AQL clauses like `.group`, `.limit`, and `.offset` to create more complex queries.

6. **Security Considerations:**
   - **Use API Keys or Tokens:** Instead of embedding plain passwords in your scripts or commands, use API keys or access tokens for authentication.
   - **Least Privilege:** Ensure the user or API key used has the minimum required permissions to execute the queries.

## Complete Example: Retrieve and Sort Docker Image Properties

Let's combine everything into a comprehensive example.

### Scenario:
- **Repository:** `docker-local`
- **Docker Image Name:** `my-app`
- **Retrieve Fields:** `name`, `repo`, `path`, `properties`
- **Sort By:** `docker.manifest.tag` in ascending order

### AQL Query:

```aql
items.find({
  "repo": "docker-local",
  "type": "file",
  "path": "my-app",
  "property": {
    "docker.manifest.name": "my-app"
  }
})
.include("name", "repo", "path", "properties")
.sort({
  "properties.docker.manifest.tag": "asc"
})
```

### Executing via `curl`:

```bash
curl -u your-username:your-password -X POST "https://your-artifactory-instance/artifactory/api/search/aql" -H "Content-Type: text/plain" -d '
items.find({
  "repo": "docker-local",
  "type": "file",
  "path": "my-app",
  "property": {
    "docker.manifest.name": "my-app"
  }
})
.include("name", "repo", "path", "properties")
.sort({
  "properties.docker.manifest.tag": "asc"
})
'
```

### Expected Response:

```json
{
  "results": [
    {
      "repo": "docker-local",
      "path": "my-app",
      "name": "my-app-1.0.0.tar",
      "properties": [
        {
          "key": "docker.manifest.name",
          "value": "my-app"
        },
        {
          "key": "docker.manifest.tag",
          "value": "v1.0.0"
        }
        // Additional properties...
      ]
    },
    {
      "repo": "docker-local",
      "path": "my-app",
      "name": "my-app-1.0.1.tar",
      "properties": [
        {
          "key": "docker.manifest.name",
          "value": "my-app"
        },
        {
          "key": "docker.manifest.tag",
          "value": "v1.0.1"
        }
        // Additional properties...
      ]
    }
    // More results...
  ],
  "range": {
    "start_pos": 0,
    "end_pos": 1,
    "total": 2
  }
}
```

**Notes:**
- The results are sorted by the `docker.manifest.tag` property in ascending order (`v1.0.0` before `v1.0.1`).
- Only the specified fields (`name`, `repo`, `path`, `properties`) are included in the response.

## Conclusion

By effectively utilizing the `.include` and `.sort` clauses in your AQL queries, you can tailor your searches in Artifactory to retrieve precisely the information you need in an organized manner. This combination enhances both the performance and readability of your queries, making it easier to manage and analyze your Docker images and their associated properties.

If you have any further questions or need assistance with more complex queries, feel free to ask!

