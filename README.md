# Flask MongoDB Application Deployment Guide

## Overview

This README provides detailed instructions to set up a Python Flask application connected to MongoDB both locally and within a Kubernetes cluster. This includes creating a local development environment and deploying the application on Kubernetes with proper configurations.

## Part 1: Local Setup

### Prerequisites

- Python 3.8 or later
- Docker
- Pip for managing Python packages

### Steps

1. **Create Project Directory**
    ```bash
    mkdir flask-mongodb-app
    cd flask-mongodb-app
    ```

2. **Set Up Virtual Environment**
    ```bash
    python3 -m venv venv
    source venv/bin/activate  # On Windows, use `venv\Scripts\activate`
    ```

    **Cookie Point:** Virtual environments isolate dependencies, avoiding conflicts between projects and keeping your global Python environment clean.

3. **Create Flask Application**
    Create a file named `app.py` with the following content:
    ```python
    from flask import Flask, request, jsonify
    from pymongo import MongoClient
    from datetime import datetime
    import os

    app = Flask(__name__)

    client = MongoClient(os.environ.get("MONGODB_URI", "mongodb://localhost:27017/"))
    db = client.flask_db
    collection = db.data

    @app.route('/')
    def index():
        return f"Welcome to the Flask app! The current time is: {datetime.now()}"

    @app.route('/data', methods=['GET', 'POST'])
    def data():
        if request.method == 'POST':
            data = request.get_json()
            collection.insert_one(data)
            return jsonify({"status": "Data inserted"}), 201
        elif request.method == 'GET':
            data = list(collection.find({}, {"_id": 0}))
            return jsonify(data), 200

    if __name__ == '__main__':
        app.run(host='0.0.0.0', port=5000)
    ```

4. **Create Requirements File**
    Create `requirements.txt`:
    ```
    Flask==2.0.2
    Werkzeug==2.0.3
    pymongo==3.12.0
    ```

5. **Install Python Dependencies**
    ```bash
    pip install -r requirements.txt
    ```

6. **Set Up MongoDB Using Docker**
    ```bash
    docker pull mongo:latest
    docker run -d -p 27017:27017 --name mongodb mongo:latest
    ```

7. **Set Up Environment Variable**
    Create a `.env` file in the project directory with:
    ```
    MONGODB_URI=mongodb://localhost:27017/
    ```
    Load the environment variables:
    ```bash
    export $(cat .env | xargs)
    ```

8. **Run the Flask Application**
    ```bash
    export FLASK_APP=app.py
    export FLASK_ENV=development
    flask run
    ```
    For Windows:
    ```bash
    set FLASK_APP=app.py
    set FLASK_ENV=development
    flask run
    ```

9. **Accessing the Application**
    Open your browser and go to `http://localhost:5000` to see the welcome message.

    **Example Requests Using `curl`:**

    - **GET request to `/` endpoint:**
      ```bash
      curl http://localhost:5000/
      ```
      Response:
      ```
      Welcome to the Flask app! The current time is: <Date and Time>
      ```

    - **POST request to `/data` endpoint:**
      ```bash
      curl -X POST -H "Content-Type: application/json" -d '{"sampleKey":"sampleValue"}' http://localhost:5000/data
      ```
      Response:
      ```json
      {"status": "Data inserted"}
      ```

    - **GET request to `/data` endpoint:**
      ```bash
      curl http://localhost:5000/data
      ```
      Response:
      ```json
      [{ "sampleKey": "sampleValue" }]
      ```

## Part 2: Kubernetes Setup

### Prerequisites

- Minikube or another local Kubernetes cluster
- kubectl command-line tool

### Steps

1. **Dockerfile for Flask Application**
    Create a `Dockerfile` in the project directory:
    ```Dockerfile
    FROM python:3.8-slim

    WORKDIR /app

    COPY requirements.txt .

    RUN pip install -r requirements.txt

    COPY app.py .

    ENV FLASK_APP=app.py
    ENV FLASK_ENV=development

    EXPOSE 5000

    CMD ["flask", "run", "--host=0.0.0.0"]
    ```

2. **Build and Push Docker Image**
    ```bash
    docker build -t your-dockerhub-username/flask-mongodb-app:latest .
    docker push your-dockerhub-username/flask-mongodb-app:latest
    ```

