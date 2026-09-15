# Docker and Image Inheritance

## 1. The Problem: "It Works on My Machine"

A Data Engineer can install everything needed for a project directly on
a computer:

``` text
Python
Java
Spark
PySpark
Jupyter
Pandas
...
```

That may work perfectly on one machine.

The problem appears when another developer needs the same environment.

For example:

``` text
Machine A
Python 3.x
Java 17
Spark 4.x

Machine B
Python 3.x
Java 11
Spark 3.x
```

The same project can now behave differently or fail because the
environments are different.

The first requirement is therefore:

> **Make the software environment reproducible and portable.**

------------------------------------------------------------------------

## 2. What Does Docker Give Us?

The basic idea is to package the environment required by an application.

Instead of asking every developer to manually install:

``` text
Python
Java
Spark
libraries
configuration
environment variables
```

we can describe an environment and build a **Docker image** containing
it.

Conceptually:

``` text
Application
    +
Required environment
    ↓
Docker Image
```

The image can then be used to create a running container.

This is why Docker becomes useful in Data Engineering environments: the
environment itself becomes something that can be built, versioned,
distributed, and reused.

------------------------------------------------------------------------

## 3. Image and Container Are Not the Same Thing

This distinction is fundamental.

### Image

An image is the packaged environment that can be used as the basis for
running a container.

Think of it as the prepared template.

### Container

A container is a running instance created from an image.

The basic relationship is:

``` text
Image
  ↓
docker run
  ↓
Container
```

For example:

``` bash
docker run ubuntu
```

uses an Ubuntu image to create and run a container.

So when we later talk about image inheritance, we are talking about
**images**, not containers.

------------------------------------------------------------------------

## 4. What Is Image Inheritance?

Suppose we start with an environment containing Ubuntu.

Then we need Python.

Then we need Jupyter.

Then we need Data Science libraries.

Instead of rebuilding everything from zero every time, we can build
progressively:

``` text
Ubuntu
  ↓
Python
  ↓
Jupyter
  ↓
Data Science
  ↓
PySpark
```

In Docker, a child image can use a parent image through the `FROM`
instruction.

For example:

``` dockerfile
FROM ubuntu:24.04

RUN apt-get update &&     apt-get install -y python3 python3-pip
```

Then another image can be built on top of that:

``` dockerfile
FROM my-foundation:1.0

RUN pip install --user jupyterlab notebook
```

And another:

``` dockerfile
FROM my-base-notebook:1.0

RUN pip install --user pandas numpy matplotlib scikit-learn
```

And finally:

``` dockerfile
FROM my-data-science:1.0

RUN pip install --user pyspark
```

The resulting chain is:

``` text
my-foundation
      ↓
my-base-notebook
      ↓
my-data-science
      ↓
my-pyspark
```

The child image gets the environment provided by its parent and adds its
own requirements.

------------------------------------------------------------------------

## 5. Why Not Put Everything in One Dockerfile?

We could.

For example:

``` dockerfile
FROM ubuntu:24.04

RUN apt install ...
RUN pip install jupyterlab
RUN pip install pandas numpy matplotlib scikit-learn
RUN pip install pyspark
```

This can produce a working image.

But imagine a company with different users:

``` text
User A → Jupyter + Python
User B → Jupyter + Data Science
User C → Jupyter + Data Science + PySpark
```

If we build every environment independently, common components are
repeatedly defined.

Inheritance allows us to share the common foundation:

``` text
                foundation
                /                        ↓           ↓
       base-notebook    another image
               ↓
        data-science
               ↓
            pyspark
```

The important design principle is:

> **Put common requirements in the parent; put specialized requirements
> in child images.**

------------------------------------------------------------------------

## 6. The Real Reason: Avoid Repeating the Same Work

Suppose every image needs:

``` text
Ubuntu
Python
Jupyter
common configuration
```

Without inheritance, each image has to establish these requirements
again.

With inheritance:

``` text
foundation
    ↓
base-notebook
    ↓
data-science
    ↓
pyspark
```

the common foundation is established once and reused.

This is closely related to the DRY principle:

> **Don't Repeat Yourself.**

The goal is not simply to make the Dockerfile shorter.

The deeper goal is to make the environment architecture:

-   reusable,
-   consistent,
-   easier to maintain,
-   easier to version,
-   easier to extend.

------------------------------------------------------------------------

## 7. Image Inheritance Is Not Container Inheritance

This is an important distinction.

When we write:

``` dockerfile
FROM my-foundation:1.0
```

we are creating a new **image** based on another image.

Inheritance happens during image construction.

Later, when we run:

``` bash
docker run my-pyspark:1.0
```

we create a **container** from the resulting image.

So:

``` text
Dockerfile
    │
    │ FROM
    ▼
Parent Image
    │
    │ add requirements
    ▼
Child Image
    │
    │ docker run
    ▼
Container
```

