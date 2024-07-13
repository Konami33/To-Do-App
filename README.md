# To-Do App

![alt text](image.png)

## Project Structure

```
todo-app/
│
├── api-gateway/
│   ├── node_modules/
│   ├── .env
│   ├── package.json
│   ├── index.js
│   └── routes.js
│
├── todo-service/
│   ├── node_modules/
│   ├── .env
│   ├── package.json
│   ├── index.js
│   └── todoController.js
│
└── docker-compose.yml
```

## API Gateway

### `api-gateway/.env`

```plaintext
PORT=5000
TODO_SERVICE_URL=http://localhost:8000
```

### `api-gateway/package.json`

```json
{
  "name": "api-gateway",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "axios": "^0.27.2",
    "dotenv": "^16.0.1",
    "express": "^4.18.1"
  }
}
```

### `api-gateway/index.js`

```javascript
const express = require('express');
const dotenv = require('dotenv');
const routes = require('./routes');

dotenv.config();

const app = express();
const port = process.env.PORT || 5000;

app.use(express.json());
app.use('/', routes);

app.listen(port, () => {
  console.log(`API Gateway listening at http://localhost:${port}`);
});
```

### `api-gateway/routes.js`

```javascript
const express = require('express');
const axios = require('axios');
const router = express.Router();
const TODO_SERVICE_URL = process.env.TODO_SERVICE_URL;

// Get all tasks
router.get('/', async (req, res) => {
  try {
    const response = await axios.get(`${TODO_SERVICE_URL}/tasks`);
    res.json(response.data);
  } catch (error) {
    res.status(500).send(error.message);
  }
});

// Get a specific task by ID
router.get('/:id', async (req, res) => {
  try {
    const response = await axios.get(`${TODO_SERVICE_URL}/tasks/${req.params.id}`);
    res.json(response.data);
  } catch (error) {
    res.status(500).send(error.message);
  }
});

// Add a new task
router.post('/', async (req, res) => {
  try {
    const response = await axios.post(`${TODO_SERVICE_URL}/tasks`, req.body);
    res.json(response.data);
  } catch (error) {
    res.status(500).send(error.message);
  }
});

// Delete a task by ID
router.delete('/:id', async (req, res) => {
  try {
    const response = await axios.delete(`${TODO_SERVICE_URL}/tasks/${req.params.id}`);
    res.json(response.data);
  } catch (error) {
    res.status(500).send(error.message);
  }
});

module.exports = router;
```

## ToDo Service

### `todo-service/.env`

```plaintext
PORT=8000
REDIS_URL=redis://localhost:6379
```

### `todo-service/package.json`

```json
{
  "name": "todo-service",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "dotenv": "^16.4.5",
    "express": "^4.19.2",
    "redis": "^4.6.15",
    "uuid": "^10.0.0"
  }
}

```

### `todo-service/index.js`

```javascript
const express = require('express');
const dotenv = require('dotenv');
const todoController = require('./todoController');

dotenv.config();

const app = express();
const port = process.env.PORT || 8000;

app.use(express.json());

app.get('/tasks', todoController.getTasks);
app.get('/tasks/:id', todoController.getTaskById);
app.post('/tasks', todoController.addTask);
app.delete('/tasks/:id', todoController.deleteTask);

app.listen(port, () => {
  console.log(`ToDo service listening at http://localhost:${port}`);
});
```

### `todo-service/todoController.js`

```javascript
const redis = require('redis');
const { v4: uuidv4 } = require('uuid'); // Import UUID library
const client = redis.createClient({
  url: process.env.REDIS_URL
});

client.connect();

exports.getTasks = async (req, res) => {
  try {
    const tasks = await client.hGetAll('tasks'); // Use hGetAll to get all tasks with their IDs
    const tasksArray = Object.keys(tasks).map(id => ({
      id,
      task: tasks[id]
    }));
    res.json(tasksArray);
  } catch (error) {
    res.status(500).send(error.message);
  }
};

exports.getTaskById = async (req, res) => {
  try {
    const taskId = req.params.id;
    const task = await client.hGet('tasks', taskId); // Get task by ID
    if (task) {
      res.json({ id: taskId, task });
    } else {
      res.status(404).send('Task not found');
    }
  } catch (error) {
    res.status(500).send(error.message);
  }
};

