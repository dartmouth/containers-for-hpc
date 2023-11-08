## Example

Here is the demo from the workshop with all the talking stripped out.  You can literally go through this in a couple minutes by copying and pasting the commands below into your terminal.  Replace `<NETID>` with your own NetID and `<JOBID>` with your job number.

1. If you don't already have an account on the Discovery cluster, request one at the [Research Computing Dashboard](https://dashboard.dartmouth.edu/research/hpc_account).

2. You need a way to run Docker on your personal computer.  We suggest installing [Rancher Desktop](https://rancherdesktop.io/) because it is free for any usage. [Docker Desktop](https://www.docker.com/products/docker-desktop/) is also an excellent choice with the caveat that it is not free for commercial purposes so you need read the licensing terms to be sure that you qualify.

3. Finally, you need to know how to open a terminal on your computer.  Installing Rancher Desktop will have configured the terminal so it has the docker commands available.  
   - Windows users can search the Start Menu for `Windows Subsystem for Linux ` and run it.  Then change to your Windows home directory with `cd /mnt/c/Users/<YOUR_WINDOWS_LOGIN_NAME> `.  
   - Mac users can search Finder for `/Applications/Utilities/Terminal` and run that.  Mac Terminal automatically puts you in your home directory.


Steps 4 - 8 are on run your personal computer, the rest are run on Discovery.

4. Create the Dockerfile using your favorite text editor.
   ```
   cat << 'EOF' > Dockerfile
   FROM python:3.12.0-alpine3.18
   COPY fib.py src/
   EOF
   ```
5. Create our program; i.e. the fib.py script.
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
6. Build the image from the Dockerfile
* For Intel computers
   ```
   docker build --file Dockerfile -t fibonacci .
   ```
* *For non-Intel Macs*
   
   ```
   docker buildx build --platform linux/amd64 --file Dockerfile -t fibonacci .
   ```

7. Save the image from to a single file.

   ```
   docker save fibonacci -o fibonacci.tar
   ```

8. Copy the file to discovery (might be different for Windows/Mac/Linux users)
   ```
   scp fibonacci.tar <NETID>@discovery:
   ```

9. Login to the Discovery head node (might be different for Windows/Mac/Linux users)
   ```
   ssh <NETID>@discovery
   ```

10. Convert the Docker image to an Apptainer image
   ```
   apptainer build fibonacci docker-archive://fibonacci.tar
   ```

11. Create an sbatch file - this uses a nifty shell feature available in Linux and Mac terminals (and Windows if you use the `wsl` terminal instead of PowerShell).
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
12. Submit a job 
    ```
    sbatch fib-sbatch.sh
    ```

13. Check the results (most recent output file if you have run the job more than once)
    ```
    ls -l fibonacci-*.out
    cat fibonacci-<JOBID>.out
    ```