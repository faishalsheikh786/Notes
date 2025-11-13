# 🐳 Docker Complete Notes

---

## 🐳 What is Docker?
Docker is a platform that lets you **build, ship, and run applications** in isolated environments called **containers**.

### 💡 Think of it like this:
- **Without Docker** → your app runs directly on your computer, and you have to install dependencies (Python, Node.js, libraries, etc.) manually.
- **With Docker** → your app and all its dependencies are packaged together so it runs the same way everywhere — on your laptop, a server, or the cloud.

---

## ⚙️ What is Docker and Daemon?
The **Docker Daemon** (`dockerd`) is the background service that runs on your system and manages Docker:
- Builds images
- Runs containers
- Handles networking
- Manages volumes

You don’t interact directly with the daemon — instead, you use the **Docker CLI (Command Line Interface)** (`docker` command), which talks to the daemon.

### 📊 Flow:
```
You → Docker CLI → Docker Daemon → Containers / Images
```

---

## 📦 What is a Docker Image?
A **Docker image** is like a **blueprint or template** that contains everything your app needs to run:

- Operating System (like Ubuntu or Alpine)
- App Code
- Libraries & Dependencies
- Configuration Instructions

You can think of it as a **frozen snapshot** of an environment.

### 📘 Example:
```bash
docker pull ubuntu
```
This downloads an image of Ubuntu Linux.

---

## 🚀 What is a Docker Container?
A **container** is a **running instance** of an image.

- When you run an image, Docker creates a container from it.
- It’s isolated from your system but can communicate through defined channels.
- You can start, stop, restart, or delete containers freely.

### 🧠 Analogy:
```
Image = Class or Blueprint
Container = Object (instance of that blueprint)
```

### Example:
```bash
docker run -it ubuntu bash
```
→ Creates and starts a container based on the Ubuntu image and gives you a bash shell inside it.

---

## 🌐 What is Port Mapping?
Each container has its own isolated network.  
If your container runs a web server (say on port 80 inside the container), you need a way to access it from your host machine.

### Example:
```bash
docker run -p 8080:80 nginx
```
- **8080** → your machine’s port
- **80** → container’s internal port

So, opening [http://localhost:8080](http://localhost:8080) accesses the container’s web server.

---

## 🐳 What Does “Dockerize” Mean?
To **Dockerize** means to package your application (and everything it needs) into a Docker image/container.  
You’re making your app portable so it runs anywhere.

---

## ⚡ Caching in Docker
When Docker builds an image from a `Dockerfile`, it executes instructions **step by step** and **caches each layer** to avoid repeating work unnecessarily.

### Example Dockerfile:
```dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
ENTRYPOINT ["node", "main.js"]
```

### 🔍 Build Process (Layers):
1. `FROM node:18` → base image (cached)
2. `WORKDIR /app` → sets directory (cached)
3. `COPY package*.json ./` → copies dependency files (cached)
4. `RUN npm install` → installs dependencies (cached)
5. `COPY . .` → copies source code

If `package.json` doesn’t change, Docker reuses cached layers → faster rebuilds. ⚡

---

## 🐳 What is Docker Compose?
**Docker Compose** helps you run **multi-container applications** with a single command using a YAML file (`docker-compose.yml`).

Instead of running multiple `docker run` commands manually, define everything once and run:
```bash
docker compose up
```

### Docker Compose handles:
- Creating containers  
- Networking between them  
- Managing volumes  
- Defining environment variables  
- Controlling startup order  

### Example:
You have:
- A **Node.js** backend  
- A **MongoDB** database  

Both can be managed together with Docker Compose.

---

## 🧠 Common Docker Commands

### ✅ Check Docker Installation
```bash
docker
docker -v
```

---

### 🏗️ Build Image
```bash
docker build -t <image-name> .
```
Example:
```bash
docker build -t my-app .
```

---

### 🚀 Run Container Using Existing Image
```bash
docker run <image-name>
```
Example:
```bash
docker run ubuntu
```

Run interactively:
```bash
docker run -it ubuntu
```

Run with port mapping:
```bash
docker run -p <host-port>:<container-port> <image-name>
```
Example:
```bash
docker run -p 8080:3000 my-app
```

Run in detached mode:
```bash
docker run -d -p 8080:3000 my-app
```

Run with environment variables:
```bash
docker run -it -e key=value -e key2=value2 <image-name>
```

Run with a name:
```bash
docker run -d --name my-container -p 8080:3000 my-app
```

---

### 📋 List Containers & Images
```bash
docker container ls         # running containers
docker container ls -a      # all containers
docker image ls             # list images
docker images               # same as above
```

---

### ▶️ Start/Stop Container
```bash
docker start <container-name>
docker stop <container-name>
```

---

### 💻 Access Running Container
```bash
docker exec -it <container-name> bash
```
Starts an interactive shell inside the running container.

---

### 🧹 Delete Containers
```bash
docker rm <container_id_or_name>             # Remove stopped container
docker rm -f <container_id_or_name>          # Force remove running container
docker container prune                       # Remove all stopped containers
docker rm -f $(docker ps -aq)                # Remove ALL containers (running + stopped)
```

---

### 🧽 Delete Images
```bash
docker rmi <image_id_or_name>                # Delete a single image
docker rmi -f <image_id_or_name>             # Force remove image
docker image prune                           # Delete unused images
docker image prune -a                        # Delete all unused images
```

---

### 🧹 Clean Everything
```bash
docker system prune -a
```
Removes:
- Stopped containers  
- Unused images  
- Unused networks  
- Build cache  

⚠️ **Use with caution**.

---

### 🐙 Docker Compose Commands
```bash
docker compose up             # start services
docker compose up -d          # start in detached mode
docker compose down           # stop and remove containers
```

---

### 🧱 Example docker-compose.yml
```yaml
version: '3.8'

services:
  postgres:
    image: postgres
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_DB: review
      POSTGRES_PASSWORD: password

  redis:
    image: redis
    ports:
      - "6379:6379"
```

---

## 🧠 Summary Table

| Command | Description |
|----------|--------------|
| `docker` | Check if Docker is installed |
| `docker -v` | Check Docker version |
| `docker image ls` | List all downloaded images |
| `docker build -t <image-name> <path>` | Build an image from a Dockerfile |
| `docker run -it <image>` | Run an image interactively (downloads if not present) |
| `docker run -p <host-port>:<container-port> <image>` | Map container port to host |
| `docker run -e key=value` | Pass environment variables to a container |
| `docker container ls` | List only running containers |
| `docker container ls -a` | List all containers (including stopped) |
| `docker start <container>` | Start a stopped container |
| `docker stop <container>` | Stop a running container |
| `docker exec -it <container> bash` | Open an interactive shell in a running container |
| `docker compose up` | Start all services defined in `docker-compose.yml` |
| `docker compose up -d` | Start services in detached mode |
| `docker compose down` | Stop and remove all services |
| `docker rm <container-id>` | Remove a stopped container |
| `docker rm -f <container-id>` | Force remove a running container |
| `docker container prune` | Remove all stopped containers |
| `docker rmi <image-id>` | Remove a specific image |
| `docker rmi -f <image-id>` | Force remove an image even if used by containers |
| `docker image prune` | Remove unused (dangling) images |

---

### 💡 Tips
- Always use **official or verified images** from [Docker Hub](https://hub.docker.com).
- Containers are **isolated** — data isn’t shared unless you use **volumes**.
- To save time, understand **layer caching** when building Dockerfiles.

---

🧾 **Author:** Docker Study Notes  
📅 **Updated:** 2025  
✅ Suitable for Beginners & Intermediate Developers
