# CST8915 Lab 4: Introduction to Docker

**Student Name:** Georges Ndongmo Ngepi

**Student ID:** 041131489

**Course:** CST8915 Full-stack Cloud-native Development

**Semester:** Fall 2026

---

## Demo Video
**Video** https://youtu.be/t2Fh0wkMKMo 


---

## Environment

- **Docker host:** Azure VM `docker-vm`, Ubuntu Server 24.04 LTS, region Belgium Central
- **VM size:** Standard_B2als_v2 (2 vCPU, 4 GiB). Standard_B2s was not available for my subscription, so I chose a size with the same CPU and memory.
- **Docker:** Docker Engine 29.9.0, CLI, containerd, Buildx and the Compose plugin (v5.6.0), installed from Docker's official APT repository
- **Connection:** SSH with a key pair, using a `docker-vm` entry in my `~/.ssh/config`
- **Network security:** only SSH (port 22) is open, and only from my IP. All tests were done with `curl` on the VM itself, so no web port was opened to the internet.

---

## Reflection Questions

### 1. What are the main differences between a Docker image and a Docker container?

An image is a read-only template. It is a stack of layers that contains the code, the runtime, the libraries and the command to start the application. A container is a running instance of an image: Docker adds a thin writable layer on top of the image and starts a process with its own file system, network and processes. One image can create many containers. In Part 5, I started `app1`, `app2` and `app3` from the same image `my-python-app:v3`. A simple comparison: the image is like a class, and the container is like an object created from it. Deleting a container does not change the image.

### 2. Explain how Docker's layered architecture improves efficiency.

Each Dockerfile instruction that changes files (`FROM`, `COPY`, `RUN`, `WORKDIR`) creates a read-only layer, and Docker caches these layers. My first build (`v1`) took 12.5 seconds. When I built `v2` with the same instructions, every step was `CACHED` and the build took 0.9 seconds. When I changed only `app.py` and built `v3`, Docker reused the base image, `WORKDIR`, `COPY requirements.txt` and the `RUN pip install` layer (18.3 MB), and rebuilt only `COPY app.py`. This is why the Dockerfile copies `requirements.txt` before the code. Layers are also shared: the nginx image and the Python image used the same Debian base layer, and it was stored only once.

### 3. Why does each container get its own writable layer?

The image layers are read-only, so many containers can share them safely. But each container still needs a place to write its own changes (new files, logs, temporary data) without changing the image or the other containers. In Part 5, `data1.txt` existed only in `app1`. With copy-on-write, when I modified `app.py` in `app1`, Docker first copied the file into `app1`'s writable layer; `app2` still showed the original file. `docker ps -s` showed about 209 kB for each writable layer, while the 160 MB of the image was shared. This layer is deleted with the container, so important data must be stored in a volume, like `redis-data` in Part 6.

### 4. What are the benefits of using Docker Compose over running containers individually?

With Compose, one YAML file describes the whole application: the services, ports, environment variables, networks and volumes. One command, `docker compose up -d`, built the web image, pulled Redis, created the network and the volume, and started the containers in the right order (`depends_on`). The services find each other by name on the private network: the Flask app connects to the host `redis`, not to an IP address, and Redis is not exposed outside the network. It is also easier to manage everything together: `logs`, `stop`, `start`, `down`, and scaling with `--scale web=3`, where three web containers shared one counter in Redis. The volume kept the data: after `docker compose down` and `up`, the counter continued at 5; only `down -v` reset it.

---

## Challenges and Lessons Learned

- **Running commands in the wrong terminal:** a few times I pasted Linux commands into a new Windows PowerShell window instead of the SSH session. Now I always check that the prompt shows `azureuser@docker-vm` before pasting.
- **Reused public IP:** Azure gave my new VM the same public IP as an old Lab 2 VM, so I removed the old entry from my SSH config and from `known_hosts` (`ssh-keygen -R`).
- **VS Code Remote-SSH:** I first chose the wrong platform (Windows) for the Linux VM, and the connection failed. I fixed it in `remote.SSH.remotePlatform` in the VS Code settings.
- **Newer Docker version:** Docker 29 removed the old `.NetworkSettings.IPAddress` field, so I used `{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}` to get the container IP.
- **Port 80 already in use:** the `python-app` container from Part 3 was still using port 80, so I stopped it before starting Part 6.
- **Removing a running container fails:** `docker rm` needs the container to be stopped first.
- **`version` warning:** Docker Compose shows that the `version: '3.8'` attribute is obsolete. It is only a warning; the file still works.
- **Slow stop of the web service:** `docker compose stop` took 10 seconds for web but 0.2 seconds for Redis. Docker sends SIGTERM, and the Flask development server does not handle it, so Docker waits 10 seconds and then sends SIGKILL.
