# Containers for HPC

## Containers, why do they exist (efficiency)

- The earliest computers could only run one program at a time.
- Multi-tasking operating systems run multiple programs simultaneously on one computer - with caveats related to unshareable resources and to security.  This was a huge step forward for efficient use of hardware.
- VMs (virtual machines) run multiple "virtual" computers on one physical computer. Each VM has its own emulated hardware and its own operating system. VMs address the caveats for programs sharing a multi-tasking operating system, but have a lot of overhead which reduces efficiency.
- Containers provide programs with (most of) the isolation of VMs and (most of) the efficiency of a multi-tasking operating system.

## Containers in research (reproducibility)

That's great.  Where is the benefit to research workloads on Dartmouth’s Discovery cluster? 

Most programs are dependent on external libraries.  Those are traditionally provided by the operating system or by modules that Research Computing maintains.  The versions of those libraries WILL change as time goes by, which means that your program's behavior could change too. Containers package those libraries with your program so everything remains the same forever - or until you change something and make a new container image.

Moreover, containers are very portable. Once containerized, a program can easily be run on Discovery, or on an HPC cluster at another school, or even at cloud providers such as AWS or GCP.

So containers enable reproducibility across time and space.  Awesome!

## Definitions

* `container` - something actually running on the computer.  Processes inside the container cannot see anything else on the host computer, only what is explicitly placed inside the container.
* `image` - essentially the files and directories that go into a container when it starts to run.
* `Dockerfile` - the instructions for building an image (Docker specific).
* `container runtime` - runs a container from an image.  e.g. Docker and Apptainer are things that can actually run a container (and can also build images from an instruction file).

## Overview of running containers on Discovery

We'll go into more detail for each of these steps, but overviews are nice.

- create a Dockerfile - these are instructions for what needs to be in the container.
- build a Docker image from the Dockerfile - this results in an entire directory of stuff
- bundle the image directory into a single image file and upload to Discovery.
- convert the Docker image to an Apptainer image which is the kind of image Discovery supports.
- submit a job
   - to run your program
   - in a container
   - created from your Apptainer image.

## Creating a Docker file

A Docker file is literally a file, traditionally named Dockerfile, which contains instructions for creating the container image; i.e. which operating system it uses, what libraries are installed, the program to run, etc.  Creating your Dockerfile is by far the hardest part of containerizing a program.  

