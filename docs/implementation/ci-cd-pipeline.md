# CI/CD Pipeline

**CI**: Jenkins (per-service pipelines)  
**Registry**: Docker Registry (git-sha tagged images)  
**Delivery**: ArgoCD (GitOps)

---

## Pipeline Stages

Each service has its own `Jenkinsfile`. Pipelines are independent — a change to `order-service` never touches `inventory-service`.

```
Stage 1: Checkout
  → git clone service repo
  → resolve shared-kernel + shared-services packages

Stage 2: Build & Test
  → mvn clean verify
  → ArchUnit tests run (architecture violations = build failure)
  → JaCoCo coverage enforced

Stage 3: Docker Build
  → Multi-stage Dockerfile (Java 25)
  → Final image: boot jar + runtime only
  → Tagged by: <registry>/<service>:<git-sha>

Stage 4: Image Push
  → Push to Docker registry

Stage 5: Config Repo Update
  → Update environments/<env>/<service>/values.yaml
  → Set image.tag: <git-sha>
  → Commit and push to micro-config-repo
  → ArgoCD detects change and syncs
```

---

## Dockerfile Pattern

Multi-stage build — production image contains only the boot jar:

```dockerfile
# Stage 1: Build
FROM eclipse-temurin:25-jdk AS builder
WORKDIR /app
COPY . .
RUN mvn clean package -DskipTests \
    -pl <service>-boot \
    -am

# Stage 2: Runtime
FROM eclipse-temurin:25-jre
WORKDIR /app
COPY --from=builder /app/<service>-boot/target/*.jar app.jar
EXPOSE 8080 9090
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## Maven CI-Friendly Versions

Root service reactor pom must include `flatten-maven-plugin`:

```xml
<plugin>
  <groupId>org.codehaus.mojo</groupId>
  <artifactId>flatten-maven-plugin</artifactId>
  <configuration>
    <updatePomFile>true</updatePomFile>
    <flattenMode>resolveCiFriendliesOnly</flattenMode>
  </configuration>
  <executions>
    <execution>
      <id>flatten</id>
      <phase>process-resources</phase>
      <goals><goal>flatten</goal></goals>
    </execution>
    <execution>
      <id>flatten.clean</id>
      <phase>clean</phase>
      <goals><goal>clean</goal></goals>
    </execution>
  </executions>
</plugin>
```

This resolves CI-friendly `${revision}` placeholders before image build and config-repo promotion.

---

## Config Repo Update (Jenkins step)

```groovy
sh """
  git clone ${CONFIG_REPO_URL} config-repo
  cd config-repo
  yq e '.image.tag = "${GIT_SHA}"' -i \\
    environments/${ENV}/${SERVICE}/values.yaml
  git commit -am "ci: update ${SERVICE} image to ${GIT_SHA}"
  git push
"""
```

Jenkins updates **exactly one** `values.yaml` — the service that changed. Never cross-service.

---

## Architecture Gates

| Gate | Tool | Failure |
|---|---|---|
| Package boundary violations | ArchUnit | Build fails |
| Missing feature shape classes | ArchUnit | Build fails |
| ADR contract violations | ArchUnit | Build fails |
| Code coverage threshold | JaCoCo | Build fails |
