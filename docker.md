
## Q1. What is Docker?

**Answer:**

Docker is a containerization platform that packages an application with its runtime dependencies into an image and runs that image as a container, providing consistency across environments.

---

## Q2. Image vs Container?

**Answer:**

An image is an immutable template containing the application and required filesystem/runtime components. A container is a runtime instance created from that image with its own writable layer and runtime state.

---

## Q3. What happens during `docker build`?

**Answer:**

Docker reads the Dockerfile and build context, executes the Dockerfile instructions, creates filesystem layers, and produces an image identified by a repository and tag.

---

## Q4. What is Docker build context?

**Answer:**

The build context is the set of files available to the Docker build. In `docker build -t app:1.0 .`, the `.` represents the current directory as the build context.

---

## Q5. Why use `.dockerignore`?

**Answer:**

It prevents unnecessary files from being sent into the build context. This improves build performance, reduces image-build overhead, and helps prevent accidental inclusion of files such as secrets.

---

## Q6. Why use multi-stage builds?

**Answer:**

Multi-stage builds allow us to separate build dependencies from runtime dependencies. Maven and JDK tooling can remain in the build stage while the final image contains only the required runtime and application artifact.

---

## Q7. CMD vs ENTRYPOINT?

**Answer:**

`ENTRYPOINT` defines the primary executable of the container, while `CMD` provides default arguments or a default command that can be overridden depending on how the container is started.

---

## Q8. COPY vs ADD?

**Answer:**

I generally use `COPY` for predictable file copying. `ADD` has additional behaviors, so I use it only when those features are specifically required.

---

## Q9. ARG vs ENV?

**Answer:**

`ARG` is primarily a build-time variable, while `ENV` defines environment variables available to the image/container runtime.

---

## Q10. How do you reduce Docker image size?

**Answer:**

I use minimal runtime base images, multi-stage builds, `.dockerignore`, appropriate layer ordering, and remove unnecessary build tools and packages from the final image.

---

## Q11. How do you secure Docker images?

**Answer:**

I use minimal trusted base images, non-root users, vulnerability scanning with tools such as Trivy, secret protection, immutable tagging, controlled registries, least privilege, and regular base-image updates.

---

## Q12. How do you troubleshoot a stopped container?

**Answer:**

I first check `docker ps -a`, then inspect `docker logs`, the container exit code, and `docker inspect`. I verify the entrypoint, environment variables, mounted files, permissions, and resource constraints.

---

## Q13. How do you troubleshoot a container that is running but inaccessible?

**Answer:**

I check container port bindings, application listening ports, Docker network configuration, health status, logs, and connectivity using tools such as `curl`.

---

## Q14. What causes `port is already allocated`?

**Answer:**

Usually another process or container has already bound the requested host port. I identify the process/container using that port and either stop it or use another host port.

---

## Q15. How do Docker volumes work?

**Answer:**

Volumes provide persistent storage managed separately from the container lifecycle. Data can survive container deletion and be mounted into another container.

---

## Q16. How do containers communicate?

**Answer:**

Containers connected to the same Docker network can communicate through the network. User-defined Docker networks also provide DNS-based service-name discovery.

---

## Q17. How do you handle Docker secrets?

**Answer:**

I avoid hardcoding secrets in Dockerfiles, images, Git repositories, or pipeline scripts. I use a secret-management solution appropriate to the platform and inject secrets at runtime.

---

## Q18. Why shouldn't we rely on `latest`?

**Answer:**

`latest` is mutable and makes artifact traceability difficult. I prefer versioned, build-number, or Git-SHA-based image tags so that deployments can be traced to a specific artifact.

---

## Q19. Docker vs Kubernetes?

**Answer:**

Docker provides container-building and container-running capabilities, while Kubernetes provides orchestration capabilities such as scheduling, scaling, service discovery, rollout management, and self-healing. Kubernetes can use container runtimes such as containerd underneath.

---

## Q20. How does Docker fit into CI/CD?

**Answer:**

After code checkout, build, testing, quality checks, and security checks, Jenkins can build the Docker image, scan it, tag it with a traceable identifier, publish it to a registry, and then deploy or promote that artifact through the deployment process.

---

# 105. Senior Interview Scenario

### Interviewer:

> A production container suddenly stopped. What would you do?

### Answer:

I would first determine whether the issue is isolated to one container or affects the service more broadly.

I would check:

```bash
docker ps -a
docker logs <container>
docker inspect <container>
docker stats
```

I would verify the exit code and investigate application logs, configuration, permissions, mounted volumes, network connectivity, and resource limits.

If it is an OOM-related issue, I would check memory usage and container limits. If the application configuration is incorrect, I would identify the configuration change and correct it.

After fixing the root cause, I would restart or redeploy the container through the controlled deployment process and validate application health.

Finally, I would document the RCA and add preventive controls such as health checks, monitoring, resource tuning, or CI/CD validation.

---

# 106. Senior Interview Scenario — Image Vulnerability

### Interviewer:

> Trivy finds a CRITICAL vulnerability in your production image. What do you do?

### Answer:

I would first identify the vulnerable package and determine whether it originates from the application dependency or the base image.

Then I would check whether a fixed version is available.

If it is a base-image vulnerability, I would update the base image and rebuild the image.

If it is an application dependency, I would upgrade the dependency where possible and rerun the security scan.

I would not simply ignore the vulnerability. I would evaluate severity, exploitability, exposure, available remediation, and organizational risk policy.

After remediation:

```text
Rebuild
 ↓
Rescan
 ↓
Test
 ↓
Publish
 ↓
Deploy
```

---

# 107. Senior Interview Scenario — Production Traceability

### Interviewer:

> How do you know which Docker image is running in production?

### Answer:

I use immutable or traceable image tags linked to the Git commit and CI build.

For example:

```text
Git SHA
   ↓
Jenkins Build 152
   ↓
hotstar:git-a81f23c
   ↓
Registry
   ↓
Production Deployment
```

This allows me to identify the exact source and build associated with the deployed artifact.

---

# 108. Hands-On Completion Checklist

## Fundamentals

```text
[ ] Docker installed
[ ] Docker daemon verified
[ ] hello-world executed
[ ] docker ps understood
[ ] docker images understood
```

## Dockerfile

```text
[ ] FROM
[ ] WORKDIR
[ ] COPY
[ ] ADD
[ ] RUN
[ ] CMD
[ ] ENTRYPOINT
[ ] EXPOSE
[ ] ARG
[ ] ENV
[ ] USER
[ ] HEALTHCHECK
```

## Containers

```text
[ ] docker run
[ ] docker stop
[ ] docker start
[ ] docker restart
[ ] docker rm
[ ] docker logs
[ ] docker exec
[ ] docker inspect
```

## Storage

```text
[ ] Named volumes
[ ] Bind mounts
[ ] Volume persistence
```

## Networking

```text
[ ] Port mapping
[ ] Docker networks
[ ] Container DNS
[ ] Container communication
```

## Compose

```text
[ ] compose.yaml
[ ] docker compose up
[ ] docker compose down
[ ] Environment variables
[ ] Health checks
```

## Security

```text
[ ] Non-root user
[ ] Trivy image scan
[ ] Secret protection
[ ] Minimal image
[ ] Immutable tags
```

## Enterprise

```text
[ ] Docker Hub / Registry
[ ] Jenkins integration
[ ] Kubernetes integration
[ ] Artifact traceability
[ ] Failure simulation
[ ] RCA
[ ] Rollback
```

---
