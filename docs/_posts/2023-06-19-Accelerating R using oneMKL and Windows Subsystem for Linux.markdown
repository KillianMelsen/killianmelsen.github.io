---
layout: post
title: "Accelerating R using oneMKL and Windows Subsystem for Linux"
date: 2023-06-20 17:15:32 +0200
categories: guide
---

The default BLAS and LAPACK libraries used by R are rather slow, meaning installing and switching to alternative libraries can result in a significant increase in performance. One such alternative is Intel's oneAPI Math Kernel Library (oneMKL) which is available as a standalone component or part of the [Intel oneAPI](https://www.intel.com/content/www/us/en/developer/tools/oneapi/overview.html) Base Toolkit. MKL was also part of Microsoft R Open, which has unfortunately been discontinued. As a result, installation of and using oneMKL on Windows does not seem to be straightforward at this time. As a workaround, Windows 10/11 users can use [Windows Subsystem for Linux (WSL)](https://ubuntu.com/wsl) running Ubuntu and install oneMKL there. Here I have combined information provided by [Carlos Santillan](https://csantill.github.io/RPerformanceWBLAS/) and [Dirk Eddelbuettel](http://dirk.eddelbuettel.com/blog/2018/04/15/) to easily set up WSL with an R installation using Intel's oneMKL BLAS and LAPACK libraries.



# Setting up Windows Subsystem for Linux

1.  Click start, and search for and Open **Command Prompt** as administrator.
2.  Run `wsl --install`.
3.  Reboot your device.
4.  After rebooting Ubuntu will start in a Command Prompt window. Choose a username and password when asked.
5.  Once Ubuntu has started the installed version can be checked by running `cat /etc/os-release` in the terminal.
6.  Update and upgrade all packages by running `sudo apt update` (enter your password when prompted) followed by `sudo apt upgrade -y`.



# Installing R and RStudio

1.  R can be installed by following the instructions at <https://cloud.r-project.org/bin/linux/ubuntu/> under **Install R** (running the lines in the terminal).

2.  Find the latest version of RStudio for the installed Ubuntu version at <https://posit.co/download/rstudio-desktop/>. Look for the download link ending in .DEB next to the correct Ubuntu version (22 for a default WSL installation, if in doubt check the version as described in step 5 in the previous section). Right-click the download link and click **Copy link address**.

3.  Download RStudio using `wget https://download1.rstudio.org/electron/jammy/amd64/rstudio-2023.06.0-421-amd64.deb` (using the copied link, the one shown here for illustrative purposes might be outdated).

4.  Install RStudio using `sudo apt install -f ./rstudio-2023.06.0-421-amd64.deb -y`. Note that the part between `-f ./` and `-y` depends on the version downloaded in step 3.

4.  Run `sudo apt update` followed by `sudo apt install libnss3-dev libgdk-pixbuf2.0-dev libgtk-3-dev libxss-dev -y` and finally `sudo apt install libasound2 -y`. In my case, RStudio would not run without these packages.

5.  The RStudio IDE GUI can now be launched by running `rstudio`. If you get any errors regarding additional missing packages I suggest looking up the error online and following any suggested instructions to install the missing packages.

5.  After opening up RStudio we can check what BLAS and LAPACK libraries we are using (the default ones at this point) by running `sessionInfo()` in the console: ![Default sessionInfo()](/assets/sessionInfo_default.png)

6.  Quit RStudio by running `q()` in the console or closing the window, then close the Ubuntu terminal. If you need to open the Ubuntu terminal again you can do this by simply running the **Ubuntu** application (look for it in the start menu) from the Windows desktop.

7.  On the Windows desktop, open the start menu, search for RStudio, and note that there is an RStudio (Ubuntu) application. An instance of RStudio running in Ubuntu can be launched by simply running this application while on the Windows desktop. No need to launch the Ubuntu terminal. Launch Ubuntu RStudio by running the application from the Windows desktop.

8.  After launching Ubuntu RStudio this way, run `parallel::detectCores()` in the console and note that the full number of threads is available to Ubuntu RStudio. The default BLAS and LAPACK libraries, however, are single-threaded and do not make full use of all threads. Close Ubuntu RStudio again.



# Installing Intel oneMKL

As mentioned, oneMKL can be installed as part of the oneAPI Base Toolkit. However, this toolkit is quite large at \~20 GB. We only need oneMKL and installing it as a standalone component (\~3 GB) saves a lot of space.

