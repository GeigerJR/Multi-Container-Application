
---

Multi-Container Node.js + MongoDB App with Docker Compose, Terraform, Ansible, and GitHub Actions

---

### A. Build Node.js Todo API Locally

1. Create project folder and enter it:

   ```bash
   mkdir todo-api
   cd todo-api
   ```

2. Initialize npm and install dependencies:

   ```bash
   npm init -y
   npm install express mongoose nodemon
   ```

3. Create `index.js` (use your editor) with this minimal code (copy-paste):

```js
// index.js
const express = require('express');
const mongoose = require('mongoose');

const app = express();
app.use(express.json());

mongoose.connect('mongodb://mongo:27017/todos', { useNewUrlParser: true, useUnifiedTopology: true });

const todoSchema = new mongoose.Schema({ title: String, completed: Boolean });
const Todo = mongoose.model('Todo', todoSchema);

app.get('/todos', async (req, res) => res.json(await Todo.find()));
app.post('/todos', async (req, res) => res.status(201).json(await new Todo(req.body).save()));
app.get('/todos/:id', async (req, res) => {
  const todo = await Todo.findById(req.params.id);
  if (!todo) return res.status(404).send('Not Found');
  res.json(todo);
});
app.put('/todos/:id', async (req, res) => {
  const todo = await Todo.findByIdAndUpdate(req.params.id, req.body, { new: true });
  if (!todo) return res.status(404).send('Not Found');
  res.json(todo);
});
app.delete('/todos/:id', async (req, res) => {
  const todo = await Todo.findByIdAndDelete(req.params.id);
  if (!todo) return res.status(404).send('Not Found');
  res.status(204).send();
});

app.listen(3000, () => console.log('Server running on http://localhost:3000'));
```

4. Add `"start": "nodemon index.js"` script to your `package.json` under `scripts`.

5. Run the API locally:

   ```bash
   npx nodemon index.js
   ```

6. Test your API with curl or Postman at `http://localhost:3000/todos`.

---

### B. Dockerize the API and MongoDB

1. Create a `Dockerfile` with this content:

```Dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["nodemon", "index.js"]
```

2. Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      - mongo
    environment:
      - MONGO_URI=mongodb://mongo:27017/todos
    volumes:
      - .:/app
    command: nodemon index.js

  mongo:
    image: mongo:6.0
    restart: always
    volumes:
      - mongo-data:/data/db

volumes:
  mongo-data:
```

3. Build and start containers:

   ```bash
   docker-compose up --build
   ```

4. Access API at `http://localhost:3000`

---

### C. Provision Remote Server with Terraform

1. Create `main.tf` with something like:

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "todo_server" {
  ami           = "ami-0c55b159cbfafe1f0"  # Ubuntu 22.04 in us-east-1 (update if needed)
  instance_type = "t2.micro"
  key_name      = "your-ssh-key-name"       # Replace with your key pair name

  tags = {
    Name = "todo-server"
  }
}
```

2. Initialize and apply Terraform:

```bash
terraform init
terraform apply
```

3. Note the public IP address from Terraform output.

---

### D. Configure Server with Ansible

1. Create `inventory` file with your server IP:

```
[todo]
your_server_ip ansible_user=ubuntu
```

2. Create `playbook.yml`:

```yaml
- hosts: todo
  become: yes
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install Docker and dependencies
      apt:
        name:
          - docker.io
          - docker-compose
        state: present

    - name: Add user ubuntu to docker group
      user:
        name: ubuntu
        groups: docker
        append: yes

    - name: Pull your Docker images or clone repo
      git:
        repo: 'https://github.com/yourusername/todo-api.git'
        dest: /home/ubuntu/todo-api
        version: main

    - name: Run docker-compose up
      command: docker-compose up -d
      args:
        chdir: /home/ubuntu/todo-api
```

3. Run the playbook:

```bash
ansible-playbook -i inventory playbook.yml
```

---

### E. Push Code to GitHub

1. Initialize git (if not done):

```bash
git init
git add .
git commit -m "Initial commit"
```

2. Create GitHub repo and push:

```bash
git remote add origin https://github.com/yourusername/todo-api.git
git branch -M main
git push -u origin main
```

---

### F. Setup GitHub Actions for CI/CD

1. Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy Todo API

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repo
        uses: actions/checkout@v3

      - name: Set up SSH
        uses: webfactory/ssh-agent@v0.5.4
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Sync files to server
        run: rsync -avz --delete ./ ubuntu@your_server_ip:/home/ubuntu/todo-api

      - name: Deploy app
        run: ssh ubuntu@your_server_ip "cd /home/ubuntu/todo-api && docker-compose pull && docker-compose up -d"
```

2. Add your private SSH key in GitHub repo secrets as `SSH_PRIVATE_KEY`.

---

### G. Bonus: Setup Nginx Reverse Proxy in Docker Compose

1. Update your `docker-compose.yml`:

```yaml
services:
  api:
    build: .
    expose:
      - "3000"
    depends_on:
      - mongo
    environment:
      - MONGO_URI=mongodb://mongo:27017/todos
    volumes:
      - .:/app
    command: nodemon index.js

  mongo:
    image: mongo:6.0
    restart: always
    volumes:
      - mongo-data:/data/db

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - api
```

2. Create `nginx.conf`:

```nginx
events {}

http {
  server {
    listen 80;

    location / {
      proxy_pass http://api:3000;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
    }
  }
}
```

3. Restart docker-compose:

```bash
docker-compose down
docker-compose up -d --build
```

---

### H. Test & Verify

* Locally: `curl http://localhost:3000/todos`
* Remotely: `curl http://your_server_ip/todos` or visit `http://your_domain.com` (if using Nginx + domain)

---
