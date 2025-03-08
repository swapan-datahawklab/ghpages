**Swapan**: K0s multi node cluster install haproxy spring boot postgres ingres json file

**ChatGPT**: **Clarifying the setup**

Hmm, I’m thinking about how the user wants to set up a K0s multi-node cluster with HAProxy, Spring Boot, Postgres, and an ingress configuration. Clarification is needed to proceed.

**ChatGPT**: Below is a high-level guide to help you set up a **K0s multi-node cluster** with **HAProxy** as a load balancer, deploy a **Spring Boot** application that connects to **PostgreSQL**, and configure an **Ingress** (shown as a JSON example). 

> **Note**: This is a broad outline and includes multiple components. Depending on your environment and specific use case, you may need to adapt steps accordingly.

---

## 1. Prepare the Environment

1. **Provision servers/VMs** (or use cloud instances):
   - At least one **controller** node for K0s.
   - One or more **worker** nodes for running workloads.
   - One node or external machine for **HAProxy** (or you can install HAProxy on the controller node if you wish, but that’s less common in production).

2. **Install necessary tools** on each node:
   - **kubectl** (on your management machine or on the controller node).
   - **k0s** (on controller and worker nodes).
   - **docker** or **containerd** (k0s bundles containerd by default, but you can use an external runtime if desired).

3. **Make sure your firewall** rules are adjusted so that:
   - Nodes can communicate on the necessary ports (especially ports used by K0s, typically 6443 for the API server, etc.).
   - HAProxy can forward traffic to your Ingress or NodePort services.

---

## 2. Install and Configure K0s

