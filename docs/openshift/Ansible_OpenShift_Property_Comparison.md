**Swapan**: ansible module to read properties file compare against openshift oc cli output and set series of playbook vars

**ChatGPT**: To accomplish this task with an Ansible module that reads a properties file, compares its values against OpenShift CLI (`oc`) output, and sets playbook variables, you can use a combination of Ansible built-in modules and custom logic. Here's how to set it up:

1. **Read the properties file**: Use the `slurp` module or `lookup` plugin to read the contents of the properties file.
2. **Parse the properties file**: Use `ini` format handling with `from_yaml` filter to convert its contents to a dictionary.
3. **Run `oc` commands**: Use the `command` or `k8s_info` modules to get the output from OpenShift.
4. **Set playbook variables**: Use the `set_fact` module.

### Example Playbook

```yaml
- name: Read properties file and set playbook variables
  hosts: localhost
  gather_facts: no
  vars:
    properties_file_path: "/path/to/your/properties/file.properties"

  tasks:
    - name: Read properties file
      slurp:
        src: "{{ properties_file_path }}"
      register: properties_file_content

    - name: Parse properties file into dictionary
      set_fact:
        properties_dict: "{{ properties_file_content['content'] | b64decode | from_yaml }}"

    - name: Run 'oc' command and get OpenShift data
      command: oc get <resource> -o json
      register: oc_output
      changed_when: false

    - name: Parse 'oc' command output
      set_fact:
        oc_data: "{{ oc_output.stdout | from_json }}"

    - name: Compare properties and OpenShift data and set playbook variables
      set_fact:
        comparison_results: >-
          {{
            properties_dict | dict2items |
            map(attribute='key') |
            select("in", oc_data['items'] | map(attribute='metadata.name')) |
            list
          }}

    - name: Display comparison results
      debug:
        var: comparison_results
```

### Explanation:
1. **`slurp`**: Reads the properties file in base64 format.
2. **`b64decode` and `from_yaml`**: Decodes the content and parses it into a dictionary.
3. **`command`**: Runs an `oc` command to retrieve resource data from OpenShift in JSON format.
4. **`from_json`**: Parses JSON output into a dictionary.
5. **Comparison Logic**: Compares keys in the properties dictionary with items from the OpenShift output and stores the matching results.

### Notes:
- Ensure `oc` CLI is installed and configured with the necessary permissions.
- Modify `<resource>` in the `command` task based on the type of OpenShift resource you want to query (e.g., `pods`, `deployments`).
- You can expand the `comparison_results` logic to set playbook variables or flags based on specific conditions.

This approach will help you read properties, compare them with `oc` output, and set variables in your playbook based on the results.

**Swapan**: convert above to python ansible module

**ChatGPT**: Here's a basic structure for a custom Ansible module in Python that reads a properties file, runs `oc` commands, compares the data, and sets Ansible playbook variables:

1. **Read the properties file and parse it into a dictionary**.
2. **Run `oc` CLI commands**.
3. **Compare the properties with the `oc` output**.

### Python Ansible Module Code

Create a file named `read_compare_properties.py`:

```python
#!/usr/bin/python

from ansible.module_utils.basic import AnsibleModule
import subprocess
import json
import os

def read_properties_file(file_path):
    """Reads and parses a properties file into a dictionary."""
    properties = {}
    try:
        with open(file_path, 'r') as f:
            for line in f:
                line = line.strip()
                if line and not line.startswith('#'):
                    key, value = line.split('=', 1)
                    properties[key.strip()] = value.strip()
    except Exception as e:
        return {"error": f"Error reading properties file: {str(e)}"}
    return properties

def run_oc_command(resource):
    """Runs an 'oc' command to get data from OpenShift."""
    try:
        result = subprocess.run(
            ['oc', 'get', resource, '-o', 'json'],
            check=True,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True
        )
        return json.loads(result.stdout)
    except subprocess.CalledProcessError as e:
        return {"error": f"Error running 'oc' command: {e.stderr}"}
    except json.JSONDecodeError:
        return {"error": "Failed to parse JSON output from 'oc' command"}

def main():
    module_args = dict(
        properties_file_path=dict(type='str', required=True),
        resource=dict(type='str', required=True)
    )

    result = dict(
        changed=False,
        comparison_results=[],
        error=None
    )

    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True
    )

    # Read the properties file
    properties_file_path = module.params['properties_file_path']
    properties = read_properties_file(properties_file_path)

    if 'error' in properties:
        module.fail_json(msg=properties['error'])

    # Run the oc command
    resource = module.params['resource']
    oc_data = run_oc_command(resource)

    if 'error' in oc_data:
        module.fail_json(msg=oc_data['error'])

    # Compare properties with OpenShift data
    oc_names = [item['metadata']['name'] for item in oc_data.get('items', [])]
    comparison_results = [key for key in properties if key in oc_names]

    result['comparison_results'] = comparison_results

    module.exit_json(**result)

if __name__ == '__main__':
    main()
```

### Explanation:

1. **`read_properties_file`**: Reads and parses a properties file, returning a dictionary.
2. **`run_oc_command`**: Runs an `oc` command and returns its JSON output as a dictionary.
3. **Comparison Logic**: Compares the keys in the properties dictionary with the resource names from the OpenShift output.

### Usage:

1. Place `read_compare_properties.py` in your Ansible library path or roles directory.
2. Call it from an Ansible playbook:

   ```yaml
   - name: Read and compare properties file with OpenShift data
     hosts: localhost
     tasks:
       - name: Run custom module to read, compare, and set variables
         read_compare_properties:
           properties_file_path: "/path/to/your/properties/file.properties"
           resource: "deployments"
         register: comparison_output

       - name: Display comparison results
         debug:
           var: comparison_output.comparison_results
   ```

### Prerequisites:
- Ensure `oc` CLI is installed and configured.
- The module should have execution permissions (`chmod +x read_compare_properties.py`).

This custom module will help automate reading properties, comparing them with `oc` output, and returning the comparison results in an Ansible playbook.

