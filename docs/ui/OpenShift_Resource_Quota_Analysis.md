**Swapan**: openshift oc command to calcuate resource quotas used vs set

**ChatGPT**: To calculate the resource quotas used versus set in OpenShift, you can use the `oc describe` or `oc get` command along with some custom processing. Here’s how you can approach this:

1. **Using `oc describe`**:
   This command will give you detailed output of the resource quota including both used and hard (set) limits.
   ```bash
   oc describe resourcequota
   ```

   This output will show each quota item (e.g., CPU, memory, pods) with the used and hard limits.

2. **Using `oc get` for more concise output**:
   You can also use `oc get` with custom output formatting to show used and hard values in a table format.
   ```bash
   oc get resourcequota -o custom-columns=NAME:.metadata.name,USED:.status.used,HARD:.status.hard
   ```

3. **Calculating usage percentage**:
   If you want to script the calculation to show usage as a percentage, you can use `jq` or another parsing tool to manipulate JSON output:
   ```bash
   oc get resourcequota -o json | jq '.items[] | {name: .metadata.name, usage: .status.used, hard: .status.hard}'
   ```

   You can further process the output using a script in `bash`, `awk`, or other tools to calculate the percentage of each resource used versus the set limit.

Would you like help writing a full script to automate this analysis?

**Swapan**: yes, please provide a full script?

**ChatGPT**: Here's a Bash script using `oc` and `jq` to calculate the percentage of resource usage against the set quotas for each type:

```bash
#!/bin/bash

# Ensure jq is installed
if ! command -v jq &> /dev/null; then
    echo "jq is required but not installed. Please install jq."
    exit 1
fi

# Get resource quota JSON data
quota_json=$(oc get resourcequota -o json)

# Check if the resource quota data was fetched successfully
if [[ -z "$quota_json" ]]; then
    echo "Failed to fetch resource quota data. Ensure you're connected to the OpenShift cluster."
    exit 1
fi

# Process and display the data
echo "NAME         RESOURCE         USED         HARD         USAGE(%)"
echo "---------------------------------------------------------------"

echo "$quota_json" | jq -r '
.items[] | .metadata.name as $name |
.status.used as $used |
.status.hard as $hard |
($used | keys[]) as $resource |
$name + " " + $resource + " " + ($used[$resource] // "0") + " " + ($hard[$resource] // "N/A") + " " +
(if ($used[$resource] | tonumber // 0) > 0 and ($hard[$resource] | tonumber // 0) > 0 then
  ((($used[$resource] | tonumber) / ($hard[$resource] | tonumber)) * 100 | tostring) + "%"
else
  "N/A"
end)
' | column -t
```

### How the Script Works:
1. It fetches resource quota data using `oc get resourcequota -o json`.
2. `jq` processes the JSON output, extracting the `used` and `hard` values for each resource.
3. It calculates the usage percentage only if both `used` and `hard` values are numeric and greater than zero.
4. The output is formatted for better readability using `column`.

### Usage:
1. Save this script to a file, e.g., `calculate_quota_usage.sh`.
2. Make it executable:
   ```bash
   chmod +x calculate_quota_usage.sh
   ```
3. Run the script:
   ```bash
   ./calculate_quota_usage.sh
   ```

### Dependencies:
- **`jq`**: Ensure `jq` is installed for JSON parsing. You can install it using your package manager, e.g., `sudo apt-get install jq` or `brew install jq`.

This script will display each resource's name, type, used value, hard limit, and the usage percentage. Adjust the formatting or logic as needed to fit your requirements.