1.  Open up the Ubuntu terminal and run the following lines (you might have to press **Enter** and enter your password after running the first line):

        wget -O- https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB | gpg --dearmor | sudo tee /usr/share/keyrings/oneapi-archive-keyring.gpg > /dev/null

        echo "deb [signed-by=/usr/share/keyrings/oneapi-archive-keyring.gpg] https://apt.repos.intel.com/oneapi all main" | sudo tee /etc/apt/sources.list.d/oneAPI.list

        sudo apt update

2.  We can now look at what versions of oneMKL are available by running the following line:

        sudo -E apt-cache pkgnames intel | grep intel-oneapi-mkl | grep -v intel-oneapi-runtime

    The latest version at the time of writing is `intel-oneapi-mkl-2023.1.0`.

3.  To install this version we run:

        sudo apt install intel-oneapi-mkl-2023.1.0 -y

4.  Now run the following lines (note the oneMKL version numbers in these commands):

         sudo update-alternatives --install /usr/lib/x86_64-linux-gnu/libblas.so libblas.so-x86_64-linux-gnu /opt/intel/oneapi/mkl/2023.1.0/lib/intel64/libmkl_rt.so 50

         sudo update-alternatives --install /usr/lib/x86_64-linux-gnu/libblas.so.3 libblas.so.3-x86_64-linux-gnu /opt/intel/oneapi/mkl/2023.1.0/lib/intel64/libmkl_rt.so 50

         sudo update-alternatives --install /usr/lib/x86_64-linux-gnu/liblapack.so liblapack.so-x86_64-linux-gnu /opt/intel/oneapi/mkl/2023.1.0/lib/intel64/libmkl_rt.so 50

         sudo update-alternatives --install /usr/lib/x86_64-linux-gnu/liblapack.so.3 liblapack.so.3-x86_64-linux-gnu /opt/intel/oneapi/mkl/2023.1.0/lib/intel64/libmkl_rt.so 50

         echo "/opt/intel/oneapi/mkl/2023.1.0/lib/intel64" | sudo tee /etc/ld.so.conf.d/mkl.conf

         sudo ldconfig

         echo "MKL_THREADING_LAYER=GNU" | sudo tee /etc/environment -a

5.  If we now open Ubuntu RStudio (either by running `rstudio` in the terminal or by launch the RStudio (Ubuntu) application from the Windows desktop) and run `sessionInfo()` in the console we should see that we are now using the Intel oneMKL BLAS and LAPACK libraries: ![Library selection](/assets/libblas_choice.png)

6.  Switching back between the default and oneMKL libraries is possible using the following commands:

         # For the BLAS library:
         sudo update-alternatives --config libblas.so.3-x86_64-linux-gnu

         # For the LAPACK library:
         sudo update-alternatives --config liblapack.so.3-x86_64-linux-gnu

    Running these lines opens a menu where a number can be entered to select the desired library: ![oneMKL sessionInfo()](/assets/sessionInfo_MKL.png)

# Setting the number of threads

While the default BLAS and LAPACK libraries are single-threaded, oneMKL can make use of multiple threads. By default it uses as many threads as there are physical cores in the system. This can be problematic if a script or function makes use of explicit parallelization using, for example, the foreach package. In that case the number of threads to be used by oneMKL must be manually set to 1.

1.  In the Ubuntu terminal check if nano is installed by running `nano --version`. If it is not installed it can be installed by running `sudo apt install nano`.

2.  Run `sudo nano /etc/environment`.

3.  Add the following line to the bottom of the file: `MKL_NUM_THREADS=1`. The file should look like this: ![Setting threads](/assets/set_threads.png)

4.  Press **Ctrl + X** followed by the **Y** key to save the changes and press **Enter** to overwrite the file.

5.  Close and reopen the Ubuntu terminal. While oneMKL using 1 thread will not be *as* fast as allowing it to use multiple threads, it will still far outperform the default BLAS and LAPACK libraries, especially if single-threaded oneMKL is combined with efficient explicit parallelization in R scripts.

# Results

The figures below show the running times for a few operations measured using microbenchmark (n = 20) and the relative speed of the oneMKL libraries compared to the default BLAS/LAPACK libraries on Windows. ![Computational times for some operations](/assets/all_log.png) ![Relative speed](/assets/all_relative.png)
