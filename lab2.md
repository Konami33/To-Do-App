# To-Do App Microservices Deployment Documentation

#### Overview

This GitHub Actions workflow automates the process of building and pushing Docker images for api-gateway and todo-service components of your application whenever changes are pushed to the main branch. It ensures that Docker images are only built and pushed if there are changes in the respective directories.

![alt text](./images/image-5.png)

#### Workflow Configuration

The workflow file is named Build and Push Docker Images.yml and resides in your repository's .github/workflows/ directory. It is triggered on a push event to the main branch, specifically monitoring changes in the api-gateway and todo-service directories.

```yaml
name: Build and Push Docker Images

on:
  push:
    branches:
      - main
    paths:
      - api-gateway/**
      - todo-service/**

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [api-gateway, todo-service]

    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Log in to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Check for changes in directory
        id: check_changes
        run: |
          if git diff --name-only HEAD~1 HEAD | grep -q "^${{ matrix.service }}/"; then
            echo "changes=true" >> $GITHUB_ENV
          else
            echo "changes=false" >> $GITHUB_ENV
          fi

      - name: Build and push image
        if: env.changes == 'true'
        uses: docker/build-push-action@v4
        with:
          context: ./${{ matrix.service }}
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/${{ matrix.service }}:latest
```
#### Workflow Explanation

1. *Triggering the Workflow*:  
   The workflow triggers on any push to the main branch, but it will only execute if changes are detected in the api-gateway or todo-service directories.

2. *Job Configuration*:  
   - The build-and-push job runs on ubuntu-latest.
   - It uses a matrix strategy to iterate over the services api-gateway and todo-service.

3. *Steps*:
   - *Checkout repository*: Checks out the repository's code base.
   - *Log in to Docker Hub*: Uses Docker login action to authenticate with Docker Hub using secrets.
   - *Set up Docker Buildx*: Prepares Docker Buildx, enabling multi-platform builds.
   - *Check for changes in directory*: Uses a script to determine if there are changes in the specific service directory (api-gateway or todo-service).
   - *Build and push image*: If changes are detected (env.changes == 'true'), it builds the Docker image from the service directory and pushes it to Docker Hub.

#### Security Considerations

- *Secrets Management*: Ensure that DOCKER_USERNAME and DOCKER_PASSWORD are securely stored in GitHub Secrets and not exposed in your workflow file.
- *Build Triggers*: Limiting builds to specific directories (api-gateway and todo-service) reduces unnecessary builds, improving security by minimizing exposure of build processes.

## Task 2: Applying Kubernetes Deployment Files

Once the Docker images are successfully pushed to `Docker Hub`, the next step is to deploy these images to Kubernetes using deployment files.

### Step-by-Step Guide

1. **Ensure Kubernetes Configuration**

   - Make sure you have `kubectl` configured to connect to your Kubernetes cluster. You can test this by running `kubectl get nodes`.

2. **Create Deployment Files**

   Create Kubernetes YAML files for each component of your application (Redis, API Gateway, and Todo Service).

   - **Redis Deployment:(redis.yml)**
    ```yml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: redis
    spec:
    replicas: 1
    selector:
        matchLabels:
        app: redis
    template:
        metadata:
        labels:
            app: redis
        spec:
        containers:
        - name: redis
            image: redis/redis-stack:latest
            ports:
            - containerPort: 6379
            - containerPort: 8001
    ---
    apiVersion: v1
    kind: Service
    metadata:
    name: redis
    spec:
    selector:
        app: redis
    ports:
    - protocol: TCP
        port: 6379
        targetPort: 6379
        name: redis
    - protocol: TCP
        port: 8001
        targetPort: 8001
        name: redis-dashboard
    type: ClusterIP
    ```
    - **API Gateway Deployment:(api-gateway.yml)**
    ```yml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: api-gateway
    spec:
    replicas: 2
    selector:
        matchLabels:
        app: api-gateway
    template:
        metadata:
        labels:
            app: api-gateway
        spec:
        containers:
        - name: api-gateway
            image: konami98/api-gateway:latest
            ports:
            - containerPort: 5000
            env:
            - name: TODO_SERVICE_URL
            value: "http://todo-service:8000"
    ---
    apiVersion: v1
    kind: Service
    metadata:
    name: api-gateway
    spec:
    selector:
        app: api-gateway
    ports:
    - protocol: TCP
        port: 5000
        targetPort: 5000
    type: NodePort
    ```
    - **API Gateway Service Deployment:(td.yml)**
    ```yml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: todo-service
    spec:
    replicas: 2
    selector:
        matchLabels:
        app: todo-service
    template:
        metadata:
        labels:
            app: todo-service
        spec:
        containers:
        - name: todo-service
            image: konami98/todo-service:latest
            ports:
            - containerPort: 8000
            env:
            - name: REDIS_URL
            value: "redis://redis:6379"
    ---
    apiVersion: v1
    kind: Service
    metadata:
    name: todo-service
    spec:
    selector:
        app: todo-service
    ports:
    - protocol: TCP
        port: 8000
        targetPort: 8000
    type: ClusterIP
    ```

3. **Apply Deployment Files**

   Use `kubectl apply` command to deploy the application into Kubernetes:

   ```bash
   kubectl apply -f redis.yml
   kubectl apply -f api-gateway.yml
   kubectl apply -f td.yml
   ```

   Replace `redis.yml`, `api-gateway.yml`, and `td.yml` with your actual file names.

4. **Verify Deployment**

   - Check the deployment status using `kubectl get pods`, `kubectl get services`, and `kubectl get deployments`.
   - Verify that all pods are running without errors.

   ![alt text](./images/image-1.png)

5. **Access Your Application**

   - To access the application, we need to find the `nodeport` where the api-gateway service is exposed. Run:
    ```sh
    kubectl get svc
    ```

    ![alt text](./images/image-2.png)

    - Get the internal-ip:
    ```sh
    kubectl get nodes -o wide
    ```

    ![alt text](./images/image-4.png)

6. **Check the api end-points**

    - To get all the TODO tasks:

    ```sh
    curl http://internal-ip:<node-port>

    ```sh
    curl http://10.62.12.246:30944
    ```
    - To get a specific TODO task:

    ```sh
    curl http://internal-ip:<node-port>/<task_id>
    ```

    - To create a todo task:

    ```sh
    curl -X POST http://internal-ip:<node-port> -H "Content-Type: application/json" -d '{"task": "Sample Task 1"}'
    ```
    - To delete a task:

    ```sh
    curl -X DELETE http://internal-ip:<node-port>/<task_id>
    ```

    ![alt text](./images/image-3.png)


### Conclusion

By following these steps, you can automate the build and deployment of your Dockerized application to Kubernetes using GitHub Actions. This approach enhances your development workflow by automating repetitive tasks and ensuring consistency in deployments across environments.