**Swapan**: bash command line option choice between three options

**ChatGPT**: To create a Bash command-line option that allows users to choose between three options, you can use the `case` statement. Below is an example script:

### Example Script

```bash
#!/bin/bash

# Display usage information
usage() {
    echo "Usage: $0 -o <option>"
    echo "Options:"
    echo "  1    Perform action for option 1"
    echo "  2    Perform action for option 2"
    echo "  3    Perform action for option 3"
    exit 1
}

# Parse command-line options
while getopts ":o:" opt; do
    case $opt in
        o)
            OPTION=$OPTARG
            ;;
        *)
            usage
            ;;
    esac
done

# Ensure an option was provided
if [ -z "$OPTION" ]; then
    usage
fi

# Perform actions based on the option
case $OPTION in
    1)
        echo "You chose option 1!"
        # Add commands for option 1
        ;;
    2)
        echo "You chose option 2!"
        # Add commands for option 2
        ;;
    3)
        echo "You chose option 3!"
        # Add commands for option 3
        ;;
    *)
        echo "Invalid option: $OPTION"
        usage
        ;;
esac
```

### How It Works:
1. **Usage Function**: Displays a usage message if the user provides incorrect input or no input.
2. **Command-line Options**: Uses `getopts` to parse the `-o` option followed by the choice (`1`, `2`, or `3`).
3. **Case Statement**: Executes the appropriate actions based on the option selected.

### Example Usage:
```bash
# Run the script with option 1
./script.sh -o 1

# Run the script with option 2
./script.sh -o 2

# Run the script with option 3
./script.sh -o 3
```

This is a robust and user-friendly way to handle multiple options in a Bash script.

**Swapan**: 1st parameter is a required parameter for id, 2nd parameter is once of 3 options  like above

**ChatGPT**: Here's how you can create a Bash script where the **first parameter is a required ID** and the **second parameter is one of three predefined options**:

### Example Script

```bash
#!/bin/bash

# Display usage information
usage() {
    echo "Usage: $0 <id> <option>"
    echo "Arguments:"
    echo "  id       A required parameter for the ID."
    echo "  option   One of the following options:"
    echo "           1    Perform action for option 1"
    echo "           2    Perform action for option 2"
    echo "           3    Perform action for option 3"
    exit 1
}

# Check if the required parameters are provided
if [ "$#" -ne 2 ]; then
    echo "Error: Invalid number of arguments."
    usage
fi

# Assign parameters to variables
ID=$1
OPTION=$2

# Validate the second parameter
case $OPTION in
    1)
        echo "ID: $ID | You chose option 1!"
        # Add commands for option 1
        ;;
    2)
        echo "ID: $ID | You chose option 2!"
        # Add commands for option 2
        ;;
    3)
        echo "ID: $ID | You chose option 3!"
        # Add commands for option 3
        ;;
    *)
        echo "Error: Invalid option '$OPTION'."
        usage
        ;;
esac
```

### How It Works:
1. **Argument Validation**:
   - Checks if exactly 2 arguments are provided.
   - First argument is treated as the ID.
   - Second argument is validated to ensure it matches one of the options (`1`, `2`, `3`).
2. **Usage Function**:
   - Provides a guide for users to understand the script's input requirements.
3. **Case Statement**:
   - Executes the appropriate commands based on the chosen option.

### Example Usage:
```bash
# Valid commands
./script.sh 123 1   # ID: 123, Option 1
./script.sh 456 2   # ID: 456, Option 2
./script.sh 789 3   # ID: 789, Option 3

# Invalid commands
./script.sh          # Missing arguments
./script.sh 123      # Missing second argument
./script.sh 123 4    # Invalid second argument
```

### Output:
For a valid command:
```bash
ID: 123 | You chose option 1!
```

For an invalid command:
```bash
Error: Invalid option '4'.
Usage: ./script.sh <id> <option>
Arguments:
  id       A required parameter for the ID.
  option   One of the following options:
           1    Perform action for option 1
           2    Perform action for option 2
           3    Perform action for option 3
``` 

This ensures that the ID is always required and the second argument is restricted to the specified options.

**Swapan**: send option prompts you to chose 1 of the 3 and waits

**ChatGPT**: To create a Bash script where the first parameter is the required `ID`, and the script **prompts the user to choose one of three options** if the second parameter is missing, you can use the following:

### Script Example