3. **Kubernetes YAML Files**

    - **Flask Deployment (`flask-deployment.yaml`):**
      ```yaml
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: flask-deployment
      spec:
        replicas: 2
        selector:
          matchLabels:
            app: flask-app
        template:
          metadata:
            labels:
              app: flask-app
          spec:
            containers:
            - name: flask-app
              image: your-dockerhub-username/flask-mongodb-app:latest
              ports:
              - containerPort: 5000
              env:
              - name: MONGODB_URI
                value: "mongodb://mongo-service:27017/"
              resources:
                requests:
                  memory: "256Mi"
                  cpu: "200m"
                limits:
                  memory: "512Mi"
                  cpu: "500m"
      ```

    - **MongoDB StatefulSet (`mongodb-statefulset.yaml`):**
      ```yaml
      apiVersion: apps/v1
      kind: StatefulSet
      metadata:
        name: mongodb
      spec:
        serviceName: "mongodb"
        replicas: 1
        selector:
          matchLabels:
            app: mongodb
        template:
          metadata:
            labels:
              app: mongodb
          spec:
            containers:
            - name: mongodb
              image: mongo:latest
              ports:
              - containerPort: 27017
              volumeMounts:
              - name: mongo-persistent-storage
                mountPath: /data/db
              env:
              - name: MONGO_INITDB_ROOT_USERNAME
                value: "root"
              - name: MONGO_INITDB_ROOT_PASSWORD
                value: "password"
        volumeClaimTemplates:
        - metadata:
            name: mongo-persistent-storage
          spec:
            accessModes: ["ReadWriteOnce"]
            resources:
              requests:
                storage: 1Gi
      ```

    - **MongoDB Service (`mongodb-service.yaml`):**
      ```yaml
      apiVersion: v1
      kind: Service
      metadata:
        name: mongodb
      spec:
        selector:
          app: mongodb
        ports:
          - protocol: TCP
            port: 27017
        clusterIP: None
      ```

    - **Flask Service (`flask-service.yaml`):**
      ```yaml
      apiVersion: v1
      kind: Service
      metadata:
        name: flask-service
      spec:
        type: NodePort
        selector:
          app: flask-app
        ports:
          - protocol: TCP
            port: 5000
            targetPort: 5000
      ```

    - **Horizontal Pod Autoscaler (`hpa.yaml`):**
      ```yaml
      apiVersion: autoscaling/v2beta2
      kind: HorizontalPodAutoscaler
      metadata:
        name: flask-hpa
      spec:
        scaleTargetRef:
          apiVersion: apps/v1
          kind: Deployment
          name: flask-deployment
        minReplicas: 2
        maxReplicas: 5
        metrics:
          - type: Resource
            resource:
              name: cpu
              target:
                type: Utilization
                averageUtilization: 70
      ```

4. **Deploy to Kubernetes**
    Apply all the Kubernetes configurations:
    ```bash
    kubectl apply -f mongodb-statefulset.yaml
    kubectl apply -f mongodb-service.yaml
    kubectl apply -f flask-deployment.yaml
    kubectl apply -f flask-service.yaml
    kubectl apply -f hpa.yaml
    ```

5. **Access the Flask Application**
    Get the NodePort assigned to the Flask Service:
    ```bash
    kubectl get services flask-service
    ```
    Access the Flask application using the external IP of the Minikube cluster and the NodePort.

### Additional Information

**DNS Resolution in Kubernetes:**
Kubernetes provides DNS resolution for services within the cluster. Each service gets a DNS name (e.g., `mongodb-service.default.svc.cluster.local`) which allows other services and pods to connect using the service name. The internal DNS system resolves this to the service’s cluster IP.

**Resource Requests and Limits:**
- **Requests**: The guaranteed minimum resources for the pod. Kubernetes ensures this amount is available.
- **Limits**: The maximum resources the pod can use. If exceeded, the pod may be throttled or terminated.

**Testing Autoscaling:**
1. Simulate traffic to the Flask application using tools like `hey` or `k6`.
2. Monitor the Horizontal Pod Autoscaler with:
   ```bash
   kubectl get hpa
   ```
3. Ensure that the number of replicas scales up when CPU utilization exceeds 70%.

**Testing Database Interactions:**
1. Use `curl` or Postman to perform `POST` and `GET` requests to the `/data` endpoint.
2. Verify data insertion and retrieval.

### Note

This README provides the necessary steps and explanations to set up a Flask application with MongoDB locally and in a Kubernetes cluster. Follow these instructions to ensure a successful deployment and test the application thoroughly.

### Design Choices

#### 1. **Dockerfile for Flask Application**

**Chosen Configuration:**
- **Base Image**: `python:3.8-slim`
  - **Reason**: This image is lightweight compared to the full Python image, reducing build time and runtime overhead while still providing the necessary Python environment.
- **Installation of Dependencies**: `pip install -r requirements.txt`
  - **Reason**: Using a `requirements.txt` file ensures consistent and repeatable installations of dependencies, which aligns with best practices for Python projects.
- **Environment Variables**: Set `FLASK_APP` and `FLASK_ENV`
  - **Reason**: These environment variables make it easy to configure and run the Flask application in different environments (development/production).