A container is not the child of another container.

------------------------------------------------------------------------

## 8. How This Appears in Jupyter Docker Stacks

The original study that led to this project examined a Jupyter Docker
Stacks image inheritance chain.

The conceptual structure is:

``` text
docker-stacks-foundation
          ↓
     base-notebook
          ↓
    minimal-notebook
       /              ↓          ↓
scipy-notebook  r-notebook
      ↓
datascience-notebook
      ↓
pyspark-notebook
      ↓
all-spark-notebook
```

The exact branches are not the main point.

The important idea is:

> **Different environments can share a common base and then branch
> according to their requirements.**

For example, the study identified the following conceptual progression:

``` text
foundation
  → Ubuntu + common environment/user setup

base-notebook
  → Jupyter capabilities

minimal-notebook
  → common notebook utilities

scipy-notebook
  → Python Data Science ecosystem

pyspark-notebook
  → Spark-related runtime components
```

The `pyspark-notebook` image is therefore not an isolated Spark
installation. It is the result of building on an existing notebook/Data
Science environment and adding the Spark ecosystem.

------------------------------------------------------------------------

## 9. Why Does the Spark Notebook Environment Need More Than Python?

A natural question is:

> "If I want PySpark, why not just install PySpark with `pip`?"

Installing a Python package can be enough for some local use cases.

But the complete runtime environment has more pieces than a Python
import.

The study connected PySpark to the Spark runtime roughly as:

``` text
Python
   ↓
PySpark
   ↓
Spark
   ↓
JVM
```

This is why a Spark-oriented environment also needs the runtime
components required by Spark.

The important lesson for this project is:

> **Packaging a Spark environment is about packaging a compatible
> runtime environment, not merely making `import pyspark` succeed.**

------------------------------------------------------------------------

## 10. `pip install` vs. Building an Image

Another important question from the study was:

> "If I can install packages with `pip`, why put them in the image?"

There are two different goals.

### Manual installation

``` bash
pip install delta-spark
pip install some-package
```

This changes the environment of the current runtime/container.

### Image-based installation

``` dockerfile
FROM pyspark-notebook

RUN pip install delta-spark
RUN pip install some-package
```

Now the installation becomes part of a reproducible image build.

The company can build the image once and distribute that standardized
environment to many users.

Conceptually:

``` text
Dockerfile
    ↓
docker build
    ↓
Company Image
    ↓
Many users
```

This is particularly useful when many people need the same dependencies.

------------------------------------------------------------------------

## 11. Why Does the Foundation Image Exist?

Another question that appeared during the study was:

> "If an Ubuntu image already exists, why create another foundation
> image?"

Because the organization may have common requirements that belong above
the generic operating-system layer.

The conceptual structure becomes:

``` text
Ubuntu
   ↓
Company/Jupyter foundation
   ↓
Notebook environment
   ↓
Specialized environments
```

The foundation can establish common components that many child images
need.

In the studied Jupyter environment, the foundation was discussed in
terms of:

``` text
Ubuntu
micromamba
jovyan
```

### Ubuntu

Provides the Linux environment on which the rest of the software stack
is built.

### micromamba

Provides an environment/package-management mechanism used by the image
ecosystem.

### jovyan

A normal Linux user used by the Jupyter Docker environment rather than
running notebook workloads as root.

The important design question is not:

> "Why exactly these three words?"

but:

> **"What common requirements should every image in this family
> inherit?"**

------------------------------------------------------------------------

## 12. Why Use a Normal User?

Running everything as `root` can make experimentation easy, but it is
not a good default for a shared notebook environment.

A shared environment benefits from having a normal user identity.

That is why the studied foundation contains a `jovyan` user.

Conceptually:

``` text
Container
   ↓
Normal notebook user
   ↓
Jupyter / user workloads
```

This separates the idea of building/configuring the environment from the
idea of running user workloads inside it.

------------------------------------------------------------------------

## 13. Jupyter: Why Is It in the Image?

Another question from the study was:

> "If my actual goal is Spark, why do I need Jupyter?"

Because Spark and Jupyter solve different problems.

``` text
Jupyter
→ interactive development environment

Spark
→ distributed data-processing engine

PySpark
→ Python interface to Spark
```

A Data Scientist may want to explore data interactively:

``` text
JupyterLab
    ↓
Notebook
    ↓
Python
    ↓
PySpark
    ↓
Spark
```

Therefore a Spark notebook image is not simply "Spark in Docker".

It is a prepared **interactive development environment containing the
components needed to work with Spark**.

------------------------------------------------------------------------

## 14. JupyterHub Is a Different Concern

The study also raised:

> "If multiple Data Scientists use the system, does everyone run Jupyter
> locally?"

Not necessarily.

For a centralized environment, JupyterHub can act as the multi-user
entry point.

Conceptually:

``` text
Many Users
    ↓
JupyterHub
    ↓
Individual Jupyter environments
```