> **Documentation**: [K0s Documentation](https://docs.k0sproject.io/)

### 2.1 Install K0s on the Controller Node
1. **Download K0s**:
   ```bash
   curl -sSLf https://get.k0s.sh | sudo sh
   ```
2. **Initialize K0s as a controller**:
   ```bash
   sudo k0s install controller --single
   ```
   > If you want a high-availability setup (multi-controller), omit `--single` and follow the HA instructions. For a single controller example, `--single` is sufficient.
3. **Start K0s**:
   ```bash
   sudo k0s start
   ```
4. **Get the join token** so workers can join:
   ```bash
   sudo k0s token create --role=worker
   ```
   > Save this token; you will need it for your worker nodes.

### 2.2 Install K0s on Worker Nodes
1. **Install K0s**:
   ```bash
   curl -sSLf https://get.k0s.sh | sudo sh
   ```
2. **Join the cluster**:
   ```bash
   sudo k0s install worker --token=<worker-join-token-from-controller>
   sudo k0s start
   ```
3. **Verify** from the controller that the worker has joined:
   ```bash
   sudo k0s kubectl get nodes
   ```
   > You should see your worker node(s) in `Ready` status after a few minutes.

---

## 3. Set Up HAProxy (Load Balancer)

> **Purpose**: HAProxy can act as a frontend load balancer to direct traffic to the K0s cluster’s Ingress or NodePort. In a simple setup, you might run HAProxy outside the cluster on a separate machine or the controller node.

### 3.1 Install HAProxy

On a Debian/Ubuntu-based system:
```bash
sudo apt-get update
sudo apt-get install -y haproxy
```

### 3.2 Configure HAProxy

A minimal HAProxy configuration might look like this (usually in `/etc/haproxy/haproxy.cfg`):

```cfg
global
    log /dev/log    local0
    log /dev/log    local1 notice
    daemon
    maxconn 4096

defaults
    mode http
    log global
    option httplog
    option dontlognull
    option forwardfor
    timeout connect 5000
    timeout client  10000
    timeout server  10000

frontend http_front
    bind *:80
    default_backend k0s_backend

backend k0s_backend
    balance roundrobin
    # Replace IPs/ports with your node addresses (or your Ingress controllers).
    server worker1 <WORKER1_IP>:<INGRESS_PORT> check
    server worker2 <WORKER2_IP>:<INGRESS_PORT> check
```

- **\<INGRESS_PORT\>** is typically **80** if you’re using an Ingress controller that listens on 80 for HTTP, or **443** for HTTPS. 
- For a NodePort service, you would specify the NodePort number instead.

Restart HAProxy:
```bash
sudo systemctl restart haproxy
```

---

## 4. Deploy PostgreSQL in K0s

You can deploy PostgreSQL in multiple ways:
- Use a [Helm chart](https://github.com/bitnami/charts/tree/main/bitnami/postgresql) for quick setup.
- Or use raw YAML manifests. Below is a simple example with a **StatefulSet** and **Service** for Postgres.

**postgresql.yaml** (example):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: "postgres"
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: mydatabase
            - name: POSTGRES_USER
              value: myuser
            - name: POSTGRES_PASSWORD
              value: mypassword
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: postgres-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: "default"
        resources:
          requests:
            storage: 2Gi
```

Apply the manifest:
```bash
kubectl apply -f postgresql.yaml
```

Once this is deployed, you can verify:
```bash
kubectl get pods
kubectl get svc
```
You should see a `postgres` Service listening on port 5432 (cluster-internal).

---

## 5. Deploy Spring Boot Application on K0s

### 5.1 Build and Push a Spring Boot Docker Image
1. In your Spring Boot project, ensure you have a `Dockerfile` similar to:
   ```dockerfile
   FROM openjdk:17-jdk-slim
   COPY target/myapp-0.0.1-SNAPSHOT.jar /app.jar
   EXPOSE 8080
   ENTRYPOINT ["java","-jar","/app.jar"]
   ```
2. **Build** the jar (Maven or Gradle), then build the Docker image:
   ```bash
   mvn clean package  # or gradle build
   docker build -t <your-registry>/my-spring-boot-app:latest .
   ```
3. **Push** to a registry accessible by your K0s nodes:
   ```bash
   docker push <your-registry>/my-spring-boot-app:latest
   ```

### 5.2 Create a Kubernetes Deployment and Service

**spring-boot.yaml** (example):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring-boot-app
  template:
    metadata:
      labels:
        app: spring-boot-app
    spec:
      containers:
        - name: spring-boot-app
          image: <your-registry>/my-spring-boot-app:latest
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_DATASOURCE_URL
              value: jdbc:postgresql://postgres:5432/mydatabase
            - name: SPRING_DATASOURCE_USERNAME
              value: myuser
            - name: SPRING_DATASOURCE_PASSWORD
              value: mypassword
            # Add any other environment variables your app needs

---
apiVersion: v1
kind: Service
metadata:
  name: spring-boot-service
spec:
  type: ClusterIP
  selector:
    app: spring-boot-app
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
      name: http
```

Apply the manifest:
```bash
kubectl apply -f spring-boot.yaml
```

Check deployment status:
```bash
kubectl get pods
kubectl logs <spring-boot-pod-name>
```
You should see logs from your Spring Boot app indicating it’s running and connected to Postgres.

---

## 6. Set Up Ingress (as JSON)

Most Ingress controllers (e.g. NGINX Ingress Controller) require a YAML definition. If you need it in **JSON** format, you can either:
- Provide the same object structure, but in JSON.
- Or use a tool like `yq` to convert from YAML to JSON.

Below is an **example Ingress** object in JSON format. It routes traffic on `myapp.example.com` to the `spring-boot-service` on port 80.

> **Important**: Make sure you have an Ingress Controller installed in your K0s cluster (e.g., the NGINX Ingress Controller). K0s does not install one by default. You can install it via Helm or the official manifests.

### 6.1 Example Ingress JSON
```json
{
  "apiVersion": "networking.k8s.io/v1",
  "kind": "Ingress",
  "metadata": {
    "name": "spring-boot-ingress",
    "annotations": {
      "kubernetes.io/ingress.class": "nginx"
    }
  },
  "spec": {
    "rules": [
      {
        "host": "myapp.example.com",
        "http": {
          "paths": [
            {
              "path": "/",
              "pathType": "Prefix",
              "backend": {
                "service": {
                  "name": "spring-boot-service",
                  "port": {
                    "number": 80
                  }
                }
              }
            }
          ]
        }
      }
    ]
  }
}
```

You can apply it with:
```bash
kubectl apply -f spring-boot-ingress.json
```

---

## 7. Validate the Setup

1. **DNS** or **/etc/hosts**: Make sure `myapp.example.com` resolves to the **HAProxy** IP or to one of your K0s nodes (depending on how you set up the routing).
2. **Traffic flow**: 
   - HAProxy listens on port 80.
   - Forwards to worker nodes on port 80 (the Ingress controller’s Service).
   - The Ingress routes `myapp.example.com/` to the `spring-boot-service`.
3. **Spring Boot** logs: Confirm the application is receiving requests, and that it can connect to PostgreSQL without errors.
4. **PostgreSQL** logs: Check for successful connections.

---

## 8. Production Considerations

1. **High Availability**:
   - Run multiple K0s controllers (HA) or replicate the controller in a multi-node setup.
   - Use multiple worker nodes for load balancing and failover.
   - Have multiple HAProxy instances (or a more advanced load balancer) for production.
2. **Data Persistence**:
   - Ensure your Postgres volume has persistent storage (e.g., a proper StorageClass with dynamic provisioning).
3. **Security**:
   - Use SSL/TLS for your Ingress (HTTPS).
   - Secure your Postgres with stronger passwords, or connect via SSL inside the cluster.
   - Restrict inbound and outbound firewall rules as necessary.
4. **Observability**:
   - Consider installing monitoring and logging (Prometheus, Grafana, ELK/EFK) to keep track of cluster and application health.

---

## Summary

By following these steps:

1. **Deploy a K0s multi-node cluster** (controller + workers).  
2. **Set up HAProxy** to distribute traffic to the cluster.  
3. **Deploy PostgreSQL** as a StatefulSet and Service.  
4. **Build & deploy a Spring Boot application** connecting to Postgres.  
5. **Set up an Ingress** in JSON format (or YAML converted to JSON).  

…you will have a working environment with load balancing, a backend database, and a Spring Boot application, all running on a K0s multi-node Kubernetes cluster. 

Feel free to adapt the YAML and JSON snippets as needed for your environment.

**Swapan**: deploy haproxy in k0s

**ChatGPT**: **Detailing HAProxy deployment**

I'm thinking through step-by-step deployment of HAProxy in K0s. Considering Deployment or DaemonSet for moderate load, leveraging ConfigMap for configuration, and setting up a Service for external access.

**Shaping the HAProxy config**

I'm sketching out the HAProxy configuration with global and default settings, front-end, and back-end sections. This setup will be stored in a ConfigMap for concise deployment.

**ChatGPT**: Below is a **basic example** of how you might deploy HAProxy **inside** a K0s (Kubernetes) cluster. This setup includes:

1. A **ConfigMap** for HAProxy configuration.  
2. A **Deployment** to run the HAProxy container.  
3. A **Service** (NodePort) to expose HAProxy.

You can adapt this manifest according to your production requirements (e.g., using LoadBalancer, custom HAProxy configuration, TLS, etc.).

---

## 1. Create a ConfigMap for HAProxy Configuration

Save this as `haproxy-configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: haproxy-config
  labels:
    app: haproxy
data:
  haproxy.cfg: |
    global
        maxconn 4096
        log stdout format raw local0

    defaults
        mode http
        log global
        option dontlognull
        option http-server-close
        option forwardfor
        timeout connect 5000
        timeout client  50000
        timeout server  50000

    frontend http_front
        bind *:80
        default_backend test_backend

    backend test_backend
        balance roundrobin
        # Example: forward traffic to an internal service
        # Adjust "example-service" and port to match your real service
        server srv1 example-service.default.svc.cluster.local:8080 check
```

> In this sample configuration, the backend is `example-service.default.svc.cluster.local:8080`.  
> Modify it to point to whatever Service your HAProxy should proxy to.  
> The frontend listens on port **80** inside the container.

---

## 2. Create the HAProxy Deployment

Save this as `haproxy-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: haproxy
  labels:
    app: haproxy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: haproxy
  template:
    metadata:
      labels:
        app: haproxy
    spec:
      containers:
      - name: haproxy
        image: haproxy:2.6
        ports:
        - containerPort: 80
        volumeMounts:
        - name: haproxy-config-volume
          mountPath: /usr/local/etc/haproxy/haproxy.cfg
          subPath: haproxy.cfg
      volumes:
      - name: haproxy-config-volume
        configMap:
          name: haproxy-config
          items:
            - key: haproxy.cfg
              path: haproxy.cfg
```

### Notes

- **image**: `haproxy:2.6` is an example. You can use other versions or custom images.
- We mount the file **`haproxy.cfg`** from the **ConfigMap** into **`/usr/local/etc/haproxy/haproxy.cfg`** in the container.
- We expose **port 80** in the container to match the “bind \*:80” directive in the HAProxy config.

---

## 3. Create a Service (NodePort or LoadBalancer)

To expose your HAProxy Deployment within or outside the cluster, create a Service. Below is a **NodePort** example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: haproxy-service
  labels:
    app: haproxy
spec:
  type: NodePort
  selector:
    app: haproxy
  ports:
    - name: http
      port: 80        # The port inside the cluster (ClusterIP)
      targetPort: 80  # The container port
      nodePort: 30080 # The port you want on your node(s) (range usually 30000-32767)
```

### Important
- **`type: NodePort`** means you can access HAProxy via `http://<node-ip>:30080`.
- If you are in a cloud environment supporting LoadBalancer Services, you could do `type: LoadBalancer` and remove `nodePort: 30080`. Then the cloud provider (if supported) allocates an external IP.
- If you only want HAProxy available internally (ClusterIP only), you can set `type: ClusterIP`. Then other Pods/Services in the cluster can reach HAProxy via `haproxy-service:80`, but you won’t have external access.

Save this as `haproxy-service.yaml`.

---

## 4. Apply Manifests to K0s

Assuming you have a working `kubectl` context to your K0s cluster, apply the files in this order:

```bash
kubectl apply -f haproxy-configmap.yaml
kubectl apply -f haproxy-deployment.yaml
kubectl apply -f haproxy-service.yaml
```

---

## 5. Validate the Deployment

1. **Check Pods**:
   ```bash
   kubectl get pods -l app=haproxy
   ```
   You should see the `haproxy` Pod in **Running** state.

2. **Check Logs**:
   ```bash
   kubectl logs <haproxy-pod-name>
   ```
   You should see HAProxy start-up logs.

3. **Access via NodePort** (if used):
   - On any node in the cluster, retrieve the node’s IP (e.g. `worker1` IP).
   - Browse or curl to `http://<node-ip>:30080/`.
   - You should see traffic routed to the backend (`example-service.default.svc.cluster.local:8080` in the example).

---

## 6. Customizing HAProxy

- **Backend Targets**: In the sample config, we used `example-service.default.svc.cluster.local:8080` for demonstration. Change it to your actual internal Service (or external endpoints if you prefer).
- **Multiple Frontends**: If you want HAProxy to listen on multiple ports or handle HTTPS, add additional frontends (and bind statements).
- **TLS/SSL**: For HTTPS support, create or mount certificates into the container, update the HAProxy config, and bind port 443 with SSL settings.
- **Scaling**: Increase `replicas` in the Deployment if you want multiple HAProxy pods behind another Service, or run them on different nodes.

---

## 7. Production Considerations

1. **High Availability**: Run multiple HAProxy replicas behind a Kubernetes Service of type LoadBalancer or NodePort with an external load balancer in front.  
2. **Configuration Updates**: When you update the ConfigMap, Pods will need to be restarted or reloaded to pick up changes (HAProxy can do a hot reload if you manage the process properly, but that typically requires advanced setups).  
3. **Ingress**: If you’re using K0s and want a typical “Kubernetes Ingress,” you might prefer a dedicated Ingress controller (e.g., NGINX, HAProxy Ingress Controller, or Traefik). The example above is purely deploying HAProxy as a standalone proxy container.  
4. **Observability**: Set up logging and metrics. HAProxy can export stats via its stats socket or stats web interface.  

---

### Quick Recap

1. **Create a ConfigMap** with your HAProxy config.  
2. **Deploy** HAProxy as a Deployment (or DaemonSet if you need it on every node).  
3. **Expose** it via a Service (NodePort, LoadBalancer, or ClusterIP).  
4. **Test** by sending traffic to the node IP or load balancer IP.  

That’s it! You have a minimal HAProxy instance running **inside** your K0s cluster. Adjust the configuration and scaling based on your environment and production needs.

