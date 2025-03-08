**Swapan**: ansible custom inventory to run play against all namespaces or set of namespaces in one openshift cluster

**ChatGPT**: To run an Ansible playbook against all namespaces or a subset of namespaces in an OpenShift cluster, you can create a custom dynamic inventory script or use Ansible's `openshift_raw` module with filters to fetch the namespaces dynamically. Here's how you can achieve it:

---

### **Steps to Create a Custom Inventory**

1. **Create the Inventory Script**
   Write a Python script (or use any language) to query the OpenShift API and fetch namespaces. The script should return data in JSON format, compliant with Ansible's dynamic inventory.

2. **Script Example: Fetch All Namespaces**
   Here's an example using Python and the OpenShift `oc` command to fetch all namespaces dynamically:

```python
#!/usr/bin/env python3
import subprocess
import json
import sys

def get_namespaces():
    # Execute `oc` command to fetch namespaces
    try:
        result = subprocess.run(
            ["oc", "get", "namespaces", "-o", "json"],
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True,
            check=True
        )
        data = json.loads(result.stdout)
        namespaces = [item["metadata"]["name"] for item in data["items"]]
        return namespaces
    except subprocess.CalledProcessError as e:
        print(f"Error fetching namespaces: {e.stderr}", file=sys.stderr)
        sys.exit(1)

def main():
    namespaces = get_namespaces()
    # Generate dynamic inventory
    inventory = {
        "all": {
            "hosts": namespaces,
            "vars": {}
        }
    }
    print(json.dumps(inventory, indent=2))

if __name__ == "__main__":
    main()
```

---

3. **Run Playbooks Against the Dynamic Inventory**
   - Save the script as `dynamic_inventory.py` and make it executable:
     ```bash
     chmod +x dynamic_inventory.py
     ```
   - Run your playbook using the dynamic inventory:
     ```bash
     ansible-playbook -i ./dynamic_inventory.py playbook.yml
     ```

---

### **Target a Subset of Namespaces**

To target specific namespaces, modify the script to include only namespaces matching specific criteria. For example:

```python
def get_namespaces(filter_prefix=None):
    result = subprocess.run(
        ["oc", "get", "namespaces", "-o", "json"],
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        text=True,
        check=True
    )
    data = json.loads(result.stdout)
    namespaces = [
        item["metadata"]["name"]
        for item in data["items"]
        if filter_prefix is None or item["metadata"]["name"].startswith(filter_prefix)
    ]
    return namespaces
```

To target namespaces with a specific prefix (e.g., "dev-"):

```bash
./dynamic_inventory.py --filter-prefix=dev-
```

---

### **Using OpenShift Ansible Modules**

Alternatively, you can use OpenShift modules like `k8s` or `openshift_raw` within a playbook to dynamically fetch namespaces:

```yaml
- name: Get all namespaces
  hosts: localhost
  tasks:
    - name: Retrieve namespaces
      k8s_info:
        api_version: v1
        kind: Namespace
      register: namespaces

    - name: Print namespace names
      debug:
        var: item.metadata.name
      loop: "{{ namespaces.resources }}"
```

Filter namespaces in the playbook using Jinja2:

```yaml
    - name: Filter namespaces with prefix
      set_fact:
        target_namespaces: >-
          {{ namespaces.resources | selectattr('metadata.name', 'search', '^dev-') | map(attribute='metadata.name') | list }}
```

---

### **Key Notes**
1. Ensure `oc` is installed and authenticated before running the inventory script.
2. If using the `k8s` module, install `kubernetes` Python library (`pip install openshift kubernetes`).
3. You can parameterize the namespace filters for flexibility when running the playbook.

Let me know if you'd like help tailoring this to your specific needs!