```bash
#!/bin/bash

# Display usage information
usage() {
    echo "Usage: $0 <id> [option]"
    echo "Arguments:"
    echo "  id       A required parameter for the ID."
    echo "  option   One of the following options (optional if not provided, prompts the user):"
    echo "           1    Perform action for option 1"
    echo "           2    Perform action for option 2"
    echo "           3    Perform action for option 3"
    exit 1
}

# Check if at least one parameter is provided
if [ "$#" -lt 1 ]; then
    echo "Error: ID is a required argument."
    usage
fi

# Assign the first parameter as the ID
ID=$1

# Prompt for the second parameter if it's missing
if [ -z "$2" ]; then
    echo "Please choose an option:"
    echo "1) Option 1"
    echo "2) Option 2"
    echo "3) Option 3"
    read -p "Enter your choice (1, 2, or 3): " OPTION
else
    OPTION=$2
fi

# Validate the selected option
case $OPTION in
    1)
        echo "ID: $ID | You chose option 1!"
        # Add commands for option 1
        ;;
    2)
        echo "ID: $ID | You chose option 2!"
        # Add commands for option 2
        ;;
    3)
        echo "ID: $ID | You chose option 3!"
        # Add commands for option 3
        ;;
    *)
        echo "Error: Invalid option '$OPTION'."
        usage
        ;;
esac
```

### How It Works:
1. **Argument Validation**:
   - The script checks if at least the `ID` (first argument) is provided. If not, it displays the usage.
   - If the second argument (`OPTION`) is not provided, it prompts the user to choose an option.
2. **Prompt for Missing Option**:
   - If the user doesn't provide the second argument, a `read` command is used to wait for input.
3. **Validation**:
   - The script validates the user-provided option and proceeds accordingly.

### Example Usage:
```bash
# Valid command with both arguments
./script.sh 123 2
# Output: ID: 123 | You chose option 2!

# Valid command with only ID (prompts for option)
./script.sh 456
# Output:
# Please choose an option:
# 1) Option 1
# 2) Option 2
# 3) Option 3
# Enter your choice (1, 2, or 3): 3
# ID: 456 | You chose option 3!

# Invalid option
./script.sh 123 5
# Output:
# Error: Invalid option '5'.
# Usage: ./script.sh <id> [option]
```

### Notes:
- The `read` command ensures the script pauses and waits for user input when the second argument is missing.
- The `case` statement validates the input and performs the corresponding action.

**Swapan**: allow only names of  openshift clusters as one of 3 options 

**ChatGPT**: Here is an updated version of the script where the second parameter can only accept names of OpenShift clusters as predefined options. If the second parameter is not provided, it prompts the user to choose one of the cluster names.

### Script Example

```bash
#!/bin/bash

# Define valid OpenShift cluster names
VALID_CLUSTERS=("dev-cluster" "test-cluster" "prod-cluster")

# Display usage information
usage() {
    echo "Usage: $0 <id> [cluster]"
    echo "Arguments:"
    echo "  id        A required parameter for the ID."
    echo "  cluster   One of the following OpenShift clusters (optional if not provided, prompts the user):"
    for cluster in "${VALID_CLUSTERS[@]}"; do
        echo "            $cluster"
    done
    exit 1
}

# Check if at least one parameter is provided
if [ "$#" -lt 1 ]; then
    echo "Error: ID is a required argument."
    usage
fi

# Assign the first parameter as the ID
ID=$1

# Validate or prompt for the second parameter
if [ -z "$2" ]; then
    echo "Please choose one of the following OpenShift clusters:"
    select cluster in "${VALID_CLUSTERS[@]}"; do
        if [[ " ${VALID_CLUSTERS[*]} " =~ " $cluster " ]]; then
            CLUSTER=$cluster
            break
        else
            echo "Invalid selection. Please try again."
        fi
    done
else
    if [[ " ${VALID_CLUSTERS[*]} " =~ " $2 " ]]; then
        CLUSTER=$2
    else
        echo "Error: Invalid cluster name '$2'."
        usage
    fi
fi

# Perform actions based on the selected cluster
echo "ID: $ID | You selected the OpenShift cluster: $CLUSTER"
# Add your OpenShift-related commands here

```

### How It Works:
1. **Valid Clusters**:
   - `VALID_CLUSTERS` is an array containing the predefined OpenShift cluster names.
2. **Validation**:
   - If the second parameter (`$2`) is missing, the script prompts the user to select a valid cluster using the `select` command.
   - If the second parameter is provided, it checks if it matches one of the cluster names in the `VALID_CLUSTERS` array.
3. **Prompting with `select`**:
   - The `select` command displays the list of clusters and allows the user to choose one.
4. **Error Handling**:
   - Invalid selections or parameters trigger the `usage` function, which displays the valid options and exits.

### Example Usage:
#### Provide Both Arguments:
```bash
./script.sh 123 dev-cluster
# Output: ID: 123 | You selected the OpenShift cluster: dev-cluster
```

#### Provide Only ID (Prompts for Cluster):
```bash
./script.sh 456
# Output:
# Please choose one of the following OpenShift clusters:
# 1) dev-cluster
# 2) test-cluster
# 3) prod-cluster
# #? 2
# ID: 456 | You selected the OpenShift cluster: test-cluster
```

