**Swapan**: oc ansible wrapper to get describe info and resources by pod grouped by label 

**ChatGPT**: Creating an Ansible wrapper to retrieve information and resources for pods grouped by label using OpenShift’s `oc` command can help automate and streamline your tasks. Here’s an outline of an Ansible playbook that does this:

### Step 1: Define your inventory and prerequisites

Make sure you have:
- An inventory file specifying your OpenShift host.
- Necessary privileges to run `oc` commands and access the OpenShift API.

### Step 2: Write the Ansible Playbook

The playbook below uses an Ansible shell task to filter pods based on a specific label and then retrieve the `oc describe` information and associated resources for each pod.

```yaml
---
- name: Retrieve pod details and associated resources
  hosts: openshift_nodes
  gather_facts: no
  vars:
    label_selector: "your-label-selector"  # e.g., app=myapp
    namespace: "your-namespace"  # Namespace where the pods are located

  tasks:
    - name: Get list of pods by label
      shell: |
        oc get pods -n {{ namespace }} -l {{ label_selector }} -o jsonpath='{.items[*].metadata.name}'
      register: pods_output

    - name: Display pod details
      debug:
        msg: "{{ pods_output.stdout_lines }}"

    - name: Get detailed information for each pod
      shell: |
        oc describe pod {{ item }} -n {{ namespace }}
      loop: "{{ pods_output.stdout_lines }}"
      register: pod_descriptions

    - name: Display pod descriptions
      debug:
        var: pod_descriptions

    - name: Get associated resources for each pod
      shell: |
        oc get all -n {{ namespace }} --selector={{ label_selector }}
      register: associated_resources

    - name: Display associated resources
      debug:
        var: associated_resources.stdout
```

### Explanation

1. **Get list of pods by label**: Uses `oc get pods` with `-l` to filter by label and `-o jsonpath` to retrieve only the pod names.
2. **Describe each pod**: Loops over each pod to retrieve detailed information using `oc describe pod`.
3. **Get associated resources**: Uses `oc get all` to retrieve all resources in the namespace that match the label.

### Execution

Run this playbook with:

```bash
ansible-playbook -i inventory.yaml your-playbook.yml
```

Replace `inventory.yaml` with your actual inventory file and `your-playbook.yml` with the filename for the playbook.

This wrapper will give you a complete description of each pod as well as associated resources in the specified namespace that match your label selector.

**Swapan**: ansible oc wrapper module for openshift to get describe info and resources by pod grouped by label

**ChatGPT**: To create an Ansible module for OpenShift that acts as a wrapper for retrieving information and resources based on a pod’s label, you can create a custom Ansible module in Python. This module will execute `oc` commands to get `describe` information and associated resources based on a given label selector.

Here's how you can build it.

### Step 1: Create the Custom Ansible Module

Create a Python script (e.g., `oc_pod_info.py`) to be used as a custom Ansible module.

```python
#!/usr/bin/python

from ansible.module_utils.basic import AnsibleModule
import subprocess
import json

def run_oc_command(command):
    """ Helper function to run oc commands and handle errors """
    try:
        result = subprocess.run(command, shell=True, check=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
        return result.stdout.decode("utf-8").strip()
    except subprocess.CalledProcessError as err:
        return err.stderr.decode("utf-8").strip()

def main():
    module = AnsibleModule(
        argument_spec=dict(
            namespace=dict(type="str", required=True),
            label_selector=dict(type="str", required=True),
        ),
        supports_check_mode=True
    )

    namespace = module.params["namespace"]
    label_selector = module.params["label_selector"]

    # Step 1: Get list of pods by label
    pods_command = f"oc get pods -n {namespace} -l {label_selector} -o json"
    pods_output = run_oc_command(pods_command)

    try:
        pods = json.loads(pods_output)["items"]
    except json.JSONDecodeError:
        module.fail_json(msg="Failed to parse pod list JSON")

    pod_descriptions = {}
    for pod in pods:
        pod_name = pod["metadata"]["name"]

        # Step 2: Describe each pod
        describe_command = f"oc describe pod {pod_name} -n {namespace}"
        describe_output = run_oc_command(describe_command)
        pod_descriptions[pod_name] = describe_output

    # Step 3: Get associated resources based on label
    resources_command = f"oc get all -n {namespace} -l {label_selector} -o json"
    resources_output = run_oc_command(resources_command)

    # Return results
    module.exit_json(
        changed=False,
        pod_descriptions=pod_descriptions,
        associated_resources=json.loads(resources_output)
    )

if __name__ == "__main__":
    main()
```

### Explanation of the Code

1. **run_oc_command**: A helper function to execute `oc` commands and handle errors.
2. **main function**: The main logic of the module, which:
   - Retrieves pod names based on the label selector.
   - Describes each pod and gathers information.
   - Fetches associated resources using the same label selector.

### Step 2: Use the Custom Module in a Playbook

After creating the custom module, you can use it within an Ansible playbook as follows:

```yaml
---
- name: Get describe info and resources for pods by label
  hosts: localhost
  tasks:
    - name: Retrieve pod describe info and associated resources
      oc_pod_info:
        namespace: "your-namespace"
        label_selector: "app=myapp"
      register: pod_info

    - name: Display pod descriptions
      debug:
        var: pod_info.pod_descriptions

    - name: Display associated resources
      debug:
        var: pod_info.associated_resources
```

### Step 3: Running the Playbook

Run the playbook using:

```bash
ansible-playbook -i localhost, playbook.yml
```

This module and playbook will give you the `describe` information for each pod grouped by label as well as associated resources in the specified namespace, returned as structured output in Ansible. Make sure to adjust `namespace` and `label_selector` to match your requirements.

