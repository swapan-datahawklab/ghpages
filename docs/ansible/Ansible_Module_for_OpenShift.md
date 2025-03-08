**Swapan**: create ansible module that wraps to openshift oc binary 

**ChatGPT**: **Creating An Ansible Module**

I’m developing an Ansible module to wrap the OpenShift 'oc' binary for executing 'oc' commands, ensuring helpfulness, accuracy, and policy adherence.

**Wrapping the command**

I'm working on an Ansible module to wrap 'oc' commands in Python, utilizing the 'ansible.module_utils.basic' AnsibleModule class.

**ChatGPT**: Certainly! Below is an example of a custom Ansible module that wraps the OpenShift `oc` binary. This module allows you to run `oc` commands within your Ansible playbooks.

### Custom Ansible Module: `oc_command`

#### Module Code (`oc_command.py`)

Create a file named `oc_command.py` in the `library` directory of your Ansible project:

```python
#!/usr/bin/python

from ansible.module_utils.basic import AnsibleModule
import subprocess
import shlex

def main():
    module = AnsibleModule(
        argument_spec=dict(
            command=dict(type='str', required=True),
            kubeconfig=dict(type='str', required=False, default=None),
            namespace=dict(type='str', required=False, default=None),
            context=dict(type='str', required=False, default=None),
            ignore_errors=dict(type='bool', required=False, default=False),
        ),
        supports_check_mode=True,
    )

    command = module.params['command']
    kubeconfig = module.params['kubeconfig']
    namespace = module.params['namespace']
    context = module.params['context']
    ignore_errors = module.params['ignore_errors']

    oc_command = ['oc'] + shlex.split(command)

    if kubeconfig:
        oc_command.extend(['--kubeconfig', kubeconfig])

    if namespace:
        oc_command.extend(['--namespace', namespace])

    if context:
        oc_command.extend(['--context', context])

    rc, out, err = module.run_command(oc_command)

    result = {
        'changed': False,
        'rc': rc,
        'stdout': out,
        'stderr': err,
        'cmd': ' '.join(oc_command),
    }

    if rc != 0 and not ignore_errors:
        module.fail_json(msg="Error running oc command", **result)

    module.exit_json(**result)

if __name__ == '__main__':
    main()
```

#### Parameters

- **command**: The `oc` command to run (e.g., `"get pods"`). *(Required)*
- **kubeconfig**: Path to a kubeconfig file to use.
- **namespace**: Kubernetes namespace to use.
- **context**: Kubernetes context to use.
- **ignore_errors**: Whether to ignore errors and continue. *(Default: `False`)*

#### Usage Example

Here's how you can use the `oc_command` module in your Ansible playbook:

```yaml
---
- name: Execute oc commands using custom module
  hosts: localhost
  tasks:
    - name: Get all pods in the default namespace
      oc_command:
        command: "get pods"
      register: pod_list

    - name: Display the list of pods
      debug:
        var: pod_list.stdout_lines

    - name: Create a new project
      oc_command:
        command: "new-project my-project"
        ignore_errors: yes  # Continue even if the project already exists

    - name: Apply a configuration file
      oc_command:
        command: "apply -f /path/to/config.yaml"
        namespace: my-project
```

### Notes

- **Error Handling**: The module will fail the task if the `oc` command returns a non-zero exit code unless `ignore_errors` is set to `True`.
- **Output**: The module returns the standard output and error streams, which you can register and use in subsequent tasks.
- **Flexibility**: This module provides a flexible way to run any `oc` command, making it easier to integrate OpenShift operations into your Ansible workflows.

### Advantages

- **Consistency**: Use the same `oc` commands you're familiar with directly in Ansible.
- **Automation**: Automate OpenShift tasks as part of your deployment processes.
- **Integration**: Combine OpenShift operations with other Ansible-managed tasks.

### Additional Enhancements

- **Parsing JSON Output**: You can modify the module to parse JSON output when using `-o json` with `oc` commands.
- **Error Messages**: Improve error messages by providing more context from the `stderr`.

---

Feel free to customize the module further to suit your specific needs!

**Swapan**: cnvert to bash

**ChatGPT**: **Converting Python to Bash**

