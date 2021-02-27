dev-env
=======

A script to create and run programs in a development environment.

Introduction
------------

All repositories should have a single-step build which happens automatically for the average user.
This project was created to assist with testing code across multiple Linux distributions while maintaining a
self-contained repository.
With `dev-env`, the workflow in your project looks something like this:

    travis@posh:~/my-repo$ ./tools/dev-env
    ... building messages ...
    travis@527e37a36be4:/my-repo$ make # or whatever you use

This approach is as unopinionated as it can be.
It works across any programming language, build tool, library, whatever; if it can be containerized, it can work with
`dev-env`.
This approach does not hurt non-containerized builds; if people want to build for their host platform, they can do that
by running the same build without a `dev-env`.

**A Typical Project Structure**

    my-repo
    ├── src
    │   ├── ... your ...
    │   └── ... stuff ...
    ├── ... other things ...
    │   └── ...
    └── tools
        ├── dev-env
        ├── docker
        │   ├── arch
        │   │   └── Dockerfile
        │   ├── debian-10.8
        │   │   └── Dockerfile
        │   ├── opensuse-15.2
        │   │   └── Dockerfile
        │   ├── ubuntu-18.04
        │   │   └── Dockerfile
        │   └── ubuntu-20.04
        │       └── Dockerfile
        ├── ... probably more ...
        └── make-docs

Usage
-----

### Initial Project Setup

To use `dev-env` in your project, simply copy and paste the `dev-env` script into your repository in a directory one
level deep.
This folder can have any name, but I like the name `tools`.
If you are extremely lazy, just copy and paste this text:

    mkdir -p tools \
     && wget 'https://raw.githubusercontent.com/tgockel/dev-env/trunk/dev-env' -O tools/dev-env \
     && chmod +x tools/dev-env

> **CUSTOMIZATION**
>
> If you do not want the `dev-env` script to be one directory deep, change `PROJECT_ROOT` in the script to `readlink`
> something different than `(dirname $0)/..`.

After you have the script, you need to create the environment-building `Dockerfile`s.
These live in a directory sibling to the `dev-env` script called `docker`.
Each folder in this directory represents a potential target platform you want to support builds on.
Name them something sensible -- I follow the pattern `${distro}-${version}` (e.g. `ubuntu-20.04`, `opensuse-15.2`).
This directory should contain a `Dockerfile` and all the requisite assets to build the image for that platform.

> **CUSTOMIZATION**
>
> If you want your Docker images to live somewhere else, change `DOCKER_ROOT` to look somewhere else.

