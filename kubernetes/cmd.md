# Kubernetes Practical Tasks

Environment: local `kind` cluster.

---

## Run Apache Pod

**Task:** Create and run an Apache Pod using image `httpd`, pod name `apache-pod`, and verify it.

```bash
kubectl run apache-pod --image=httpd
kubectl get pods
kubectl describe pod apache-pod
```

---

## Apache Deployment

**Task:** Create a deployment `apache-deployment` using image `httpd` and verify it.

```bash
kubectl create deployment apache-deployment --image=httpd
kubectl get deployments
kubectl get pods
```

---

## Increase and Decrease Replicas

**Task:** Scale the Apache Deployment to 4 replicas, verify ReplicaSet and Pods, then scale down to 2 and verify.

```bash
# Increase to 4
kubectl scale deployment apache-deployment --replicas=4
kubectl get rs
kubectl get pods

# Decrease to 2
kubectl scale deployment apache-deployment --replicas=2
kubectl get rs
kubectl get pods
```

---

## Use YAML File

**Task:** Write a YAML file for Apache Deployment using image `httpd` with 3 replicas, apply it, and verify all resources.

`apache-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apache-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: apache-deployment
  template:
    metadata:
      labels:
        app: apache-deployment
    spec:
      containers:
        - name: httpd
          image: httpd
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f apache-deployment.yaml
kubectl get all
```

---

## ClusterIP Service

**Task:** Expose Apache Deployment using a ClusterIP service `apache-clusterip` on port 80 and test access inside the cluster.

```bash
kubectl expose deployment apache-deployment --name=apache-clusterip --type=ClusterIP --port=80 --target-port=80
kubectl get svc apache-clusterip

# Test from inside the cluster
kubectl run test-pod --rm -it --image=busybox --restart=Never -- wget -qO- http://apache-clusterip
```

Expected output: `<html><body><h1>It works!</h1></body></html>`

---

## NodePort Service

**Task:** Expose Apache Deployment using a NodePort service `apache-nodeport` on port 80 and access it from a browser.

```bash
kubectl expose deployment apache-deployment --name=apache-nodeport --type=NodePort --port=80 --target-port=80
kubectl get svc apache-nodeport

# Node IP
kubectl get nodes -o wide
```

Browser: `http://<node-ip>:<node-port>`

---

## LoadBalancer Service

**Task:** Expose Apache Deployment using a LoadBalancer service `apache-loadbalancer` on port 80 and verify the external IP.

```bash
kubectl expose deployment apache-deployment --name=apache-loadbalancer --type=LoadBalancer --port=80 --target-port=80
kubectl get svc
```

On `kind`, `EXTERNAL-IP` stays `<pending>` unless a load balancer provider is running. Run `cloud-provider-kind` in a separate terminal, then re-run `kubectl get svc`.

---

## Flask Application Deployment

**Task:** Create a Flask app displaying PRN number and student name, containerize it, deploy it to Kubernetes, expose it with a Service, and access it in a browser.

`app.py`

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return """
    <h1>PRN: 123456</h1>
    <h1>Name: Student Name</h1>
    """

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

`Dockerfile`

```dockerfile
FROM python:3.12-slim
WORKDIR /app
RUN pip install flask
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
```

```bash
# Build image
docker build -t flask-app:v1 .

# Load image into kind cluster
kind load docker-image flask-app:v1 --name mycluster

# Deploy
kubectl create deployment flask-deployment --image=flask-app:v1
kubectl get pods

# Expose
kubectl expose deployment flask-deployment --name=flask-service --type=NodePort --port=5000 --target-port=5000
kubectl get svc flask-service

# Access in browser
kubectl port-forward svc/flask-service 5000:5000
```

Browser: `http://localhost:5000` (or `http://<node-ip>:<node-port>`)

---

## Rolling Update Flask Application

**Task:** Update the Flask app output to include PRN number, student name, and course name; build a new image version; perform a rolling update and verify.

Update `app.py`

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return """
    <h1>PRN: 123456</h1>
    <h1>Name: Student Name</h1>
    <h1>Course: Course Name</h1>
    """

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

```bash
# Build new version and load into kind
docker build -t flask-app:v2 .
kind load docker-image flask-app:v2 --name mycluster

# Rolling update
kubectl set image deployment/flask-deployment flask-app=flask-app:v2

# Verify
kubectl rollout status deployment/flask-deployment
kubectl get pods

# Access updated output
kubectl port-forward svc/flask-service 5000:5000
```

Browser: `http://localhost:5000`

Rollback (if needed):

```bash
kubectl rollout undo deployment/flask-deployment
```