This is different from image inheritance.

Image inheritance answers:

> **How do we build the environment?**

JupyterHub answers:

> **How do multiple users access a Jupyter service?**

These concerns can later meet in the larger architecture of this
project.

------------------------------------------------------------------------

## 15. What Does the Image Contain?

A useful mental model from the study is to think in layers of
responsibility:

``` text
Linux environment
       ↓
Python / environment management
       ↓
Jupyter
       ↓
Data Science libraries
       ↓
Spark runtime
       ↓
Project-specific dependencies
```

For example, a company-specific Data Engineering image might
conceptually be:

``` text
pyspark-notebook
       +
company requirements
       ↓
company-data-engineering-image
```

This is much easier to reason about than starting from an empty image
every time.

------------------------------------------------------------------------

## 16. A Real Local Example

The study eventually turned the inheritance concept into a small
project.

The structure was:

``` text
docker-inheritance-demo/
│
├── foundation/
│   └── Dockerfile
│
├── base-notebook/
│   └── Dockerfile
│
├── data-science/
│   └── Dockerfile
│
├── pyspark/
│   └── Dockerfile
│
└── docker-compose.yml
```

### Foundation

``` dockerfile
FROM ubuntu:24.04

RUN apt-get update &&     apt-get install -y python3 python3-pip &&     rm -rf /var/lib/apt/lists/*

RUN useradd -m -s /bin/bash jovyan

WORKDIR /home/jovyan
USER jovyan
```

This creates the common base.

### Base Notebook

``` dockerfile
FROM my-foundation:1.0

RUN pip install --user jupyterlab notebook

EXPOSE 8888

CMD ["python3", "-m", "jupyterlab", "--ip=0.0.0.0", "--no-browser"]
```

This adds Jupyter.

### Data Science

``` dockerfile
FROM my-base-notebook:1.0

RUN pip install --user     pandas     numpy     matplotlib     scikit-learn
```

This adds common Data Science libraries.

### PySpark

``` dockerfile
FROM my-data-science:1.0

RUN pip install --user pyspark
```

Now the final environment contains the inherited components plus
PySpark.

------------------------------------------------------------------------

## 17. Build Order Matters

When using locally built images as parents, Docker must be able to find
the parent image.

Therefore the chain is built from the bottom upward:

``` bash
docker build -t my-foundation:1.0 ./foundation
docker build -t my-base-notebook:1.0 ./base-notebook
docker build -t my-data-science:1.0 ./data-science
docker build -t my-pyspark:1.0 ./pyspark
```

Then:

``` bash
docker images
```

should show the resulting images.

The important mental model is:

``` text
Build foundation
      ↓
Build child from foundation
      ↓
Build child from previous child
      ↓
Build final image
```

------------------------------------------------------------------------

## 18. What Happens When We Run the Final Image?

We can run:

``` bash
docker run -p 8888:8888 my-pyspark:1.0
```

The container is created from the final image.

But that final image contains the environment inherited through the
entire chain:

``` text
my-pyspark
    ↓
my-data-science
    ↓
my-base-notebook
    ↓
my-foundation
    ↓
Ubuntu
```

So the container does not need to know that these were separate build
stages.

From the runtime perspective, it receives the final assembled image.

------------------------------------------------------------------------

## 19. What Does a Registry Add?

In a real organization, images normally need to be distributed.

Instead of keeping:

``` text
my-pyspark:1.0
```

only on one developer's machine, an organization can publish its image
to a registry.

Conceptually:

``` text
Company Registry
│
├── company/foundation:1.0
├── company/base-notebook:1.0
├── company/data-science:1.0
└── company/pyspark:1.0
```

Another developer can then retrieve the image:

``` bash
docker pull company/pyspark:1.0
```

and run it:

``` bash
docker run company/pyspark:1.0
```

This is where image packaging becomes a platform-level capability rather
than just a local development trick.

------------------------------------------------------------------------

## 20. The Mental Model to Keep

The most important question is not:

> "What does every box in the diagram mean?"

Instead, ask:

> **"What common environment already exists, and what new requirement do
> I need to add?"**

For example:

``` text
I need Linux
    ↓
foundation

I need Jupyter
    ↓
base-notebook

I need Data Science libraries
    ↓
data-science

I need Spark
    ↓
pyspark
```

This is the core of image inheritance.

------------------------------------------------------------------------

## 21. Connection to the Rest of This Project

Image inheritance solves one part of our larger problem:

> **How do we build standardized Spark environments?**

It does not yet answer:

> How does a user select one of those environments from Jupyter?

or:

> Where does the Spark kernel actually run?

Those questions lead to the next parts of the project:

``` text
Docker / Images
      ↓
Spark Images
      ↓
Jupyter / Kernels
      ↓
Enterprise Gateway
      ↓
Kubernetes / YARN
```

So this document is the foundation for the rest of the architecture.
