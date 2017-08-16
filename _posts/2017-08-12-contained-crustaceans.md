---
layout: post
title: Contained Crustaceans
date: 2017-08-12 13:28
comments: true
summary: Setting up a development environment with docker for rust
categories: docker rust
---
[Ferris](http://www.rustacean.net/) the crab is a [crustacean](https://www.britannica.com/animal/crustacean) and  [Moby Dock](https://blog.docker.com/2013/10/call-me-moby-dock/), the whale, is a mammal. These mascots are both marine life and they share the moniker of "cartoon" - putting them in an interesting subcategory where it might benefit them to stick together! Let's spend a few minutes bringing them even closer together as we discuss setting up a development environment in [`Docker`](https://www.docker.com/)  for building applications in the [`Rust`](rust-lang.org) programming language.

#### Environmental Concerns
I've been using versions of the `Docker` toolchain in development for a few years now[^1], and overall I've enjoyed the experience. It's a great way to create a consistent environment for your application (and your peers). When I first started developing with `ruby` it was common practice to use something like [`rvm`](https://rvm.io/) or [`rbenv`](https://github.com/rbenv/rbenv) to contextually switch `ruby` versions based on some configuration in the application. The `rust` community has something similair with [`rustup`](https://github.com/rust-lang-nursery/rustup.rs)[^2] it provides a way to switch[^3] the version of `rust` so that the application can be appropriatly compiled with the intended version of the compiler.

Although this type of context switching can be extremely useful it is unlikely that the compiler version is the _only_ thing your application will rely on in its build environment as it matures. You may also have some specific package or file dependency like: `openssh`, `curl`, `libssh2` or a file named `foo` and while many systems might meet these dependencies, and perhaps you can easily[^4] facilitate them in your development environment, [Murphy's Law](https://en.wikipedia.org/wiki/Murphy%27s_law) tells us that your coworker and [CI server](https://www.thoughtworks.com/continuous-integration) are going to flounder.

#### Much Ado About Docker
`Docker` or more specifically "containerization" is the trending solution to this problem and with good reason. Think of a "running" container as an executing instance of your application passing the cycles sheltered from the whims and woes of other processes and attached to a filesystem that meets its wildest dreams. It's the pack up your stuff and live your days on a beach version of the process lifecycle.[^5]

#### My Container is Rusting... 
We can containerize `rust` applications to reep the benefits of our beach life (hopefully without the sand) and to do so all we really need is a `Dockerfile`[^6]. If you haven't installed `Docker` yet take a second to [visit the docs](https://docs.docker.com/engine/installation/) for installation instructions for your system. 

There are several prebuilt images for rust available on the [DockerHub](https://hub.docker.com/search/?isAutomated=0&isOfficial=0&page=1&pullCount=0&q=rust&starCount=0) but no official[^7] images yet, so this `Dockerfile` is built `FROM` the official `debian`[^8] image and installs `rust` from source. 

```Dockerfile
FROM debian:jessie

ENV RUST_VERSION=1.19.0
ENV RUST_TARGET=x86_64-unknown-linux-gnu

RUN \
  # Install tools needed from the package manager
  apt-get update && \
  DEBIAN_FRONTEND=noninteractive \
  apt-get install -y \
    ca-certificates \
    curl \
    build-essential \
    gcc && \

  # Use curl to download and install rust
  curl \
    -sO \
    https://static.rust-lang.org/dist/rust-$RUST_VERSION-$RUST_TARGET.tar.gz \
    && \
  tar -xzf rust-$RUST_VERSION-$RUST_TARGET.tar.gz && \
  ./rust-$RUST_VERSION-$RUST_TARGET/install.sh --without=rust-docs && \

  # Cleanup by removing files and utilities that are no longer needed
  DEBIAN_FRONTEND=noninteractive apt-get remove --purge -y curl && \
  rm -rf \
    ./rust-$RUST_VERSION-$RUST_TARGET \
    rust-$RUST_VERSION-$RUST_TARGET.tar.gz \
    /var/lib/apt/lists/* \
    /tmp/* \
    /var/tmp/*

# Creating a directory to work from
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

# Copy our app into that directory
COPY . /usr/src/app

# Build our app
CMD ["cargo", "build"]
```

This `image` that we've created is great, it installs rust and its default command is to build our application, lets put it through its paces by setting up a new application for development:

1. Drop the `Dockerfile` into an empty directory
2. Build the image and tag it with a name we can use to reference it later:
    `docker build -t my-rust-app-image .`
3. Once the image is built create a new project with `cargo` (in the current directory):
    ```sh
    docker run \
      --rm \                                  # declutter!
      -e USER='Super Dev' \                   # an env var for cargo init
      my-rust-app-image \                     # Our image name
      cargo init --bin --name the-best-app-rs # Override default command
    ```
    
If you do this you'll notice that the command seems to execute succesfully but when you check the filesystem there are **no new files** created. This is because the files that we created with `cargo` were created during the fleeting existance of the container and existed only within its filesystem. In order to persist the files on the host filesystem we need to mount the host filesystem into the container with a [`volume`](https://docs.docker.com/engine/admin/volumes/volumes/).

Our new command looks like this:

```sh
docker run \
  --rm \
  -e USER='Super Dev' \
  -v $PWD:/usr/src/app     # Special Sauce; mount the host working directory
  my-rust-app-image \
  cargo init --bin --name the-best-app-rs
```

And now running it produces the desired effect, we have initialized a new `cargo` project in the current working directory on the host machine.

#### Composed C**rust**aceans
We could continue to develop our application with `docker run` but our command is a bit tedious to say the least. [`Docker Compose`](https://docs.docker.com/compose/) solves this problem for us: it abstracts the `run` command so that we can deal with a much more agreeable `YAML` file. Let's add a `docker-compose.yml` to our new project:

```yml
version: '3.0'

services:
  bin:
    build:
      context: .
      dockerfile: Dockerfile
    command: cargo test
    volumes:
      - .:/usr/src/app
      - registry:/root/.cargo/registry # Secret Sauce

volumes:
  registry:
    driver: local
```

Now with this configuration[^9] in place we can execute:

````sh
docker-compose run --rm bin
````

and a container will start based on our image and `cargo test` will run the tests for our application; this is great for our development cycle because `cargo test` will `build` our application if there have been any changes since the last build and because *we're definitley writing tests* and we want them to run as well. Once the the container has finished the daemon will cleanup the container because we passed the `--rm` flag.[^10] 

#### Extra Special Sauce
You'll notice in the `docker-compose.yml` above that we add a volume that gets mounted into the container at `/root/.cargo/registry`; this is where `cargo` stores `crates`[^11] our application depends on. 

Adding a volume here rather then adding a `cargo build` layer[^12] to the `Dockerfile` prevents us from needing to rebuild the image after each change to the `Cargo.toml`. We can continue to develop normally and when `cargo` installs a new `crate` it will be cached for us in the `registry` volume.

#### fini
Now that we're done we have a `Dockerfile` that describes the filesystem our application needs and a `docker-compose.yml` that helps us by adding some nifty cacheing and mounting our working directory into the container so that we can see our code changes in real time. There's definitley some work that can be done to make this better (maybe `cargo test` is a better default command for the image) but it's a decent start and our process can begin to enjoy its new digs.

The example files can be found here: [blankenshipz/docker-rust-example](https://github.com/blankenshipz/docker-rust-example)

[^1]: The community has come a long way! `boot2docker`, `dinghy` and now [`Docker For Mac`](https://www.docker.com/docker-mac)
[^2]: `rustup` [refers](https://github.com/rust-lang-nursery/rustup.rs#how-rustup-works) to these sorts of utilities as "toolchain multiplexers"
[^3]: The switch can occur as a result of a user command, an environment variable, or a file: `.rustup-toolchain`. For a complete list [see the docs](https://github.com/rust-lang-nursery/rustup.rs#override-precedence)
[^4]: `touch foo`
[^5]: If you're interested in how `Docker` creates/runs containers take a look at [`runc`](https://github.com/opencontainers/runc): my current understanding is that the main parameters to `runc` are as simple as a "command" and a filesystem. It does the heavy lifting of process isoloation/namespacing.
[^6]: The Dockerfile [Reference](https://docs.docker.com/engine/reference/builder/) and [Best Practices](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/) are excellent resources to learn more about crafting images.
[^7]: You can learn more about "official" images [here](https://docs.docker.com/docker-hub/official_repos/)
[^8]: Typically the authors of docker images attempt to shrink the size of the image if at all possible, rather then using `debian` here a better alternative would be the minimalist distro `alpine` - according to [this issue](https://github.com/andrew-d/docker-rust-musl/issues/7) it should be possible to now install `rust` directly from the `alpine` package manger.
[^9]: You may have noticed that I choose `bin` as the name of our service; I've found that it's best when moving between projects to keep the names of services conventional and because `cargo` uses the terminology `bin` and `lib` I've found that those are the best names to use in a `cargo `project.
[^10]: Cleaning up is important here because each time we issue a `docker run` a **new** container is created, if we dont' remove it we'll end up with a bunch of orphaned containers on our system.
[^11]: A `crate` much like a `gem` in `ruby` is an external library, check out [crates.io](https://crates.io) for more information.
[^12]: A `Docker` "image" is a Union Filesystem where each of the statements in the `Dockerfile` create a `layer` and any changes at a specific `layer` invalidate the following layers requiring new layers to be created for that part of the image. [Here is a much better explanation](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/)