Pretty much every container image starts with a `base image` pulled down from a repository like [Docker Hub](https://hub.docker.com).  Docker Hub is a repository of freely available images built by the Docker community.  This discussion will make a lot more sense if you go there and poke around.  Sometimes you can find a base image which has everything you need. For example, there are images for Python, R, and NVIDIA/CUDA.  If this is you, that's great because the hard step just became a lot easier.  Other times you start with a base image and layer on everything else you need.

Docker Hub is not the only image repository.  Some software vendors have their own.  But pulling base images from Docker Hub is so common that it is the default when no repository is specified in the Dockerfile. For example, say you need a python environment.  Dockerhub has dozens of official base images under the name python.  The simplest possible Dockerfile to run Python is a one liner saying "start my image `FROM` the python image". 

```
FROM python
```

We haven't specified a repository of base images, so docker defaults to Dockerhub.  We haven't specified a version of Python, so it defaults to the latest supported version (3.12.0 as of this writing).  We haven't specified what operating system the base image should be, so it defaults to the latest version of whatever OS the builder of the image chose (Debian 12 as of this writing).

For reproducible research, we don't like seeing "as of this writing" or "whatever the builder chose".  Thankfully, you can use `tags` to be explicit about which base image you want. Looking at Docker Hub, the tag `3.12.0-alpine3.18` will explicitly request python 3.12.0 and version 3.18 of the alpine operating system.  alpine based images are almost 20x smaller than Debian (1GB vs 50MB) which is really nice for a simple demo!  So after adding the tag, our Dockerfile becomes

```
FROM python:3.12.0-alpine3.18
```

You could stop here, build a container image, and then run a container based on that image.  That might be useful during development.  e.g. You could get a shell inside a container running this image and do some interactive Python.  But where is our program?  As an example, suppose we want to calculate the Nth element in the Fibonacci sequence.  I've saved the following script in a file named `fib.py`.  To get the 10th number in the sequence `python fib.py 10`.

```
import sys

# Each number in the Fibonacci sequence is the sum of the preceding two numbers
# Traditionally, the first two numbers are 0 and 1
def fib(x):
    if x == 0: return 0
    if x == 1: return 1
    else: return(fib(x-1)+fib(x-2))

fib_num = int(sys.argv[1])
print(fib(fib_num))
```

Now we want to add this program to the image.  That's another line for our Dockerfile.  This `COPY` directive will copy a file from the docker working directory (whatever that means when you actually build the container) to a directory named `src` inside the container (this already exists in the base image we used).

```
FROM python:3.12.0-alpine3.18
COPY fib.py src/
```

That's it.  We have a Dockerfile.  The hard part is over.

Reproducibility Note: Saving your Dockerfiles is important if you ever want to rebuild the image.  Maybe you want to change versions of libraries, or build for a different architecture, or add some new capabilities.  These are tiny, so save them.

## Build the Docker image

This is a super simple command.  But we'll spend some time exploring here because once we've built an image we can finally run containers.

```
docker build --file Dockerfile -t fibonacci .
```

* `docker build` is the command to build images
* `--file Dockerfile` could actually be omitted because Dockerfile is the default name when you don't specify one.
* `-t fibonacci` is the name we are giving to the image.  This is also optional but it's much nicer to deal with names than image id numbers.
* `.` is how we tell docker to use the current working directory as its working directory (e.g. so COPY can find fib.py). 

The result will be a directory of files, i.e. the container image, in an obscure location on your computer.  We don't need to know where the image is for what we are doing today.  We'll just use docker tools to work with it.  

*Macs with Apple silicon (M1, M2, etc) only*

*Container images are very portable, but not across different CPU architectures.  So if you are building containers for Discovery (does not use Apple CPUs), on a non-Intel Mac, then you need to tell docker that.  The command to do so is*

*`docker buildx build --platform linux/amd64 --file Dockerfile -t fibonacci .`*

* *`docker buildx build` uses emulated hardware (very much like what VMs do) to build the image with different CPU support than the computer doing the build*
* *`--platform linux/amd64` is the CPU to build for*
* *the rest is the same*

We won't dive too deeply into all the things you can do with docker here, but knowing a few more commands is useful.

To list your docker images

```
$ docker image ls
REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
fibonacci    latest    1768a0e33c6d   3 seconds ago   51.8MB
```

How about running a container to calculate the 10th number in the Fibonacci sequence? *(The Intel/AMD CPU image we've built probably works on a non-Intel Mac but only because of the same kind of emulation that would be used to build it on a non-Intel Mac.)*

```
$ docker run --rm -it fibonacci /bin/sh
/ # ls src/
fib.py
/ # python src/fib.py 10
55
/ # exit
$
```

* `docker run` - command to run a container.
* `--rm`  - automatically stop the container when the command finishes.
* `-it` - actually two options which, together, let you go inside the container and interact with it from your terminal
* `fibonacci` - the container to run.
* `/bin/sh` - the command to run in the container.  Let's get a shell here so we can poke around.  You either need to specify a command here or have a `CMD` directive in your Dockerfile that does the same thing.  Otherwise the container won't run because there is nothing to run.

We do not have to go into the container to run our Fibonacci script.  Instead of the shell, `/bin/sh`, we can give `docker run` our python command.

```
$ docker run --rm -it fibonacci python src/fib.py 10
55
```

Very cool!

## Archive the image to a single file

Also a super simple step.

```
docker save fibonacci -o fibonacci.tar
```

* `docker save` is the command to save an image directory to a tar file.
* `fibonacci` is the name of the image we built.
* `-o fibonacci.tar` is the name of the tar file we want to create.

Reproducibility Note: Saving this tar file is the most surefire way to be able to reproduce your container at any time or place in the future - barring architectural differences like the Apple CPUs.  As previously mentioned, saving the Dockerfile is useful when you want to rebuild the image (e.g. with updated libraries or for a different architecture/platform).  We strongly suggest saving both the Dockerfile and the image file.

Copying the tar file to discovery is left as an exercise for the reader. 

## Convert image to Apptainer format

This is also super simple to do, but what is Apptainer?  Apptainer (formerly Singularity) is a competing container platform.  It has some extremely significant benefits in the HPC world, but Docker (and the availability of Dockerhub) are easier to work with.  So while it is possible to create container images natively in Apptainer, we recommend doing everything in Docker and then converting the image to Apptainer as the last step.

We need to be logged into discovery here.

```
apptainer build fibonacci docker-archive://fibonacci.tar
```

* `apptainer build` - apptainer's version of docker build, but used to build apptainer images.
* `fibonacci` - the name we give the image.  This was good enough for when it was a docker image, why change now.
* `docker-archive://fibonacci.tar` - says to build from the file fibonacci.tar, which we previously copied to discovery, with a prefix that tells apptainer what format the file is in.

Let's test the Apptainer version of the image.

```
apptainer exec fibonacci python /src/fib.py 10
55
```

* `apptainer exec` - command to run a container; like `docker run`.
* `fibonacci` - is the name of the image we gave when building it
* `python /src/fib.py 10` - the command to run inside the container

## Run it on the cluster

We aren't supposed to run computational jobs on the head node like we just did.  But it was small and quick - hopefully nobody noticed.  :-) What we want to do is submit a job to the scheduler and let it find a place to run our program.

First, we need a minimal sbatch script.  I've saved this one as `fib-sbatch.sh`.  This requests 1 CPU on 1 node with 100MB of RAM and a walltime of 1 minute.  Our output will be stored in a file named `fibonacci-<jobid>.out`.  By now, the final line should be very recognizable as the program we've been running.  

```
#!/bin/bash
#SBATCH --job-name=fibonacci
#SBATCH --output=%x-%J.out
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=1
#SBATCH --mem=100M
#SBATCH --time=00:01:00
apptainer exec fibonacci python -u /src/fib.py 10
```

Submit it to the scheduler.

```
sbatch fib-sbatch.sh
```

Check progress

```
$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
           8368448  standard fibonacc    NETID  R       0:00      1 n07
```

When it is finished look at our output.

```
$ cat fibonacci-8368448.out
55
```

Time to publish our results!

## Going through the example yourself

Here is the process with all the talking stripped out.  You can literally go through this in a couple minutes if you copy and paste the commands (replace `<NETID>` with your own NetID of course).

1. If you don't already have an account on the Discovery cluster, request one at the [Research Computing Dashboard](https://dashboard.dartmouth.edu/research/hpc_account).

2. You need a way to run Docker on your personal computer.  We suggest installing [Rancher Desktop](https://rancherdesktop.io/) because it is free for any usage. [Docker Desktop](https://www.docker.com/products/docker-desktop/) is also an excellent choice with the caveat that it is not free for commercial purposes so you should read the licensing terms to make sure you qualify.

Steps 3 - 7 are on your personal computer, the rest are run on Discovery.

3. Create the Dockerfile - this uses a nifty shell trick so you can just paste this into your terminal and hit enter to create the file.
   ```
   cat << 'EOF' > Dockerfile
   FROM python:3.12.0-alpine3.18
   COPY fib.py src/
   EOF
   ```
4. Create the fib.py script - same shell trick
   ```
   cat << 'EOF' > fib.py
   import sys
   
   # Each number in the Fibonacci sequence is the sum of the preceding two numbers
   # Traditionally, the first two numbers are 0 and 1
   def fib(x):
      if x == 0: return 0
      if x == 1: return 1
      else: return(fib(x-1)+fib(x-2))
   
   fib_num = int(sys.argv[1])
   print(fib(fib_num))
   EOF
   ```
5. Build the image from the Dockerfile
* For Intel computers
   ```
   docker build --file Dockerfile -t fibonacci .
   ```
* *For non-Intel Macs*
   ```
   docker buildx build --platform linux/amd64 --file Dockerfile -t fibonacci .
   ```

6. Save the image from to a single file.

   ```
   docker save fibonacci -o fibonacci.tar
   ```

7. Copy the file to discovery
   ```
   scp fibonacci.tar <NETID>@discovery:
   ```

8. Login to the Discovery head node
   ```
   ssh <NETID>@discovery
   ```

9. Convert the Docker image to an Apptainer image
   ```
   apptainer build fibonacci docker-archive://fibonacci.tar
   ```

10. Create an sbatch file
    ```
    cat << 'EOF' > fib-sbatch.sh
    #!/bin/bash
    #SBATCH --job-name=fibonacci
    #SBATCH --output=%x-%J.out
    #SBATCH --nodes=1
    #SBATCH --ntasks=1
    #SBATCH --cpus-per-task=1
    #SBATCH --mem=100M
    #SBATCH --time=00:01:00
    apptainer exec fibonacci python -u /src/fib.py 10
    EOF
    ```
11. Submit a job 
    ```
    sbatch fib-sbatch.sh
    ```

12. Check the results (most recent output file if you have run the job more than once)
    ```
    ls -l fibonacci-*.out
    cat fibonacci-<JOBID>.out
    ```

## Technical Details

The classic analogy is that a physical host (or a VM) with a multi-tasking OS is a single family mansion while containers are apartments in a similarly sized building.  Programs on a physical host share everything; utilities, furniture, furnishings, decorations, etc.  Programs in a container only share a few utilities like electrical, water, and septic services; everything else they bring with them to furnish the apartment.  

So how does that really work? What are the utilities that containers share?  The key to restricting what containers can see and use is kernel namespaces.

Namespaces partition kernel resources. A process in a namespace is restricted to seeing only the resources in its namespace.  Containers typically have 7 or 8 namespaces for partitioning different types of resources.  They are

* PID - processes have their own set of PIDs that are independent of those in the rest of the OS
* net - processes have their own IP address and ports
* uts - processes have their own hostname - unfortunately named for historical reasons
* user - processes have their own set of UIDs and GIDs
* mnt - processes are chroot'd to their own root directory and have their own filesystem mounts
* ipc - processes have their own interprocess communication space
* cgroup - processes can be forced to see and use only a subset of the CPU and RAM available (typically, though control groups can limit things like I/O bandwidth too)
* time - processes in a container can have their own time (Mars standard time?)

That provides all the isolation most programs need,though it is important to remember that the cgroup resources are ultimately still constrained by what's available on the physical host.  You can't fill an apartment with crypto mining hardware without tripping breakers.  And you can't fill a host with cryptomining containers without crashing it.

In terms of efficiency, by not needing to be "general purpose" computers, containers can be really focused with their furnishings.  e.g. A "minimal" Ubuntu installation (like you might put in a VM) is usually several gigabytes while a container based on alpine starts at about 8MB.  The problems with programs not "playing nice" with each other in a shared environment are very real, which was why VMs were invented, but the overhead of a VM is so much greater than that of containers.

## Links to more information
ToDo