I’m working through the user's request to switch an Ansible module from Python to bash. Ansible modules typically use Python, but they can also operate in other languages, including bash.

**Preparing the script**

I'm working on a bash script to serve as an Ansible module, wrapping the `oc` binary. This involves accepting JSON input and outputting JSON while adhering to specific Ansible module standards.

**Setting up the script**

I’m outlining the module’s parameters and functionality, detailing steps like accepting input, executing commands, capturing output, and handling errors. Parsing JSON in bash is tricky but possible with `jq`.

**Evaluating parsing options**

I’m weighing different ways to parse JSON in bash, including shell constructs, a helper function, or a bash script as an Ansible module using `jq`.

**Crafting the script**

I’m creating a bash script to execute OpenShift CLI (`oc`) commands, ensuring essential parameters are present and parsing JSON input. The script accommodates optional parameters like `kubeconfig`, `namespace`, and `context`.

**Piecing together methods**

Ansible modules can acquire arguments via `ANSIBLE_MODULE_ARGS` or command line parameters. This approach, simplified by the Ansible 2.0 module API, is particularly useful for non-Python modules.

**Executing and handling**

I’m parsing JSON input to build and run an `oc` command. I capture its output, check for errors, and format the results as JSON.

**ChatGPT**: Certainly! Below is an example of the custom Ansible module rewritten in **Bash**. This module wraps the OpenShift `oc` binary and allows you to execute `oc` commands within your Ansible playbooks.

### Custom Ansible Module: `oc_command` (Bash Version)

#### Module Code (`oc_command.sh`)

Create a file named `oc_command.sh` in the `library` directory of your Ansible project and make it executable:

```bash
#!/bin/bash

# Read the JSON input from stdin
INPUT_JSON=$(cat)

# Use jq to parse parameters
COMMAND=$(echo "$INPUT_JSON" | jq -r '.command // empty')
KUBECONFIG=$(echo "$INPUT_JSON" | jq -r '.kubeconfig // empty')
NAMESPACE=$(echo "$INPUT_JSON" | jq -r '.namespace // empty')
CONTEXT=$(echo "$INPUT_JSON" | jq -r '.context // empty')
IGNORE_ERRORS=$(echo "$INPUT_JSON" | jq -r '.ignore_errors // false')

# Check required parameter
if [ -z "$COMMAND" ]; then
    echo '{"failed": true, "msg": "command parameter is required"}'
    exit 1
fi

# Build the oc command
OC_COMMAND=("oc")
# Split the command into array
read -r -a COMMAND_ARRAY <<< "$COMMAND"
OC_COMMAND+=("${COMMAND_ARRAY[@]}")

if [ -n "$KUBECONFIG" ]; then
    OC_COMMAND+=("--kubeconfig" "$KUBECONFIG")
fi

if [ -n "$NAMESPACE" ]; then
    OC_COMMAND+=("--namespace" "$NAMESPACE")
fi

if [ -n "$CONTEXT" ]; then
    OC_COMMAND+=("--context" "$CONTEXT")
fi

# Run the oc command
OUTPUT=$("${OC_COMMAND[@]}" 2>&1)
RC=$?

# Prepare result JSON
RESULT_JSON=$(jq -n \
    --argjson changed false \
    --argjson rc "$RC" \
    --arg stdout "$OUTPUT" \
    --arg cmd "${OC_COMMAND[*]}" \
    '{changed: $changed, rc: $rc, stdout: $stdout, cmd: $cmd}')

if [ "$RC" -ne 0 ] && [ "$IGNORE_ERRORS" != "True" ] && [ "$IGNORE_ERRORS" != "true" ]; then
    echo "$RESULT_JSON" | jq '. + {failed: true, msg: "Error running oc command"}'
    exit 1
else
    echo "$RESULT_JSON"
    exit 0
fi
```

**Make sure to give execution permission to the script:**

```bash
chmod +x library/oc_command.sh
```

#### Parameters

- **command**: The `oc` command to run (e.g., `"get pods"`). *(Required)*
- **kubeconfig**: Path to a kubeconfig file to use.
- **namespace**: Kubernetes namespace to use.
- **context**: Kubernetes context to use.
- **ignore_errors**: Whether to ignore errors and continue. *(Default: `false`)*