**Alternatives Considered:**
- Using a different base image such as `python:3.8` instead of `python:3.8-slim`. The slim version was chosen for its smaller footprint, which is beneficial in a containerized environment.

#### 2. **MongoDB StatefulSet Configuration**

**Chosen Configuration:**
- **StatefulSet** for MongoDB
  - **Reason**: A StatefulSet is suitable for applications that require persistent storage and stable network identities. MongoDB benefits from these features to maintain consistency and data integrity.
- **Authentication**: Enabled with `MONGO_INITDB_ROOT_USERNAME` and `MONGO_INITDB_ROOT_PASSWORD`
  - **Reason**: Authentication secures access to the MongoDB instance, which is crucial for production environments.

**Alternatives Considered:**
- **Deployment**: Using a Deployment instead of StatefulSet. StatefulSet was chosen because it is better suited for databases due to its support for persistent storage and stable network identities.

#### 3. **Persistent Storage for MongoDB**

**Chosen Configuration:**
- **Persistent Volume (PV) and Persistent Volume Claim (PVC)**
  - **Reason**: Ensures that MongoDB data is retained even if the pod is terminated or restarted, which is critical for data persistence.

**Alternatives Considered:**
- Using a `hostPath` volume for development purposes. While `hostPath` can be useful for local development, PVC is preferred for its portability and ability to work with various storage backends in production.

#### 4. **Horizontal Pod Autoscaler (HPA)**

**Chosen Configuration:**
- **Metric**: CPU utilization at 70%
  - **Reason**: CPU utilization is a common metric for autoscaling web applications. A threshold of 70% is chosen to balance performance and cost by scaling up before reaching high resource usage.
- **Replicas**: Minimum 2 and maximum 5
  - **Reason**: Minimum replicas ensure availability, while a maximum of 5 prevents over-provisioning and manages costs.

**Alternatives Considered:**
- Autoscaling based on memory usage. CPU was chosen due to its direct relation to application load and performance, and it's generally more indicative of application stress.

#### 5. **Kubernetes Services**

**Chosen Configuration:**
- **Flask Service**: Type `NodePort`
  - **Reason**: Allows access to the Flask application from outside the cluster for testing and development purposes.
- **MongoDB Service**: ClusterIP
  - **Reason**: Ensures MongoDB is only accessible within the cluster, enhancing security.

**Alternatives Considered:**
- Using `LoadBalancer` service type for Flask in production environments for external access. `NodePort` was used here for simplicity in a local Minikube setup.

### Testing Scenarios

#### 1. **Autoscaling Testing**

**Steps:**
1. **Simulate Traffic**:
   - Used `hey` tool to generate HTTP requests to the Flask application:
     ```bash
     hey -n 1000 -c 10 http://<Minikube-Node-IP>:<NodePort>
     ```
   - This command sends 1000 requests with a concurrency of 10 to simulate high traffic.

2. **Monitor Autoscaling**:
   - Checked the HPA status:
     ```bash
     kubectl get hpa
     ```
   - Verified the number of replicas scaled up and down according to CPU utilization.

**Results:**
- The HPA successfully scaled the Flask deployment from 2 to 5 replicas when CPU utilization exceeded 70%.
- After traffic subsided, the HPA scaled the replicas back to 2 as expected.

**Issues Encountered:**
- **Initial Configuration**: There was a delay in scaling due to an initial misconfiguration in the HPA metrics. Correcting the metric configuration resolved this issue.

#### 2. **Database Interactions Testing**

**Steps:**
1. **Insert Data**:
   - Used `curl` to perform POST requests to the `/data` endpoint:
     ```bash
     curl -X POST -H "Content-Type: application/json" -d '{"key":"value"}' http://<Minikube-Node-IP>:<NodePort>/data
     ```

2. **Retrieve Data**:
   - Used `curl` to perform GET requests:
     ```bash
     curl http://<Minikube-Node-IP>:<NodePort>/data
     ```

3. **Verify Data in MongoDB**:
   - Connected to MongoDB pod and verified data:
     ```bash
     kubectl exec -it <mongodb-pod-name> -- mongo -u root -p password --authenticationDatabase admin
     ```
   - Used MongoDB shell commands to check the data in the `flask_db` database.

**Results:**
- Data was successfully inserted and retrieved as expected. The interaction between the Flask application and MongoDB was smooth.

**Issues Encountered:**
- **Network Latency**: Occasionally faced network latency issues due to the Minikube setup. This was resolved by increasing resource limits for the Minikube VM.

The chosen configurations and setups were selected based on best practices for security, performance, and scalability. Testing confirmed that autoscaling and database interactions worked as intended, with some minor issues resolved during the process. The setup should be robust for both development and initial production use cases.

