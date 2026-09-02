# Day 2 — Docker Revision Notes

## 1. Multi-Stage Builds
- **What it does**: Uses multiple `FROM` stages in one Dockerfile so build tools/dependencies don't bloat the final image. Only the artifact you need gets copied into the final, slim stage.
- **Example**:
```dockerfile
# Stage 1: build
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: run
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```
- **Why it matters (interview angle)**: Final image doesn't carry Maven, source code, or build cache — smaller attack surface, smaller size, faster pulls in CI/CD.

## 2. Non-Root Dockerfile
- **Why**: Running containers as root is a security risk — if the container is compromised, the attacker has root inside it (and potentially escapes to host root depending on misconfig).
- **Example**:
```dockerfile
FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --chown=appuser:appgroup target/app.jar app.jar
USER appuser
ENTRYPOINT ["java", "-jar", "app.jar"]
```
- **Interview point**: Mention this ties directly into DevSecOps — container image scanners (Trivy, Grype) and Kubernetes `PodSecurityStandards`/`securityContext.runAsNonRoot` often flag or block root containers.

## 3. Layer Caching
- **How Docker builds layers**: Each instruction (`FROM`, `RUN`, `COPY`, etc.) creates a cached layer. If a layer's input hasn't changed, Docker reuses the cache instead of rebuilding.
- **Best practice — order matters**:
```dockerfile
# BAD: any source change invalidates dependency install
COPY . .
RUN mvn install

# GOOD: dependency layer cached separately from source code
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package
```
- **Why it matters**: Faster builds in CI — dependencies aren't re-downloaded every time only source code changes.

## 4. `.dockerignore`
- **What it does**: Same concept as `.gitignore` — excludes files/folders from the build context sent to the Docker daemon.
- **Typical entries**:
```
.git
node_modules
target
*.log
.env
Dockerfile
README.md
```
- **Why it matters**: Smaller build context = faster builds, and prevents accidentally baking secrets (`.env`) or bloated folders (`node_modules`, `.git`) into the image.

## 5. Networking Modes
- **bridge** (default): Container gets its own network namespace, connected to a virtual bridge on the host. Containers on the same bridge network can talk to each other by container name (via Docker's embedded DNS).
- **host**: Container shares the host's network namespace directly — no port mapping needed, but less isolation, port conflicts possible.
- **none**: No networking at all — fully isolated container.
- **overlay**: Used in Docker Swarm (multi-host networking) — not relevant for single-host but good to mention it exists.
- **Interview point**: Default bridge network doesn't have automatic DNS between containers unless you create a **user-defined bridge network** (`docker network create mynet`) — only then can containers resolve each other by name.

## 6. Volumes vs Bind Mounts
| | Volumes | Bind Mounts |
|---|---|---|
| Managed by | Docker (`/var/lib/docker/volumes/`) | User (any host path) |
| Portability | Portable across environments | Tied to host filesystem structure |
| Use case | Persistent data (DB storage) | Local dev (mount source code for hot reload) |
| Command | `docker volume create myvol` then `-v myvol:/data` | `-v /host/path:/container/path` |
- **Interview point**: Volumes are the recommended way for production persistent data because Docker manages lifecycle, backup, and driver plugins (e.g., for cloud storage). Bind mounts are great for local dev but risky in prod (host path dependency, permission issues).

## 7. Debugging Commands
```bash
docker exec -it <container> sh        # get a shell inside running container
docker exec -it <container> bash      # if bash is available instead of sh

docker logs <container>               # view stdout/stderr logs
docker logs -f <container>            # follow logs live
docker logs --tail 100 <container>    # last 100 lines

docker inspect <container>            # full JSON metadata: network, mounts, env, IP, etc.
docker inspect -f '{{.NetworkSettings.IPAddress}}' <container>   # extract specific field

docker ps -a                          # all containers including stopped
docker stats                          # live CPU/memory usage per container
```
- **Interview point for `docker inspect`**: It's your go-to when a container's networking or mount isn't behaving as expected — shows the actual runtime config, not just what you wrote in the Dockerfile/compose file.

---

## Quick Self-Test (do this without looking)
1. Why would a multi-stage build reduce your final image's vulnerability count when scanned by Trivy?
2. What's the actual difference in how Docker treats a `RUN` layer when only your source code (not `pom.xml`) changes, assuming correct Dockerfile ordering?
3. On the default `bridge` network, can two containers reach each other by container name? What do you need instead?
4. When would you choose a bind mount over a volume, and why is that risky in production?