#### Usage Example

Here's how you can use the `oc_command` module in your Ansible playbook:

```yaml
---
- name: Execute oc commands using custom module
  hosts: localhost
  vars:
    ansible_python_interpreter: /usr/bin/python3
  tasks:
    - name: Get all pods in the default namespace
      oc_command:
        command: "get pods"
      register: pod_list

    - name: Display the list of pods
      debug:
        var: pod_list.stdout_lines

    - name: Create a new project
      oc_command:
        command: "new-project my-project"
        ignore_errors: yes  # Continue even if the project already exists

    - name: Apply a configuration file
      oc_command:
        command: "apply -f /path/to/config.yaml"
        namespace: my-project
```

#### Notes

- **Dependencies**: This module requires `jq` to be installed on the target machine where the module is executed.
  - Install `jq` using your package manager, e.g., `sudo yum install -y jq` or `sudo apt-get install -y jq`.
- **Error Handling**: The module will fail the task if the `oc` command returns a non-zero exit code unless `ignore_errors` is set to `True`.
- **Output**: The module returns the standard output and error streams, which you can register and use in subsequent tasks.
- **Execution Permission**: Ensure the script has execution permission and is placed in the correct `library` directory.

#### Advantages

- **Simplicity**: Uses Bash scripting, which may be easier to modify for those familiar with shell scripting.
- **Compatibility**: Can be used in environments where Python is not preferred or available.

### Additional Enhancements

- **Parsing JSON Output**: Modify the module to parse JSON output when using `-o json` with `oc` commands by parsing `stdout` with `jq`.
- **Verbose Logging**: Add more detailed logging for troubleshooting purposes.
- **Error Messages**: Improve error messages by providing more context from the `stderr` output.

---

Feel free to customize the module further to suit your specific needs!

**Swapan**: add functionality to get pod metrics and info by selector 

**ChatGPT**: **Integrating selector functionality**

I’m updating the Bash Ansible module to fetch pod metrics and info by selector, ensuring the response is policy compliant, precise, and helpful.

**Updating the Bash module**

I’m updating the Bash Ansible module to include pod metrics and info retrieval via selector. New parameters like 'metrics' and 'selector' will refine 'oc get pods --selector ...' and 'oc adm top pods ...' commands.

**Showcasing updates**

The assistant is preparing to provide the updated module code, explicate new parameters, demonstrate usage examples, and include necessary notes.

**Enhancing the module**

I’ve updated the Ansible module `oc_command.sh` to include `selector` and `metrics`, allowing for fetching pod information and metrics by selector. This enhances flexibility and utility in managing OpenShift environments.

**ChatGPT**: Certainly! I've updated the `oc_command` Bash module to include functionality for retrieving pod metrics and information using selectors.

### Updated Custom Ansible Module: `oc_command` (Bash Version with Selector and Metrics)

#### Module Code (`oc_command.sh`)

Here's the enhanced `oc_command.sh` script with the new parameters `selector` and `metrics`:

```bash
#!/bin/bash

# Read the JSON input from stdin
INPUT_JSON=$(cat)

# Use jq to parse parameters
COMMAND=$(echo "$INPUT_JSON" | jq -r '.command // empty')
KUBECONFIG=$(echo "$INPUT_JSON" | jq -r '.kubeconfig // empty')
NAMESPACE=$(echo "$INPUT_JSON" | jq -r '.namespace // empty')
CONTEXT=$(echo "$INPUT_JSON" | jq -r '.context // empty')
IGNORE_ERRORS=$(echo "$INPUT_JSON" | jq -r '.ignore_errors // false')
SELECTOR=$(echo "$INPUT_JSON" | jq -r '.selector // empty')
METRICS=$(echo "$INPUT_JSON" | jq -r '.metrics // false')

# Check for required parameter when not using metrics or selector
if [ -z "$COMMAND" ] && [ -z "$SELECTOR" ] && [ "$METRICS" != "true" ] && [ "$METRICS" != "True" ]; then
    echo '{"failed": true, "msg": "command parameter is required unless selector or metrics is specified"}'
    exit 1
fi

# Build the oc command
OC_COMMAND=("oc")

if [ "$METRICS" == "true" ] || [ "$METRICS" == "True" ]; then
    # Use 'oc adm top pods' for metrics
    OC_COMMAND+=("adm" "top" "pods")
elif [ -n "$COMMAND" ]; then
    # Split the command into array
    read -r -a COMMAND_ARRAY <<< "$COMMAND"
    OC_COMMAND+=("${COMMAND_ARRAY[@]}")
fi

if [ -n "$SELECTOR" ]; then
    OC_COMMAND+=("--selector" "$SELECTOR")
fi

if [ -n "$KUBECONFIG" ]; then
    OC_COMMAND+=("--kubeconfig" "$KUBECONFIG")
fi

if [ -n "$NAMESPACE" ]; then
    OC_COMMAND+=("--namespace" "$NAMESPACE")
fi

if [ -n "$CONTEXT" ]; then
    OC_COMMAND+=("--context" "$CONTEXT")
fi

# Run the oc command
OUTPUT=$("${OC_COMMAND[@]}" 2>&1)
RC=$?

# Prepare result JSON
RESULT_JSON=$(jq -n \
    --argjson changed false \
    --argjson rc "$RC" \
    --arg stdout "$OUTPUT" \
    --arg cmd "${OC_COMMAND[*]}" \
    '{changed: $changed, rc: $rc, stdout: $stdout, cmd: $cmd}')

if [ "$RC" -ne 0 ] && [ "$IGNORE_ERRORS" != "True" ] && [ "$IGNORE_ERRORS" != "true" ]; then
    echo "$RESULT_JSON" | jq '. + {failed: true, msg: "Error running oc command"}'
    exit 1
else
    echo "$RESULT_JSON"
    exit 0
fi
```

**Ensure the script is executable:**

```bash
chmod +x library/oc_command.sh
```

#### New Parameters

- **selector**: A label selector to filter pods (e.g., `"app=myapp"`). *(Optional)*
- **metrics**: A boolean indicating whether to fetch pod metrics instead of basic pod information. *(Default: `false`)*

#### Usage Examples

Here are some examples of how to use the updated `oc_command` module in your Ansible playbooks.

##### Example 1: Get Pod Information by Selector

```yaml
- name: Get pods with a specific label
  oc_command:
    selector: "app=myapp"
    namespace: my-namespace
  register: pod_info

- name: Display pod information
  debug:
    var: pod_info.stdout_lines
```

##### Example 2: Get Pod Metrics by Selector

```yaml
- name: Get pod metrics for pods with a specific label
  oc_command:
    selector: "app=myapp"
    namespace: my-namespace
    metrics: true
  register: pod_metrics

- name: Display pod metrics
  debug:
    var: pod_metrics.stdout_lines
```

##### Example 3: Use Additional Command Arguments

You can still use the `command` parameter to pass additional `oc` command arguments.

```yaml
- name: Describe pods with a specific label
  oc_command:
    command: "describe pods"
    selector: "app=myapp"
    namespace: my-namespace
  register: pod_description

- name: Display pod description
  debug:
    var: pod_description.stdout_lines
```

#### Notes

- **Parameter Precedence**: When `metrics` is set to `true`, the module uses `oc adm top pods` to fetch metrics. Otherwise, it uses `oc get pods` or the command specified in `command`.
- **Selector Usage**: The `selector` parameter allows you to filter pods based on label selectors, similar to the `--selector` option in `oc` commands.
- **Combining Parameters**: You can combine `selector`, `namespace`, `kubeconfig`, and other parameters to customize the `oc` command execution.
- **Error Handling**: As before, the module handles errors based on the `ignore_errors` parameter.

#### Dependencies

- **jq**: Ensure `jq` is installed on the target system as it is required for JSON parsing.
  - Install `jq` using your package manager:
    - For CentOS/RHEL: `sudo yum install -y jq`
    - For Ubuntu/Debian: `sudo apt-get install -y jq`

#### Advantages of the Update