exports.addTask = async (req, res) => {
  try {
    const task = req.body.task;
    const taskId = uuidv4(); // Generate a unique ID for the task
    await client.hSet('tasks', taskId, task); // Store task in a hash with the task ID
    res.send({ id: taskId, task });
  } catch (error) {
    res.status(500).send(error.message);
  }
};

exports.deleteTask = async (req, res) => {
  try {
    const taskId = req.params.id;
    await client.hDel('tasks', taskId); // Delete the task by ID
    res.send('Task deleted');
  } catch (error) {
    res.status(500).send(error.message);
  }
};
```

### Docker Compose

### `docker-compose.yml`

```yaml
version: '3.8'

services:
  redis:
    image: 'redis/redis-stack:latest'
    ports:
      - '6379:6379'
      - '8001:8001'
  
  api-gateway:
    build: ./api-gateway
    ports:
      - '5000:5000'
    environment:
      - TODO_SERVICE_URL=http://todo-service:8000
    depends_on:
      - todo-service

  todo-service:
    build: ./todo-service
    ports:
      - '8000:8000'
    environment:
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis

```

## Running the Application

1. Navigate to the root directory of your project.
2. Run `docker-compose up --build`.

This setup will create the API gateway and ToDo services, and use Redis as the database. The client can make requests to the API gateway, which will forward them to the ToDo service, and the tasks will be stored in Redis.




















root@8df25aaaa58a90a3:~/code/To-Do-App/Deployment_files# kubectl describe pods todo-deployment-6b686597b9-njbnq
Name:             todo-deployment-6b686597b9-njbnq
Namespace:        default
Priority:         0
Service Account:  default
Node:             cluster-uie4dq-worker-1/10.62.12.241
Start Time:       Sat, 13 Jul 2024 13:56:25 +0000
Labels:           app=todo-app
                  pod-template-hash=6b686597b9
Annotations:      <none>
Status:           Running
IP:               10.42.2.5
IPs:
  IP:           10.42.2.5
Controlled By:  ReplicaSet/todo-deployment-6b686597b9
Containers:
  todo-container:
    Container ID:   containerd://698bab98ad4b2dff1f8b974b0969ec9840c0b12caf58b69a9acbd5a03bce35d7
    Image:          konami98/todo-service:latest
    Image ID:       docker.io/konami98/todo-service@sha256:113cbc9c8b24810e02fbf79b5433c58d9d6783ff2a1a4462c9249b12a4dfe3cf
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
      Started:      Sat, 13 Jul 2024 13:59:48 +0000
      Finished:     Sat, 13 Jul 2024 13:59:49 +0000
    Ready:          False
    Restart Count:  5
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-d7ttc (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-d7ttc:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                    From               Message
  ----     ------     ----                   ----               -------
  Normal   Scheduled  4m29s                  default-scheduler  Successfully assigned default/todo-deployment-6b686597b9-njbnq to cluster-uie4dq-worker-1
  Normal   Pulled     4m21s                  kubelet            Successfully pulled image "konami98/todo-service:latest" in 8.255s (8.255s including waiting). Image size: 45465097 bytes.
  Normal   Pulled     4m17s                  kubelet            Successfully pulled image "konami98/todo-service:latest" in 1.825s (1.825s including waiting). Image size: 45465097 bytes.
  Normal   Pulled     4m2s                   kubelet            Successfully pulled image "konami98/todo-service:latest" in 1.903s (1.903s including waiting). Image size: 45465097 bytes.
  Normal   Created    3m31s (x4 over 4m21s)  kubelet            Created container todo-container
  Normal   Started    3m31s (x4 over 4m21s)  kubelet            Started container todo-container
  Normal   Pulled     3m31s                  kubelet            Successfully pulled image "konami98/todo-service:latest" in 1.781s (1.781s including waiting). Image size: 45465097 bytes.
  Warning  BackOff    2m51s (x8 over 4m16s)  kubelet            Back-off restarting failed container todo-container in pod todo-deployment-6b686597b9-njbnq_default(e6ee2c93-a512-4590-bdc1-51a97791fb0e)
  Normal   Pulling    2m36s (x5 over 4m29s)  kubelet            Pulling image "konami98/todo-service:latest"