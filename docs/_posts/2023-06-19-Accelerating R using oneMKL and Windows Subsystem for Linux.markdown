---
layout: post
title: "Accelerating R using oneMKL and Windows Subsystem for Linux"
date: 2023-06-20 17:15:32 +0200
categories: guide
---

The default BLAS and LAPACK libraries used by R are rather slow, meaning installing and switching to alternative libraries can result in a significant increase in performance. One such alternative is Intel's oneAPI Math Kernel Library (oneMKL) which is available as a standalone component or part of the [Intel oneAPI](https://www.intel.com/content/www/us/en/developer/tools/oneapi/overview.html) Base Toolkit. MKL was also part of Microsoft R Open, which has unfortunately been discontinued. As a result, installation of and using oneMKL on Windows does not seem to be straightforward at this time. As a workaround, Windows 10/11 users can use [Windows Subsystem for Linux (WSL)](https://ubuntu.com/wsl) running Ubuntu and install oneMKL there. Here I have combined information provided by [Carlos Santillan](https://csantill.github.io/RPerformanceWBLAS/) and [Dirk Eddelbuettel](http://dirk.eddelbuettel.com/blog/2018/04/15/) to easily set up WSL with an R installation using Intel's oneMKL BLAS and LAPACK libraries.

# Setting up Windows Subsystem for Linux

1.  Click start, and search for and open **Command Prompt** as administrator.
2.  Run `wsl --install`.
3.  Reboot your device.
4.  After rebooting Ubuntu will start in a Command Prompt window. Choose a username and password when asked.
5.  Once Ubuntu has started the installed version can be checked by running `cat /etc/os-release` in the terminal.
6.  Update and upgrade all packages by running `sudo apt update` (enter your password when prompted) followed by `sudo apt upgrade -y` and `sudo apt install build-essential -y`.

# Installing R and RStudio

1.  R can be installed by following the instructions at <https://cloud.r-project.org/bin/linux/ubuntu/> under **Install R** (running the lines in the terminal).

2.  Follow the instructions at <https://posit.co/download/rstudio-server/> to install RStudio Server (starting at step 3 because we already istalled R itself). At the top of the page and under step 1, select Debian/Ubuntu for the operating system and Debian 12/Ubuntu 22 for the server version. RStudio Server should be started automatically after the last step.

3.  On your Windows desktop, open a browser and navigate to `localhost:8787`.

4.  Log in using the username and password chosen in step 4 of the previous section.

5.  After opening up RStudio Server and logging in we can check what BLAS and LAPACK libraries we are using (the default ones at this point) by running `sessionInfo()` in the console: ![Default sessionInfo()](/assets/sessionInfo_default.png)

6.  We can run `parallel::detectCores()` in the console and see that the full number of threads is available to Ubuntu RStudio Server. The default BLAS and LAPACK libraries, however, are single-threaded and do not make full use of all threads.

7.  Quit RStudio Server by running `q()` in the console and close the browser tab.

8.  In the Ubuntu terminal, run `sudo rstudio-server stop` to stop the RStudio server (it can be started again later by running `sudo rstudio-server start`).

# Installing Intel oneMKL

As mentioned, oneMKL can be installed as part of the oneAPI Base Toolkit. However, this toolkit is quite large at \~20 GB. We only need oneMKL and installing it as a standalone component (\~3 GB) saves a lot of space.

1.  In the Ubuntu terminal, run the following lines (you might have to press **Enter** and enter your password after running the first line):

    ```         
    wget -O- https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB | gpg --dearmor | sudo tee /usr/share/keyrings/oneapi-archive-keyring.gpg > /dev/null

    echo "deb [signed-by=/usr/share/keyrings/oneapi-archive-keyring.gpg] https://apt.repos.intel.com/oneapi all main" | sudo tee /etc/apt/sources.list.d/oneAPI.list

    sudo apt update
    ```

2.  We can now look at what versions of oneMKL are available by running the following line:

    ```         
    sudo -E apt-cache pkgnames intel | grep intel-oneapi-mkl-20
    ```

    The latest version at the time of writing is `intel-oneapi-mkl-2024.1`.