- **Enhanced Functionality**: The module now directly supports fetching pod metrics and information based on label selectors.
- **Flexibility**: Users can easily get metrics or detailed information for a subset of pods without constructing complex `oc` commands manually.
- **Integration**: Simplifies the integration of OpenShift pod monitoring and management into Ansible workflows.

### Additional Enhancements

- **JSON Output Parsing**: Modify the module to parse JSON output by adding `-o json` to the `oc` commands when needed.
- **Custom Output Formatting**: Allow specifying output formats like `yaml` or `wide` for more detailed information.
- **Error Details**: Include more detailed error messages by capturing and returning `stderr`.

---

Feel free to adjust or extend the module to better suit your requirements!

**Swapan**: create a no low dependency python versoin of the module

**ChatGPT**: **Creating a minimal Python module**

The assistant is tasked with developing a Python module similar to the Bash version, but with minimal dependencies.

**Crafting the solution**

Piecing together a Python version of the module with minimal dependencies, supporting pod metrics and info retrieval by selector, and ensuring ease of use and integration.

**Crafting Python code**

I’m working on a Python module to wrap the 'oc' command, ensuring it can run arbitrary 'oc' commands and get pod metrics/info by selector, with a focus on minimal dependencies.

**ChatGPT**: Certainly! Below is a custom Ansible module written in Python with minimal dependencies. This module wraps the OpenShift `oc` binary and includes functionality to get pod metrics and information by selector.

### Custom Ansible Module: `oc_command` (Low-Dependency Python Version)

#### Module Code (`oc_command.py`)

Create a file named `oc_command.py` in the `library` directory of your Ansible project:

```python
#!/usr/bin/python

from ansible.module_utils.basic import AnsibleModule
import subprocess
import shlex

def main():
    module = AnsibleModule(
        argument_spec=dict(
            command=dict(type='str', required=False, default=None),
            kubeconfig=dict(type='str', required=False, default=None),
            namespace=dict(type='str', required=False, default=None),
            context=dict(type='str', required=False, default=None),
            ignore_errors=dict(type='bool', required=False, default=False),
            selector=dict(type='str', required=False, default=None),
            metrics=dict(type='bool', required=False, default=False),
        ),
        supports_check_mode=True,
        required_one_of=[['command', 'selector', 'metrics']],
    )

    command = module.params['command']
    kubeconfig = module.params['kubeconfig']
    namespace = module.params['namespace']
    context = module.params['context']
    ignore_errors = module.params['ignore_errors']
    selector = module.params['selector']
    metrics = module.params['metrics']

    oc_command = ['oc']

    if metrics:
        oc_command.extend(['adm', 'top', 'pods'])
    elif command:
        oc_command.extend(shlex.split(command))
    else:
        oc_command.extend(['get', 'pods'])

    if selector:
        oc_command.extend(['--selector', selector])

    if kubeconfig:
        oc_command.extend(['--kubeconfig', kubeconfig])

    if namespace:
        oc_command.extend(['--namespace', namespace])

    if context:
        oc_command.extend(['--context', context])

    cmd_str = ' '.join(shlex.quote(arg) for arg in oc_command)

    rc, out, err = module.run_command(oc_command)

    result = {
        'changed': False,
        'rc': rc,
        'stdout': out,
        'stderr': err,
        'cmd': cmd_str,
    }

    if rc != 0 and not ignore_errors:
        module.fail_json(msg="Error running oc command", **result)

    module.exit_json(**result)

if __name__ == '__main__':
    main()
```

#### Parameters

- **command**: The `oc` command to run (e.g., `"get pods"`). *(Optional)*
- **kubeconfig**: Path to a kubeconfig file to use. *(Optional)*
- **namespace**: Kubernetes namespace to use. *(Optional)*
- **context**: Kubernetes context to use. *(Optional)*
- **ignore_errors**: Whether to ignore errors and continue. *(Default: `False`)*
- **selector**: Label selector to filter pods (e.g., `"app=myapp"`). *(Optional)*
- **metrics**: Whether to fetch pod metrics instead of basic information. *(Default: `False`)*

> **Note**: At least one of `command`, `selector`, or `metrics` must be specified.

#### Usage Examples

Below are examples of how to use the `oc_command` module in your Ansible playbooks.