#### Invalid Cluster Name:
```bash
./script.sh 123 invalid-cluster
# Output:
# Error: Invalid cluster name 'invalid-cluster'.
# Usage: ./script.sh <id> [cluster]
# Arguments:
#   id        A required parameter for the ID.
#   cluster   One of the following OpenShift clusters:
#             dev-cluster
#             test-cluster
#             prod-cluster
```

### Benefits:
- Ensures only valid OpenShift cluster names are accepted.
- Provides user-friendly interaction if the cluster name is missing.
- Clear error messages guide the user to correct usage.

**Swapan**: allow parameters like ./script.sh --id 123 --os-scluster dev-cluster 

**ChatGPT**: Here is a version of the script that supports the `--id` and `--os-cluster` parameters explicitly using long-form options:

### Script Example

```bash
#!/bin/bash

# Define valid OpenShift cluster names
VALID_CLUSTERS=("dev-cluster" "test-cluster" "prod-cluster")

# Display usage information
usage() {
    echo "Usage: $0 --id <id> --os-cluster <cluster>"
    echo "Arguments:"
    echo "  --id          A required parameter for the ID."
    echo "  --os-cluster  One of the following OpenShift clusters (optional if not provided, prompts the user):"
    for cluster in "${VALID_CLUSTERS[@]}"; do
        echo "                $cluster"
    done
    exit 1
}

# Parse arguments
ID=""
CLUSTER=""
while [[ $# -gt 0 ]]; do
    case "$1" in
        --id)
            ID=$2
            shift 2
            ;;
        --os-cluster)
            CLUSTER=$2
            shift 2
            ;;
        *)
            echo "Error: Unknown argument '$1'"
            usage
            ;;
    esac
done

# Validate the ID
if [ -z "$ID" ]; then
    echo "Error: The --id parameter is required."
    usage
fi

# Validate or prompt for the OpenShift cluster
if [ -z "$CLUSTER" ]; then
    echo "Please choose one of the following OpenShift clusters:"
    select cluster in "${VALID_CLUSTERS[@]}"; do
        if [[ " ${VALID_CLUSTERS[*]} " =~ " $cluster " ]]; then
            CLUSTER=$cluster
            break
        else
            echo "Invalid selection. Please try again."
        fi
    done
else
    if [[ ! " ${VALID_CLUSTERS[*]} " =~ " $CLUSTER " ]]; then
        echo "Error: Invalid cluster name '$CLUSTER'."
        usage
    fi
fi

# Perform actions based on the selected ID and cluster
echo "ID: $ID | You selected the OpenShift cluster: $CLUSTER"
# Add your OpenShift-related commands here
```

---

### How It Works:
1. **Parameter Parsing**:
   - The script parses arguments using `case` to match `--id` and `--os-cluster`.
   - Each option is followed by its value, which is extracted into variables (`ID` and `CLUSTER`).
2. **Validation**:
   - The `--id` parameter is mandatory. If missing, the script displays an error and exits.
   - If the `--os-cluster` parameter is missing, the script prompts the user to select a cluster.
   - If a provided cluster name does not match the predefined valid clusters, an error is shown.
3. **Cluster Selection Prompt**:
   - If the user doesn’t provide the `--os-cluster` parameter, a menu is displayed using the `select` command.
4. **Error Handling**:
   - If an unknown argument is provided, the script displays the usage instructions.

---

### Example Usage:
#### Provide Both Parameters:
```bash
./script.sh --id 123 --os-cluster dev-cluster
# Output: ID: 123 | You selected the OpenShift cluster: dev-cluster
```

#### Provide Only ID (Prompts for Cluster):
```bash
./script.sh --id 456
# Output:
# Please choose one of the following OpenShift clusters:
# 1) dev-cluster
# 2) test-cluster
# 3) prod-cluster
# #? 2
# ID: 456 | You selected the OpenShift cluster: test-cluster
```

#### Missing `--id` Parameter:
```bash
./script.sh --os-cluster test-cluster
# Output:
# Error: The --id parameter is required.
# Usage: ./script.sh --id <id> --os-cluster <cluster>
# Arguments:
#   --id          A required parameter for the ID.
#   --os-cluster  One of the following OpenShift clusters:
#                 dev-cluster
#                 test-cluster
#                 prod-cluster
```

#### Invalid Cluster Name:
```bash
./script.sh --id 123 --os-cluster invalid-cluster
# Output:
# Error: Invalid cluster name 'invalid-cluster'.
# Usage: ./script.sh --id <id> --os-cluster <cluster>
# Arguments:
#   --id          A required parameter for the ID.
#   --os-cluster  One of the following OpenShift clusters:
#                 dev-cluster
#                 test-cluster
#                 prod-cluster
```

---

### Key Features:
- **Explicit Long-form Parameters**:
  Users must provide `--id` and optionally `--os-cluster`.
- **Interactive Prompt**:
  If the `--os-cluster` parameter is omitted, the script interactively asks the user to choose a cluster.
- **Validation**:
  Ensures the user inputs a valid ID and a valid cluster name from the predefined list.

