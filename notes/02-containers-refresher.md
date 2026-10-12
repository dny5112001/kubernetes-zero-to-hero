# 02 · Containers in 10 minutes

Kubernetes runs **containers**. You don't need to be a Docker expert, but you
need these ideas.

## Image vs container

- An **image** is a **packed lunchbox**: your app plus everything it needs
  (runtime, libraries, files), frozen and read-only.
- A **container** is that lunchbox **opened and running**. You can run many
  containers from one image.

```
image  nginx:1.29-alpine   ──run──►  container #1
                           ──run──►  container #2
```

## Why containers?

"It works on my machine" goes away. The image contains everything, so it runs
the same on your laptop, on a test server, and in production. And containers
start in seconds and share the computer efficiently (lighter than virtual machines).

## Image names

```
docker.io/library/nginx:1.29-alpine
└───┬───┘ └──┬──┘ └─┬─┘ └────┬────┘
 registry   owner  name     tag (version)
```

- **Registry** = an online shelf of images (Docker Hub, GitHub Container Registry, AWS ECR).
- **Tag** = the version. **Avoid `latest`** in Kubernetes: you can't tell which
  version is running, and rollbacks get confusing. Use real versions like `1.29-alpine`.

## A Dockerfile = the recipe for an image

```dockerfile
FROM python:3.13-slim            # start from an existing image
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
USER 10001                       # don't run as root
CMD ["python", "app.py"]         # what runs when the container starts
```

```bash
docker build -t my-app:1.0 .     # recipe → image
docker run -p 8000:8000 my-app:1.0   # image → running container
docker push registry.example.com/me/my-app:1.0   # put it on a shelf
```

Kubernetes then **pulls** the image from the registry onto whichever computer
runs it.

> With kind (our local cluster) you can skip the registry:
> `kind load docker-image my-app:1.0 --name learn` copies the image straight in.

## Containers are temporary

A very important habit: **anything written inside a container is lost when it
stops.** Kubernetes restarts and moves containers all the time. So:
- Keep **data** in databases or volumes (chapter 10).
- Keep **settings** in ConfigMaps and Secrets (chapter 09), not baked into images.
- Write **logs** to the screen (stdout), not to files — Kubernetes collects them.

Apps built this way are called **stateless** — perfect for Kubernetes.

## Try it

```bash
docker run --rm -p 8095:80 nginx:1.29-alpine
# open http://localhost:8095, then press Ctrl+C
```

## Remember

- Image = packed lunchbox (frozen). Container = opened and running.
- Use real version tags, not `latest`.
- Containers are temporary — keep data and settings outside them.