##### Example 1: Get Pod Information by Selector

```yaml
- name: Get pods with a specific label
  oc_command:
    selector: "app=myapp"
    namespace: my-namespace
  register: pod_info

- name: Display pod information
  debug:
    var: pod_info.stdout_lines
```

##### Example 2: Get Pod Metrics by Selector

```yaml
- name: Get pod metrics for pods with a specific label
  oc_command:
    selector: "app=myapp"
    namespace: my-namespace
    metrics: true
  register: pod_metrics

- name: Display pod metrics
  debug:
    var: pod_metrics.stdout_lines
```

##### Example 3: Run Custom `oc` Command

```yaml
- name: Describe pods with a specific label
  oc_command:
    command: "describe pods"
    selector: "app=myapp"
    namespace: my-namespace
  register: pod_description

- name: Display pod description
  debug:
    var: pod_description.stdout_lines
```

##### Example 4: Apply Configuration File

```yaml
- name: Apply a configuration file
  oc_command:
    command: "apply -f /path/to/config.yaml"
    namespace: my-project
```

#### Notes

- **Minimal Dependencies**: This module only uses the standard Python library and `ansible.module_utils.basic` which is available in Ansible environments.
- **Error Handling**: The module will fail the task if the `oc` command returns a non-zero exit code unless `ignore_errors` is set to `True`.
- **Command Construction**: The module intelligently constructs the `oc` command based on the parameters provided.
- **Parameter Precedence**:
  - If `metrics` is `True`, the module will execute `oc adm top pods`.
  - If `command` is provided, it will be used.
  - If neither `metrics` nor `command` is provided but `selector` is specified, it defaults to `oc get pods`.

#### Advantages of This Module

- **Low Dependency**: Relies only on the standard Python library and Ansible's built-in modules.
- **Flexible**: Supports fetching metrics, information by selector, and running custom commands.
- **Integration**: Easily integrates with Ansible playbooks for automating OpenShift operations.

#### Additional Enhancements

- **JSON Output Parsing**: To parse JSON output, you can modify the module to add `-o json` to the `oc` commands and parse the output using `json` module.
- **Custom Output Formats**: Allow users to specify output formats (e.g., `json`, `yaml`, `wide`) by adding an `output` parameter.
- **Enhanced Error Messages**: Include more detailed error messages by returning `stderr` in the result.

#### Example with JSON Output Parsing (Optional Enhancement)

If you often use `-o json` and want to parse the output within the module, you can enhance the module as follows:

```python
import json

# Inside the main function, after running the command
if rc == 0 and '-o json' in cmd_str:
    try:
        json_output = json.loads(out)
        result['json_output'] = json_output
    except json.JSONDecodeError:
        module.fail_json(msg="Failed to parse JSON output", **result)
```

And in your playbook:

```yaml
- name: Get pods as JSON
  oc_command:
    command: "get pods -o json"
    namespace: my-namespace
  register: pod_info

- name: Display pod names
  debug:
    msg: "{{ pod_info.json_output.items | map(attribute='metadata.name') | list }}"
```

---

Feel free to customize the module further to suit your specific needs!

### Placement and Execution

- **File Location**: Save the `oc_command.py` file in the `library` directory relative to your playbook or in a directory specified in `ANSIBLE_LIBRARY` environment variable.
- **Permissions**: Ensure the module file has the appropriate permissions to be executed.
- **Ansible Version**: This module should work with Ansible 2.9 and later versions.

### Troubleshooting

- **Module Not Found**: If Ansible cannot find the module, check that the `library` directory is correctly placed and that the module file is named `oc_command.py`.
- **Dependencies**: Ensure that the `oc` command-line tool is installed and accessible in the system `PATH` for the user executing the Ansible playbook.
- **Permissions**: The user running the Ansible playbook must have the necessary permissions to execute `oc` commands and access the Kubernetes cluster.

### Conclusion

This custom Ansible module provides a flexible and low-dependency way to interact with OpenShift clusters using the `oc` command-line tool. By integrating it into your Ansible playbooks, you can automate and manage your OpenShift resources efficiently.

---

If you have any questions or need further assistance with the module, feel free to ask!

