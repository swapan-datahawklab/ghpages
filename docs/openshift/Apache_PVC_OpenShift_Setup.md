**Swapan**: apache webserver openshift to serve files from persistent volume claim\

**ChatGPT**: To configure an Apache webserver running in OpenShift to serve files from a Persistent Volume Claim (PVC), follow these steps:

---

### 1. **Create a Persistent Volume (PV) and Persistent Volume Claim (PVC)**

Ensure you have a PVC created in your OpenShift project. You can use a YAML definition like the following:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: apache-pvc
  namespace: <your-namespace>
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```

Apply this configuration:

```bash
oc apply -f pvc.yaml
```

---

### 2. **Create a Custom Apache Docker Image**
To serve files from a PVC, mount the PVC to the Apache document root. You can use a custom Apache Docker image or directly mount the volume if using a standard Apache image.

#### Dockerfile Example:
```dockerfile
FROM httpd:2.4
COPY ./httpd.conf /usr/local/apache2/conf/httpd.conf
```

#### Example `httpd.conf` Adjustments:
Set the `DocumentRoot` to point to the mounted PVC directory, e.g., `/var/www/html`.

```conf
DocumentRoot "/var/www/html"
<Directory "/var/www/html">
    AllowOverride None
    Require all granted
</Directory>
```

Build and push the image to a container registry accessible by OpenShift:

```bash
docker build -t <your-registry>/apache-pvc:latest .
docker push <your-registry>/apache-pvc:latest
```

---

### 3. **Deploy the Apache Server on OpenShift**

Create a deployment using your custom image or the standard Apache image, and mount the PVC.

#### Deployment YAML:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apache
  namespace: <your-namespace>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: apache
  template:
    metadata:
      labels:
        app: apache
    spec:
      containers:
      - name: apache
        image: <your-registry>/apache-pvc:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: apache-volume
          mountPath: /var/www/html
      volumes:
      - name: apache-volume
        persistentVolumeClaim:
          claimName: apache-pvc
```

Apply the deployment:

```bash
oc apply -f deployment.yaml
```

---

### 4. **Expose the Apache Service**

Create a service and route to expose the application:

#### Service YAML:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: apache-service
  namespace: <your-namespace>
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: apache
```

#### Route YAML:
```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: apache-route
  namespace: <your-namespace>
spec:
  to:
    kind: Service
    name: apache-service
  port:
    targetPort: 80
```

Apply the configurations:

```bash
oc apply -f service.yaml
oc apply -f route.yaml
```

---

### 5. **Test Your Setup**

1. Upload files to the PVC directory (via OpenShift UI or CLI).
2. Access your Apache server via the route URL created in OpenShift.
3. Files stored in the PVC should now be served by the Apache webserver.

---

This approach ensures that files stored in the PVC are persistently available and served by the Apache server running in OpenShift.

**Swapan**: do the docker build using the oc command

**ChatGPT**: You can use the OpenShift `oc` command to build and push your custom Apache image directly into the OpenShift internal image registry. Here's how:

---

### 1. **Prepare Your Source Code**

Ensure you have the following in your local directory:

- `Dockerfile` (to define your custom Apache setup)
- `httpd.conf` (optional, for custom Apache configuration)

---

### 2. **Create a BuildConfig in OpenShift**

You can use `oc new-build` to create a BuildConfig in OpenShift for building your image.

#### Run the Command:
```bash
oc new-build --name=apache-pvc --strategy=docker --binary
```

This creates a new BuildConfig named `apache-pvc` and prepares it to accept a local build context.

---

### 3. **Start the Build**

Run the following command to start the build with your local Docker context:

```bash
oc start-build apache-pvc --from-dir=. --follow
```

- `--from-dir=.`: Specifies the current directory as the build context.
- `--follow`: Streams the build logs to your terminal.

---

### 4. **Verify the Image**

Once the build is complete, OpenShift will push the image to its internal image registry. Verify the image:

```bash
oc get is
```

You should see an image stream named `apache-pvc` with the latest tag.

---

### 5. **Deploy the Image**

Now you can deploy the built image using a DeploymentConfig or Deployment.

#### Example Deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apache
  namespace: <your-namespace>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: apache
  template:
    metadata:
      labels:
        app: apache
    spec:
      containers:
      - name: apache
        image: image-registry.openshift-image-registry.svc:5000/<your-namespace>/apache-pvc:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: apache-volume
          mountPath: /var/www/html
      volumes:
      - name: apache-volume
        persistentVolumeClaim:
          claimName: apache-pvc
```

Apply the deployment:

```bash
oc apply -f deployment.yaml
```

---

### 6. **Expose the Service**

Expose your deployment with a service and route:

```bash
oc expose deployment apache --port=80
oc expose svc/apache
```

This will create a route to access your Apache server.

---

### 7. **Access Your Application**

Get the route URL and test it:

```bash
oc get routes
```

Use the route URL to verify your Apache server is serving files from the PVC.

