# 🐳 Docker

## Table of Contents

- [What are Containers?](#what-are-containers)
- [Benefits of Using Containers](#benefits-of-using-containers)
- [Docker Engine](#docker-engine)
- [Docker Image vs Docker Container](#docker-image-vs-docker-container)
- [Common Docker Commands](#common-docker-commands)
- [Dockerfile](#dockerfile)
  - [Common Dockerfile Instructions](#common-dockerfile-instructions)
  - [Passing Environment Variables](#passing-environment-variables)
- [Layers & Caching](#layers--caching)
- [Volumes](#volumes)
- [Network](#network)
- [Docker Compose](#docker-compose)
- [Docker Push](#docker-push)

---

## What are Containers?

Containers allow you to package an application, along with all its dependencies and libraries, into a single unit that can be run on any machine with a container runtime, such as Docker. They run consistently across different environments.

![Container Diagram](./images/container.png)


You can talk to Docker using:
- **Terminal**
- **REST API**
- **Docker Desktop**

---

## Benefits of Using Containers

1. Let you describe your `configuration` in a single file
2. Can run in isolated environments
3. Makes local setup of OS projects a breeze
4. Makes installing auxiliary services/DBs easy

---

## Docker Engine

Docker Engine is an open-source `containerization` technology that allows developers to package applications into `containers`.

**Containers** are standardized executable components combining **application source code** with the **operating system (OS) libraries and dependencies** required to run that code in any environment.

---

## Docker Image vs Docker Container

### Docker Image

A Docker image is a lightweight, standalone, executable package that includes everything needed to run a piece of software — the code, a runtime, libraries, environment variables, and config files.

> 💡 A good mental model for an image is **your codebase on GitHub**.

### Docker Container

A container is a running instance of an image. It encapsulates the application or service and its dependencies, running in an isolated environment.

> 💡 A good mental model for a container is when you run `node index.js` on your machine from some source code you got from GitHub.

![Image vs Container Diagram](./images/ImgvsCont.png)
---

## Common Docker Commands

### 1. `docker images`
Shows all the images that you have on your machine.

### 2. `docker ps`
Shows all the containers you are running on your machine.

### 3. `docker run <image>`
Lets you start a container.
- `-p` → lets you create a port mapping
- `-d` → lets you run it in detached mode (in the background so you don't need a new terminal)

### 4. `docker build`
Lets you build an image.

### 5. `docker push`
Lets you push your image to a registry.

### 6. Extra Commands

| Command | Description |
|--------|-------------|
| `docker kill` | Stop a running container immediately |
| `docker exec` | Run a new command inside a running container / access the container's file system |
| `docker rmi <image_name>` | Remove the image from the machine |

**Examples using `docker exec`:**

```bash
# List contents of a container folder
docker exec <container_name_or_id> ls /path/to/directory

# Run an interactive shell inside a container
docker exec -it <container_name_or_id> /bin/bash
```

> **Note on `-d` flag:** Running with `-d` starts the container in the background so you can keep using the same terminal for other work.

---

## Dockerfile

If you want to create an image from your own code that you can push to DockerHub, you need to create a `Dockerfile` for your application.

A Dockerfile is a text document that contains all the commands a user could call on the command line to create an image.

### Common Dockerfile Instructions

| Instruction | Description |
|-------------|-------------|
| `WORKDIR` | Sets the working directory for `RUN`, `CMD`, `ENTRYPOINT`, `COPY` instructions that follow it |
| `RUN` | Executes commands in a new layer on top of the current image and commits the results (**build time**) |
| `CMD` | Provides defaults for executing a container — runs **when the container starts** (can be overridden) |
| `EXPOSE` | Informs Docker that the container listens on the specified network ports at runtime |
| `ENV` | Sets environment variables |
| `COPY` | Copies files from Docker host into the Docker image |

**`COPY . .`** means — copy everything from the current directory on your local machine → into the current working directory inside the Docker image (e.g. `/app`).

> **Key difference — `RUN` vs `CMD`:**  
> `RUN` executes **during image build** (e.g., installing packages, compiling code).  
> `CMD` executes **when a container starts** from the image; it does not create a new layer and can be overridden via `docker run`.

You can also create a `.dockerignore` file to exclude files/folders from being copied when running the `COPY` command.

**Build command:**

```bash
docker build -t <image_name> .
```

- `-t` → Tag (name) for the image
- `.` → Specifies the build context (current directory containing your `Dockerfile` and app files)

![dockerfile Diagram](./images/c1.png)
---

### Passing Environment Variables

```bash
docker run -p 3000:3000 -e DATABASE_URL="postgres://user:pass@host/db" image_name
```

> ⚠️ Avoid hardcoding env vars in the Dockerfile — inject them at runtime instead.

**Better approach — use an `.env` file:**

```env
DATABASE_URL=postgres://user:pass@host/db
API_KEY=xyz123
```

```bash
docker run --env-file .env image_name
```

This keeps environment variables **organized and secure**.

---

### Layers & Caching

Layers are a fundamental part of Docker image architecture. A Docker image is essentially built up from a series of layers, each representing a set of differences from the previous layer.

- If something changes at a particular step, **everything after that step will re-run**.
- Caching is computed based on **checksums of instructions, parent layers, and file contents** — layers whose hashes match a previous build are reused.
- Caching works **across images** — if two separate images share identical first few layers, those are cached and shared.

**Optimization tip:**

Instead of this (inefficient):
```dockerfile
COPY . .
RUN npm install
```

Do this (optimized):
```dockerfile
COPY package.json package-lock.json ./
RUN npm install
COPY . .
```

This way, `npm install` only re-runs when `package.json` changes — not every time your source code changes.

![dockerfile Diagram](./images/c2.png)

![dockerfile Diagram](./images/c3.png)
---

## Volumes

If you restart a `mongo` Docker container, you'll notice that your data goes away. This is because Docker containers are **transitory** — they don't retain data across restarts.

A **Docker volume** is a directory managed by Docker that exists outside the container's layered filesystem, so data persists even if the container is deleted.

**Using volumes:**

```bash
# 1. Create a volume
docker volume create volume_database

# 2. Mount it when running the container
docker run -v volume_database:/data/db -p 27017:27017 mongo
```

`volume_database` lives outside the container — even if the container is deleted, data stays.

![Volume Diagram](./images/volume.png)
---

## Network

- A Docker network is a powerful feature that allows containers to communicate with each other and with the outside world.
- **Docker containers can't talk to each other by default.**
- `localhost` on a Docker container means **its own network**, not the host machine's network.

### How to Make Containers Talk to Each Other

**Step 1 — Create a network:**

```bash
docker network create <network_name>
# e.g.
docker network create custom_network
```

**Step 2 — Start MongoDB on the same network:**

```bash
docker run -d \
  -v <volume_name>:<db_data_path_inside_container> \
  --name <db_container_name> \
  --network <network_name> \
  -e <env_variables_if_required> \
  -p <host_port>:<container_port> \
  <db_image>

# e.g.
docker run -d -v volume_database:/data/db --name mongo_network --network custom_network -p 27017:27017 mongo
```

**Step 3 — Build the app** (make sure to use correct DB URL):

```bash
docker build -t <backend_image_name> .
# e.g.
docker build -t mongoapp .
```

Inside backend code, DB connection string should use the **container name**:
```
mongodb://<db_container_name>:27017
# e.g.
mongodb://mongo_network:27017/myDatabase
```

Check logs:
```bash
docker logs <container_id>
```

**Step 4 — Run the backend container:**

```bash
docker run -d \
  -p <host_port>:<container_port> \
  --name <backend_container_name> \
  --network <network_name> \
  <backend_image_name>

# e.g.
docker run -d -p 3000:3000 --name backend_network --network custom_network mongoapp
```

![Network Diagram](./images/network.png)
---
### Port Mapping

```bash
docker run -d -p 27018:27017 mongo
#                 ↑       ↑
#            host port   container port
```

---

## Docker Compose

Docker Compose is a tool used to define and manage **multi-container applications**.

Instead of running multiple `docker run` commands manually, you define everything in a `docker-compose.yml` file. With a single command, you can create and start all the services from your configuration.

| Without Compose | With Compose |
|----------------|--------------|
| Manually create network | Automatic network |
| Manually create volumes | Volumes defined in one file |
| Manually start DB | One command: `docker-compose up` |
| Manually start backend | Cleaner and production-friendly |
| Maintain correct order | `depends_on` handles ordering |

![Docker-Compose Diagram](./images/docker-compose.png)
### Example `docker-compose.yml`

```yaml
services:
  mongodb_service:
    image: "mongo"
    container_name: "mongodb_container"
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db
  backend_service:
    build: .
    container_name: "backend_container"
    ports:
      - "5000:5000"
    depends_on:
      - mongodb_service
    environment:
      - MONGO_URI=mongodb://mongodb_service:27017/mydatabase
volumes:
  mongo_data:
```

**What the above Compose file does:**
- Creates a Docker network automatically so `backend_service` and `mongodb_service` can communicate
- Creates a Docker volume `mongo_data` to store MongoDB data persistently
- Pulls the MongoDB image (`mongo`) from Docker Hub
- Starts a MongoDB container named `mongodb_container`
- Mounts `mongo_data` to `/data/db` so data isn't lost when the container stops
- Maps MongoDB port `27017` → `Host:27017`
- Builds the backend image using the `Dockerfile` in the current directory
- Starts the backend container named `backend_container`
- Maps backend port `5000` → `Host:5000`
- Starts MongoDB before backend using `depends_on`
- Passes `MONGO_URI` via environment variable so backend connects to MongoDB using the service name

### Equivalent without Compose

```bash
docker network create custom_network

docker volume create mongo_data

docker run -d \
  --name mongodb_container \
  --network custom_network \
  -p 27017:27017 \
  -v mongo_data:/data/db \
  mongo

docker build -t backend_service_image .

docker run -d \
  --name backend_container \
  --network custom_network \
  -p 5000:5000 \
  -e MONGO_URI=mongodb://mongodb_container:27017/mydatabase \
  backend_service_image
```

### Compose Commands

```bash
# Start everything
docker-compose up

# Stop everything (including volumes)
docker-compose down --volumes
```

> **Note:** In the env variable inside Compose, use the **service name** (e.g., `mongodb_service`), not the `container_name`. Without Compose, use the `container_name`.
>
> If you already have a built image or an image on DockerHub, use `image: <image_name>`. If you haven't built it yet, use `build: .` — this creates the image with a default name.

---

## Docker Push

Once you've created your image, push it to DockerHub to share it.

```bash
docker push your_username/your_reponame:tagname
```

When you build an image like:

```bash
docker build -t myapp .
```

Docker automatically assigns:
- Repository → `myapp`
- Tag → `latest` (default)

So internally it becomes `myapp:latest`.

To push to DockerHub, **you must include your username** (so Docker knows which registry/namespace to push to):

```bash
docker push ayushman/myapp
```

You can also tag and push with a version:

```bash
docker build -t ayushman/myapp:v1 .
docker push ayushman/myapp:v1
```

Running `docker images` would then show:

```
REPOSITORY       TAG       IMAGE ID
myapp            latest    abc123
ayushman/myapp   v1        abc123
```

---
