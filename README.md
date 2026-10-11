# Day 9 — Dockerfile Deep Dive

## 1. Project Overview

This project demonstrates how to build a reusable Docker image for a Node.js HTTP API, configure it through environment variables, optimize Docker layer caching, and understand CMD versus ENTRYPOINT.

## 2. Project Structure

```text
day9-dockerfile/
├── Dockerfile
├── Dockerfile.greeter
├── .dockerignore
├── package.json
├── server.js
├── README.md
└── evidence/
```

## 3. Technologies Used
- Docker
- Node.js 20 Alpine
- Alpine Linux 3.20
- Docker BuildKit
## 4. Dockerfile Instructions
- FROM: Selects the base image.
- WORKDIR: Sets /app as the working directory.
- ARG: Defines a build-time variable named APP_VERSION.
- ENV: Sets runtime environment variables such as PORT and APP_ENV.
- COPY: Copies application files into the image.
- RUN: Installs production dependencies using npm install --omit=dev.
- EXPOSE: Documents that the application listens on port 3000.
- USER: Runs the application as the non-root node user.
- CMD: Specifies the default command to start the Node.js server.
## 5. Build the API Image
```docker build --progress=plain -t demo-api:v1 \
  --build-arg APP_VERSION=1.0 .
```

## 6. Run and Test the API
```docker run -d --name api -p 3000:3000 demo-api:v1
curl http://localhost:3000/
curl http://localhost:3000/health
```

Expected health endpoint response:
``` text
ok
```


## 7. Runtime Environment Variables
The API was tested with custom runtime environment variables:
docker rm -f api

docker run -d --name api -p 3000:3000 \
  -e GREETING="Hi Winter Arc" \
  -e APP_ENV=prod demo-api:v1


Test the response:
curl http://localhost:3000/


Expected response:
Hi Winter Arc | env=prod


This demonstrates that runtime environment variables can override the application's default values.
## 8. Docker Layer Caching Experiment
Experiment A: Change application source code
The default greeting in server.js was changed to Hello from Docker 9.
docker build --progress=plain -t demo-api:v2 .


Observation: Docker reused the cached COPY package*.json ./ and RUN npm install --omit=dev layers. The application source-copy layer was rebuilt.
Experiment B: Change package metadata
The package.json version was changed to 1.0.1.
docker build --progress=plain -t demo-api:v3 .
docker history demo-api:v3


Observation: The package manifest copy and dependency-install steps were executed again. Docker could not reuse the dependency-install layer because its preceding package-manifest input had changed.
Why this matters
Copying package manifests before the rest of the source code allows Docker to reuse the dependency-install layer when only application source files change.
## 9. CMD vs ENTRYPOINT
The Dockerfile.greeter file contains:
FROM alpine:3.20
ENTRYPOINT ["echo", "Hello"]
CMD ["World"]


Build the image:
docker build -f Dockerfile.greeter -t greeter:local .


Run with the default argument:
docker run --rm greeter:local


Output:
Hello World


Override the default argument:
docker run --rm greeter:local DevOps


Output:
Hello DevOps


Conclusion: ENTRYPOINT defines the executable and fixed arguments, while CMD provides default arguments that can be overridden at runtime.
## 10. COPY vs ADD
- COPY copies files and directories from the build context into the image.
- ADD supports additional behavior, including extracting supported local tar archives and handling remote URLs.
- COPY is preferred for ordinary file copying because its behavior is more explicit.
## 11. ARG vs ENV
- ARG defines a build-time variable.
- ENV defines an environment variable available to the container at runtime.
- Neither should be used to store secrets in an image.
## 12. EXPOSE vs Port Publishing
EXPOSE 3000 documents the application's intended listening port. It does not publish the port to the host.
The option -p 3000:3000 publishes the container port on host port 3000.
## 13. .dockerignore
The .dockerignore file excludes unnecessary files from the build context:
.git
.env
node_modules
*.log
.venv
__pycache__


This reduces unnecessary build-context content and helps prevent accidental inclusion of sensitive local files.
## 14. Image Inspection
docker history demo-api:v3
docker image inspect demo-api:v3


These commands help inspect image layers, configuration, environment variables, the working directory, exposed ports, and startup command.
## 15. Evidence Screenshots
Save screenshots of the following experiments in the evidence/ directory:
1. API root and health endpoint responses.
2. Runtime environment-variable response.
3. Cached layers during the demo-api:v2 build.
4. Dependency-install step running during the demo-api:v3 build.
5. Hello World and Hello DevOps outputs.
6. Docker image history or inspection output.
## 16. Conclusion
This project demonstrates Docker image creation, runtime configuration, non-root execution, layer caching, image inspection, and the practical difference between CMD and ENTRYPOINT. These concepts help build more maintainable and efficient containerized applications.
