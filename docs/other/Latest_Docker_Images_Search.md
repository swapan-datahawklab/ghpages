**Swapan**: artifactory search for latest version of all docker images in one repo

**ChatGPT**: **Dusting off specifics**

OK, let me see. The task involves finding the latest version of all Docker images in a specific Artifactory repository.

**Evaluating options**

I’m working through various methods to fetch Docker images from Artifactory, including the UI, REST API, Artifactory Query Language (AQL), and JFrog CLI. AQL seems promising for querying specific repositories and the latest image versions.

**Crafting an AQL query**

I’m mapping out an AQL query to group Docker artifacts by properties like 'docker.manifest' and sort them by date. This might help in identifying the latest Docker image versions in the 'docker-local' repository.

**Navigating Artifactory**

I’m exploring Docker image storage in Artifactory and considering the Artifactory REST API and JFrog CLI. Grouping by image name for the latest version is intriguing.

**Examining options**

I’m digging into how to locate the latest Docker image tags in Artifactory. Considering using AQL queries, Docker API, JFrog CLI, or an Artifactory REST API.

**Pulling image details**

I’m gathering info on using Artifactory's Docker Registry HTTP API. I craft URL commands to list images and tags. Authentication might be needed, and some endpoints may be off by default.

**Listing Docker images**

I’m detailing how to list Docker images and their tags in Artifactory using REST API calls, considering methods to identify the latest tag by sorting or examining creation dates.

**Listing Docker images**

I’m identifying all Docker images in Artifactory using the REST API and their tags to pinpoint the latest version.

**Listing and identifying**

I'm getting the image names from the Docker repository's catalog using an API call. Then, I’m retrieving tags for each image to pinpoint the latest version, using sorting or creation date to inform the decision.

**ChatGPT**: To search for the latest version of all Docker images in a single repository within JFrog Artifactory, you can utilize Artifactory's REST API, specifically the Docker V2 API endpoints. Below, I'll guide you through the process step-by-step, including example commands and explanations.

---

### **Step 1: Retrieve All Docker Image Names in the Repository**

Use the `_catalog` endpoint to get a list of all Docker images (repositories) within your specified Docker repository.

**API Endpoint:**

```
GET /api/docker/{repo-key}/v2/_catalog
```

**Example Command:**

```bash
curl -u username:password "https://artifactory.example.com/artifactory/api/docker/my-docker-repo/v2/_catalog"
```

**Response Format:**

```json
{
  "repositories": [
    "image1",
    "image2",
    "image3"
  ]
}
```

---

### **Step 2: Retrieve Tags for Each Docker Image**

For each image retrieved in Step 1, fetch the list of tags using the `tags/list` endpoint.

**API Endpoint:**

```
GET /api/docker/{repo-key}/v2/{image-name}/tags/list
```

**Example Command:**

```bash
curl -u username:password "https://artifactory.example.com/artifactory/api/docker/my-docker-repo/v2/image1/tags/list"
```

**Response Format:**

```json
{
  "name": "image1",
  "tags": [
    "1.0.0",
    "1.1.0",
    "2.0.0"
  ]
}
```

---

### **Step 3: Determine the Latest Tag for Each Image**

To find the latest version:

- **Option A: Sort Tags Semantically**

  If your tags follow semantic versioning (e.g., `1.0.0`, `1.1.0`, `2.0.0`), you can sort them using semantic versioning rules.

  **Example in Bash with `sort`:**

  ```bash
  latest_tag=$(echo -e "1.0.0\n1.1.0\n2.0.0" | sort -V | tail -n 1)
  echo $latest_tag  # Outputs: 2.0.0
  ```

- **Option B: Use the Manifest to Get the Creation Date**

  If tags are not semantically versioned or you prefer to use the creation date, fetch the manifest for each tag to obtain its creation timestamp.

  **API Endpoint:**

  ```
  GET /v2/{image-name}/manifests/{tag}
  ```

  **Note:** You'll need to adjust the URL and headers to accept the manifest in version 2 schema 2 format.

  **Example Command:**

  ```bash
  curl -u username:password \
       -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
       "https://artifactory.example.com/artifactory/api/docker/my-docker-repo/v2/image1/manifests/1.0.0"
  ```

  **Extract the `created` field from the response and compare dates to find the latest tag.**

---

### **Step 4: Automate the Process with a Script**

Below is an example of how you might automate these steps using a Bash script:

```bash
#!/bin/bash

ARTIFACTORY_URL="https://artifactory.example.com/artifactory"
REPO_KEY="my-docker-repo"
AUTH="username:password"

# Step 1: Get all images
images=$(curl -su $AUTH "$ARTIFACTORY_URL/api/docker/$REPO_KEY/v2/_catalog" | jq -r '.repositories[]')

# Initialize an associative array to hold the latest tags
declare -A latest_tags

# Step 2 & 3: For each image, get tags and determine the latest
for image in $images; do
  tags=$(curl -su $AUTH "$ARTIFACTORY_URL/api/docker/$REPO_KEY/v2/$image/tags/list" | jq -r '.tags[]')
  
  # Option A: Sort tags semantically
  latest_tag=$(echo "$tags" | sort -V | tail -n 1)
  
  # Store the latest tag
  latest_tags["$image"]="$latest_tag"
done

# Output the latest tags for each image
for image in "${!latest_tags[@]}"; do
  echo "Image: $image, Latest Tag: ${latest_tags[$image]}"
done
```

**Note:**

- Make sure you have `jq` installed for JSON parsing.
- Replace `username:password` with your actual Artifactory credentials.
- Ensure that your tags can be sorted semantically for Option A.

---

### **Alternative Method: Using Artifactory Query Language (AQL)**

While AQL doesn't support grouping and aggregation like SQL, you can still craft queries to find artifacts based on properties such as creation date.

**Example AQL Query:**

```json
items.find({
  "repo": "my-docker-repo",
  "type": "file",
  "name": {"$match": "manifest.json"}
}).include("name", "path", "created").sort({"$desc": ["created"]})
```

**Usage:**

- Run this query via the Artifactory REST API:

  ```bash
  curl -u username:password -X POST \
       -H "Content-Type: text/plain" \
       -d @aql_query.txt \
       "https://artifactory.example.com/artifactory/api/search/aql"
  ```

- **Note:** Save your AQL query in a file named `aql_query.txt`.

**Limitations:**

- AQL may not directly provide the "latest" version per image without additional processing.
- You might need to parse and process the results programmatically to determine the latest versions.

---

### **Considerations**

- **Authentication:** Ensure that the user has the necessary permissions to access the Docker repository and execute API calls.
- **API Rate Limits:** Be mindful of any rate limits imposed by Artifactory when scripting API calls.
- **Non-Semantic Tags:** If your tags are dates or commit hashes, adjust the sorting mechanism accordingly.
- **Security:** Avoid hardcoding credentials in scripts. Use environment variables or prompt for input.

---

### **Conclusion**

By leveraging Artifactory's REST API and some scripting, you can effectively retrieve the latest versions of all Docker images in a repository. This method provides flexibility and can be integrated into automation pipelines or monitoring tools as needed.

If you need further assistance or have questions about specific steps, feel free to ask!

