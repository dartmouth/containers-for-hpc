# Example

Here is the demo from the workshop with the talking stripped out.  Replace `<NETID>` with your own NetID and `<JOBID>` with your job number.

## Prerequisites

* If you don't already have an account on the Discovery cluster, request one at the [Research Computing Dashboard](https://dashboard.dartmouth.edu/research/hpc_account).

* You need a way to run Docker on your personal computer.  We suggest installing [Rancher Desktop](https://rancherdesktop.io/) because it is free for any usage and available on Mac, Windows, and Linux.  [Docker Desktop](https://www.docker.com/products/docker-desktop/) is also an excellent choice with the caveat that it is not free for commercial purposes so you need to read the licensing terms to be sure that it's appropriate for you to use the free version.

* Finally, you need to know how to open a terminal on your computer.  Installing Rancher Desktop will have configured the terminal so it has the docker commands available.
   - Windows users can search the Start Menu for `Powershell ` and run it.
   - Mac users can search Finder for `/Applications/Utilities/Terminal` and run that.

## Building and Running

1. Create the `Dockerfile` file using your favorite text editor.  Be cognizant of the name you get.  Some text editors will silently append `.txt` resulting in `Dockerfile.txt` (e.g. Windows notepad). You will need to make sure you specify the correct filename in step 3 where you build the image.
   ```
   FROM python:3.12.0-alpine3.18
   COPY fib.py src/
   ```

2. Create the `fib.py` file using your favorite text editor.

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

3. Build the image from the Dockerfile
   * For Intel computers
   ```
   docker build --file Dockerfile -t fibonacci .
   ```
   * For non-Intel Macs

   ```
   docker buildx build --platform linux/amd64 --file Dockerfile -t fibonacci .
   ```

4. Save the image to a single file.

   ```
   docker save fibonacci -o fibonacci.tar
   ```

5. Copy the file to discovery
   ```
   scp fibonacci.tar <NETID>@discovery.dartmouth.edu:
   ```

6. Login to the Discovery head node
   ```
   ssh <NETID>@discovery.dartmouth.edu
   ```

7. Convert the Docker image to an Apptainer image
   ```
   apptainer build fibonacci docker-archive://fibonacci.tar
   ```

8. Create an sbatch file - this uses a nifty Linux shell trick to create the file without having to open an editor.  Just cut and paste into the Discovery terminal.
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

9. Submit a job 
   ```
   sbatch fib-sbatch.sh
   ```

10. Check the results. You will have multiple output files if you have run the job more than once, so use the most recent one with the job id that matches the job you just submitted.
    ```
    ls -l fibonacci-*.out
    cat fibonacci-<JOBID>.out
    ```
