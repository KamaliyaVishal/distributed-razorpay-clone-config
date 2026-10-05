# ⚙️ Distributed Razorpay Clone — Config Repository

Centralized, Git-backed configuration for the **[spring-boot-microservice-razorpay-clone](https://github.com/KamaliyaVishal/spring-boot-microservice-razorpay-clone)** payment platform.

This repository holds **no application code**. It is the configuration source that the platform's **Spring Cloud Config Server** (`config-server`) reads from and serves to every microservice at startup.

![Spring Cloud Config](https://img.shields.io/badge/Spring%20Cloud-Config%20Server-6DB33F?logo=spring)
![Git backed](https://img.shields.io/badge/Backend-GitHub-181717?logo=github)
![Kubernetes](https://img.shields.io/badge/Kubernetes-kind-326CE5?logo=kubernetes&logoColor=white)

---

## 📑 Table of Contents

1. [How it works](#-how-it-works)
2. [Repository layout](#-repository-layout)
3. [How config is resolved](#-how-config-is-resolved)
4. [Pointing the config server at this repo](#-pointing-the-config-server-at-this-repo)
5. [How services consume it](#-how-services-consume-it)
6. [Verifying the config server](#-verifying-the-config-server)
7. [Changing configuration](#-changing-configuration)
8. [Startup order on Kubernetes](#-startup-order-on-kubernetes)
9. [Security notes](#-security-notes)

---

## 🔄 How it works

```
┌──────────────────────────────┐
│  distributed-razorpay-clone- │
│  config  (this GitHub repo)  │
└──────────────┬───────────────┘
               │ git clone / pull
               ▼
        ┌─────────────┐
        │config-server│   Spring Cloud Config Server
        └──────┬──────┘
               │ HTTP: /{service-name}/{profile}
   ┌───────────┼────────────┬──────────────┬──────────────┐
   ▼           ▼            ▼              ▼              ▼
api-gateway  merchant-    payment-     operations-     vault-
             service      service      service         service
```

1. A microservice starts with only a minimal bootstrap configuration (its name and the config server location).
2. It asks `config-server` for its configuration.
3. `config-server` fetches this repository from GitHub and returns the merged result.
4. The service finishes starting using the received properties.

---

## 📂 Repository layout

| File | Applies to |
| --- | --- |
| `application.yaml` | Shared defaults for **every** service |
| `api-gateway.yaml` | `api-gateway` only |
| `merchant-service.yaml` | `merchant-service` only |
| `payment-service.yaml` | `payment-service` only |
| `operations-service.yaml` | `operations-service` only |
| `vault-service.yaml` | `vault-service` only |

The file name (without extension) must match the service's `spring.application.name`. That is how the config server decides which file belongs to which service.

---

## 🧩 How config is resolved

For a service named `payment-service`, the config server merges, in order:

1. `application.yaml`: shared values (lowest priority)
2. `payment-service.yaml`: service-specific values (**override** the shared ones)

If a property exists in both files, the service-specific file wins. Put anything common to all services (for example, common Spring, actuator or observability settings) in `application.yaml`, and keep service-specific values (ports, database settings, routes and so on) in that service's own file.

---

## 🛠 Pointing the config server at this repo

In the `config-server` module of the main project, the Git backend is configured like this:

```yaml
spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/KamaliyaVishal/distributed-razorpay-clone-config
          default-label: main
          clone-on-start: true
server:
  port: 8888
```

> Adjust the property values to match your `config-server` setup (for example, a different port or label).

The `config-server` application also needs `@EnableConfigServer` on its main class and the `spring-cloud-config-server` dependency.

---

## 🔌 How services consume it

Each microservice imports its configuration from the config server, for example in its local `application.yaml`:

```yaml
spring:
  application:
    name: payment-service        # must match the file name in this repo
  config:
    import: "configserver:http://config-server:8888"
```

Under Kubernetes, `config-server` is the name of the Kubernetes `Service` for the config server, so the URL above resolves inside the cluster. For local runs outside Kubernetes, use `http://localhost:8888` instead.

---

## ✅ Verifying the config server

Once `config-server` is running, you can see exactly what a service will receive:

```bash
# merged config for payment-service
curl http://localhost:8888/payment-service/default

# a single file, rendered as YAML
curl http://localhost:8888/payment-service-default.yaml
```

On Kubernetes with kind, port-forward first:

```bash
kubectl port-forward svc/config-server 8888:8888
```

If the response lists both `payment-service.yaml` and `application.yaml` as property sources, resolution is working.

---

## ✏️ Changing configuration

1. Edit the relevant YAML file in this repository.
2. Commit and push to `main`.
3. Restart the affected service so it fetches the new values:

```bash
kubectl rollout restart deployment/payment-service
```

Because the configuration lives in Git, there is no need to rebuild any Docker image, and every change has a commit history that can be reviewed or reverted.

---

## ☸️ Startup order on Kubernetes

Every service needs `config-server` at startup, so it **must be running and ready before the other services**.

The main project handles this by applying `kustomization.yaml` in two phases:

1. **Phase 1:** deploy data stores and `config-server` only (service manifests commented out), then wait until `config-server` is `Ready`.
2. **Phase 2:** uncomment the five service manifests (`api-gateway`, `merchant-service`, `payment-service`, `operations-service`, `vault-service`) and run `kubectl apply -k .` again.

See the **Deploying on Kubernetes** section of the [main README](https://github.com/KamaliyaVishal/spring-boot-microservice-razorpay-clone/blob/main/README.md#deploying-on-kubernetes) for the full steps.

---

## 🔐 Security notes

- This repository is **public**, so never commit real secrets (database passwords, API keys, encryption keys, tokens) in plain text.
- Provide sensitive values through Kubernetes `Secret`s and environment variables, and reference them from the YAML files with placeholders such as `${DB_PASSWORD}`.
- For private config repositories, give the config server a read-only deploy key or personal access token via `spring.cloud.config.server.git.username` / `password`, supplied from a Secret.

---

**Related repository:** [spring-boot-microservice-razorpay-clone](https://github.com/KamaliyaVishal/spring-boot-microservice-razorpay-clone), the services that consume this configuration.
