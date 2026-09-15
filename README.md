# Spark Runtime & Remote Kernel Platform

A hands-on study and implementation project focused on how Spark environments are packaged, delivered, and managed for Data Science / Data Engineering workloads.

---

## Project Goal

This project was started from a real engineering question:

> How are Spark images built for Data Science projects, and how can those environments be presented and managed through Jupyter on Kubernetes or YARN?

The focus is deliberately not on learning Spark APIs or building Spark applications.

The main focus is the platform side:

```text
Spark Runtime
      ↓
Container Image
      ↓
Jupyter Environment
      ↓
Remote Kernel
      ↓
Enterprise Gateway
      ↓
Kubernetes / YARN
```

The project will gradually turn this conceptual flow into a working implementation.

### Why Does This Problem Exist?

Imagine a company with many Data Scientists and Data Engineers. Different users may need different environments:

* One user needs only Jupyter and Python,
* another needs the Python Data Science ecosystem,
* another needs PySpark,
* another needs a particular Spark version,
* another may need to run the same environment on a shared cluster.

Installing everything manually on every machine quickly creates dependency and version problems.

The basic idea of this project is therefore:

> Build standardized environments once, package them as images, and make those environments available to users in a controlled way.

---

## The Learning Path

The project is intentionally built step by step.

### 1. Docker and Image Inheritance

First we establish the basic model:

```text
Docker
 ├── Image
 ├── Container
 ├── Dockerfile
 ├── FROM
 └── Image inheritance
```

The key question is:

> Why build every environment from scratch when common components can be shared through parent images?

### 2. Spark Images

Next we examine how a notebook-oriented environment can evolve into a Spark environment:

```text
foundation
    ↓
base-notebook
    ↓
minimal-notebook
    ↓
scipy-notebook
    ↓
pyspark-notebook
```

The important point is that each layer adds capabilities required by the next use case.

We will then build our own version of this idea and work with the required Spark versions:

* Spark 3.5.9
* Spark 4.0.4
* Spark 4.1.3
* Spark 4.2.0

### 3. Jupyter and Kernels

After understanding the images, we move to the user side:

```text
User
 ↓
JupyterLab
 ↓
Jupyter Server
 ↓
Kernel
```

The goal is to understand what a notebook kernel actually is and why a remote kernel becomes useful when computation should happen outside the user's local machine.

### 4. Enterprise Gateway

Enterprise Gateway becomes the bridge between the Jupyter side and the remote compute environment:

```text
Jupyter Server
       ↓
Enterprise Gateway
       ↓
KernelSpec / Process Proxy
       ↓
Remote Kernel
```

This is one of the central components of the project.

### 5. Kubernetes / YARN

Finally, the remote kernel needs somewhere to run.

The project will examine two possible backends:

```text
                 Enterprise Gateway
                    /            \
                   ↓              ↓
             Kubernetes          YARN
                   ↓              ↓
                  Pod        Application
                   ↓              ↓
                Kernel          Kernel
```

The objective is not to learn every feature of Kubernetes or YARN, but to understand the parts required for this architecture.

---

## Final Target Architecture

The intended end-to-end architecture is:

```text
                         USER
                          │
                          ▼
                     JupyterLab
                          │
                          ▼
                    Jupyter Server
                          │
                          ▼
                 Enterprise Gateway
                          │
                 ┌────────┴────────┐
                 │                 │
                 ▼                 ▼
             Kubernetes           YARN
                 │                 │
                 ▼                 ▼
                Pod          Spark Application
                 │                 │
                 ▼                 ▼
            Spark Kernel       Spark Kernel
                 │                 │
                 └────────┬────────┘
                          ▼
                    Spark Runtime
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
           3.5.9        4.0.4        4.1.3
                                       │
                                     4.2.0
```

This architecture is the destination. The repository will build it incrementally rather than creating everything at once.

---

## Repository Structure

```text
spark-platform/
│
├── README.md
│
├── docs/
│   ├── 01-docker-and-image-inheritance.md
│   ├── 02-spark-images.md
│   ├── 03-jupyter-and-kernels.md
│   ├── 04-enterprise-gateway.md
│   ├── 05-kubernetes-and-yarn.md
│   └── 06-end-to-end-architecture.md
│
├── docker/
├── kubernetes/
├── enterprise-gateway/
├── scripts/
└── tests/
```

The `docs/` directory explains what we are doing and why.

The implementation directories contain the actual Dockerfiles, configurations, manifests, scripts, and tests.

---

## Documentation

| Document | Purpose |
| :--- | :--- |
| [01 - Docker and Image Inheritance](docs/01-docker-and-image-inheritance.md) | Docker fundamentals required to understand the image chain |
| [02 - Spark Images](docs/02-spark-images.md) | Spark image structure and versioned Spark runtimes |
| [03 - Jupyter and Kernels](docs/03-jupyter-and-kernels.md) | Jupyter, kernels, and remote execution |
| [04 - Enterprise Gateway](docs/04-enterprise-gateway.md) | Gateway, KernelSpec, ProcessProxy, and kernel lifecycle |
| [05 - Kubernetes and YARN](docs/05-kubernetes-and-yarn.md) | Compute backends used by remote kernels |
| [06 - End-to-End Architecture](docs/06-end-to-end-architecture.md) | Complete platform architecture |

---

## Current Status

- [x] ~~Understand the problem and target architecture~~
- [x] ~~Understand Docker image vs. container~~
- [x] ~~Understand image inheritance~~
- [x] ~~Understand the Jupyter Docker Stacks image family concept~~
- [ ] Build the first local inheritance chain
- [ ] Build versioned Spark images
- [ ] Integrate Jupyter
- [ ] Integrate Enterprise Gateway
- [ ] Run remote kernels on Kubernetes
- [ ] Investigate / implement YARN integration
- [ ] Validate the complete flow

---

## Important Distinction

This project does not treat:

> "using Spark" 

and 

> "providing and managing Spark" 

as the same problem.

The first is about using Spark's APIs and processing data.

The second is about packaging the runtime, providing the environment to users, selecting versions, launching kernels, and managing where those kernels run.

This repository focuses primarily on the second problem.