The contents of that `Dockerfile` depend entirely on what you are doing.
But for this example, let's consider a normal C++ project: configured with [CMake](https://cmake.org/), compiled with
[GCC](https://gcc.gnu.org/), built with [Ninja](https://ninja-build.org/), and dependent on the
[Boost Context](https://github.com/boostorg/context) library.
We'll throw in [GDB](https://www.gnu.org/software/gdb/), too, since it is such a commonly-used tool.

For this example, the `tools/docker/ubuntu-20.04/Dockerfile` would look something like this:

    FROM ubuntu:20.04

    RUN apt-get update \
     && apt-get install --yes \
        cmake \
        g++ \
        gdb \
        libboost-context-dev \
        ninja-build \
     && rm -rf /var/lib/apt/lists/*

With this file created, you can run `./tools/dev-env` and, after a brief period where your image is built, you will be
dropped into a container with your project's directory mounted.
Build away!

### Running Inside a Container

Unless you provide your own `CMD` or `ENTRYPOINT` in the `Dockerfile`, `dev-env` will drop you into a shell (usually
Bash).
If you want to run a program, pass `--` and the remaining arguments will passed to `docker exec`.
This allows you to easily run programs inside of the Docker image.

For example, if you have another script called `tools/make-package` which creates a package for whatever Linux distro
you are on, run it like so:

    $> ./tools/dev-env -- ./tools/make-package

Or, if you want to create a package for all supported distros, you could do something like this:

    $> for DISTRO in $(ls tools/docker); do
         ./tools/dev-env --distro ${DISTRO} -- ./tools/make-package --save-to build-${DISTRO}
       done

These can be composed with whatever sort of other *NIX tools through the power of the `|` operator (or `||`, `&&`, or
anything else).
The whole point of this to to make your environment feel like regular *NIX program that plays nicely with anything else
you use.
If you plan on using `dev-env` within a script, the `--no-build` and `--no-tty` flags will be helpful for you;
`--no-build` avoids repeated builds in the same script and `--no-tty` turns of the pseudo-TTY used for interactivity.
See the `--help` output for more flags and more information.

### Customizing Behavior

If you want custom, project-specific behavior, alter the script itself.
Just edit the text to do what you need it to do.

The `dev-env` script was designed with customization points in mind, so most functionality has neat little isolated code
blocks for modifications.
While the vertical space might look a little bit verbose, this is intentional to make it easier for your modifications
to trivially survive the 3-way merge used in [upgrades](#upgrades).

### Upgrades

Upgrading the `dev-env` script is easy, just run:

    ./tools/dev-env --upgrade-dev-env

This pulls the latest tagged `dev-env` version from GitHub and 3-way merges it using the script you originally copied as
a base.
If you are unfamiliar with [3-way merges][3-way-merge], I envy your ability to fast-forward merge all the time.
But basically there are three parts:

* **base**: The version of the script you originally copied, as set by `DEVENV_BASED_ON`
* **current**: The `dev-env` file as it currently is (potentially-modified from **base**)
* **upstream**: The version of the script you are updating to, which is either the highest-version Git tag or an
  explicit version from `--upgrade-dev-env-to`

We know that there are changes from **base** to **upstream** which were made in the repository you would like to have.
But there are also changes from **base** to **current** that you made which did not make it to the upstream repository.
A 3-way merge makes sure that both sets of changes make it into the result in some sane manner, asking you for help if
there are conflicting changes.

The `dev-env` script uses the same merge tool you would use with Git by asking `git config merge.tool`.
So if you have ever had a conflict in `git pull` or `git rebase`, the experience will be exactly as wonderful when
dealing with conflicts in the `dev-env` script.
There is only one file, so that is a distinct advantage.

Future Work
-----------

### Support for non-Docker

[Docker][Docker] is overall pretty functional, but it isn't perfect.
Other container runtimes like [Kata](https://katacontainers.io/) and
[Firecracker](https://firecracker-microvm.github.io/) have advantages and, even if they do not, they have users.
Adding support for full virtual environments with [Vagrant](https://www.vagrantup.com/) also seems helpful.

### Gold-plated Variants

If an alternative container runtime like [Kata](https://katacontainers.io/) is something a lot of people want, it would
be nice to have a script they can download instead of everyone patching it by hand.
It would be nice if [initial project setup](#initial-project-setup) is just downloading `variant/kata/dev-env`.
This same goal could be achieved by forking the repo and changing `DEVENV_REPO` to point at the fork, but having support
for this in the mainline would make it easy to keep all the variants in sync.

F.A.Q.
------

### Why not just use Docker directly?

There are a lot of flags you have to remember and nobody will ever consistently run them all.
If you look inside the `# Running` section of the script, there are a few steps you have to run every single time you
run the container.
I am an incredibly forgetful person, so if something isn't in a script, I will forget how to do it.
The `dev-env` script helps me remember through the power of file storage.

### Why is this not packaged as real software?

The problem with packaging software is that it is an external dependency on _something_.
I really dislike coming into a repo and having to download _Wonky Build Tool 12_ and _Weird Dependency X_.
Put that crap in [Docker][Docker].

The dependencies of this script are the standard set of tools you have in a regular old Linux distribution and also
Docker (which is almost ubiquitous).
So the "packaging" of this project is the boring text of a Bash script.
Users of your repository don't have to think about anything -- they just `./tools/dev-env` and get on with their lives.

The biggest drawback to using copy+paste as a distribution model are upgrades to the script.
However, the use of 3-way merges as mechanism for [upgrades](#upgrades) mitigates this problem.

### Isn't running random scripts from the internet insecure?

Yeah, but you're free to look at the content of the `dev-env` script.
It is just using a bunch of standard-fare Linux tools to accomplish its task.
Your `npm`/`pip`/`cargo`/`gradle`/etc call is a million times less secure than using this little script.
Isolating these external dependencies that you definitely don't do real security checks on to inside a Docker container
is probably a good practice.

### What about non-Linux platforms?

* **macOS**: Things should probably work just fine on macOS
* **BSD**: Docker is not well-supported on BSD variants, but there is no reason why the script would not work there
* **Windows**: Completely untested and is unlikely to be working

### Why is the branch called `trunk`?

When you clone a Git _tree_ and make a _branch_, isn't "trunk" the most logical name for the thing the branches grew out
of?
I will die on this hill.

[3-way-merge]: https://en.wikipedia.org/wiki/Merge_(version_control)#Three-way_merge
[Docker]:      https://www.docker.com/