3.  To install this version we run:

    ```         
    sudo apt install intel-oneapi-mkl-2024.1 -y
    ```

4.  Now run the following lines (note the oneMKL version numbers in these commands):

    ```         
     sudo update-alternatives --install /usr/lib/x86_64-linux-gnu/libblas.so libblas.so-x86_64-linux-gnu /opt/intel/oneapi/mkl/2024.1/lib/libmkl_rt.so 50

     sudo update-alternatives --install /usr/lib/x86_64-linux-gnu/libblas.so.3 libblas.so.3-x86_64-linux-gnu /opt/intel/oneapi/mkl/2024.1/lib/libmkl_rt.so 50

     sudo update-alternatives --install /usr/lib/x86_64-linux-gnu/liblapack.so liblapack.so-x86_64-linux-gnu /opt/intel/oneapi/mkl/2024.1/lib/libmkl_rt.so 50

     sudo update-alternatives --install /usr/lib/x86_64-linux-gnu/liblapack.so.3 liblapack.so.3-x86_64-linux-gnu /opt/intel/oneapi/mkl/2024.1/lib/libmkl_rt.so 50

     echo "/opt/intel/oneapi/mkl/2024.1/lib" | sudo tee /etc/ld.so.conf.d/mkl.conf

     sudo ldconfig

     echo "MKL_THREADING_LAYER=GNU" | sudo tee /etc/environment -a
    ```

5.  If we now start Ubuntu RStudio Server (by running `sudo rstudio-server start` and going to `localhost:8787` in a browser) and run `sessionInfo()` in the console we should see that we are now using the Intel oneMKL BLAS and LAPACK libraries: ![Library selection](/assets/sessionInfo_MKL.png)

6.  Switching back between the default and oneMKL libraries is possible using the following commands:

    ```         
     # For the BLAS library:
     sudo update-alternatives --config libblas.so.3-x86_64-linux-gnu

     # For the LAPACK library:
     sudo update-alternatives --config liblapack.so.3-x86_64-linux-gnu
    ```

    Running these lines opens a menu where a number can be entered to select the desired library: ![oneMKL sessionInfo()](/assets/libblas_choice.png)

# Setting the number of threads

While the default BLAS and LAPACK libraries are single-threaded, oneMKL can make use of multiple threads. By default it seems to be able to use as many threads as there are logical cores in the system, but the actual number of threads that is used is determined dynamically. Multithreading can be problematic if a script or function makes use of explicit parallelization using, for example, the foreach package. In that case the maximum number of threads to be used by oneMKL must be manually set to 1 (or another number so that the number of MKL threads times the number of parallel R processes does not become too large).

1.  In the Ubuntu terminal check if nano is installed by running `nano --version`. If it is not installed it can be installed by running `sudo apt install nano`.

2.  Run `sudo nano /usr/lib/R/etc/Renviron.site`.

3.  Add the following line to the bottom of the file: `MKL_NUM_THREADS=1`. The file should look something like this: ![Setting threads](/assets/Renviron.png)

4.  Press **Ctrl + X** followed by the **Y** key to save the changes and press **Enter** to overwrite the file.

5.  While oneMKL using 1 thread will not be *as* fast as allowing it to use multiple threads, it will still far outperform the default BLAS and LAPACK libraries, especially if single-threaded oneMKL is combined with efficient explicit parallelization in R scripts.

6.  The dynamic aspect of how many threads MKL actually uses can be disabled by similarly adding `MKL_DYNAMIC=FALSE` to `/usr/lib/R/etc/Renviron.site`.

# Results

The figures below show the running times for a few operations measured using microbenchmark (n = 20, 10 threads on an Intel Core i5-13600KF) and the relative speed of the oneMKL libraries compared to the default BLAS/LAPACK libraries on Windows. ![Computational times for some operations](/assets/all_log.png) ![Relative speed](/assets/all_relative.png)
