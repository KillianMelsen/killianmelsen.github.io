---
layout: post
title:  "Accelerating R using oneMKL."
date:   2023-03-28 17:42:46 +0200
categories: guide
---

The default BLAS and LAPACK libraries used by R are rather slow, meaning installing and switching to alternative libraries can result in a significant increase in performance. One such alternative is Intel's oneAPI Math Kernal Library (oneMKL) which is available as a standalone component or part of the [Intel oneAPI](https://www.intel.com/content/www/us/en/developer/tools/oneapi/overview.html) Base Toolkit. MKL was also part of Microsoft R Open, which has unfortunately been discontinued. As a result, installation of and using oneMKL on Windows does not seem to be straight forward at this time. As a workaround, Windows users can set up a simple virtual machine running Ubuntu and install oneMKL there. Here I have combined information provided by [Carlos Santillan](https://csantill.github.io/RPerformanceWBLAS/), [Dirk Eddelbuettel](http://dirk.eddelbuettel.com/blog/2018/04/15/) and [Anton Karl Ingason](http://linguist.is/2020/08/12/expand-ubuntu-disk-after-hyper-v-quick-create/) to easily set up an Ubuntu virtual machine with an R installation using Intel's oneMKL BLAS and LAPACK libraries. I have used Hyper-V, which requires Windows 10/11 Pro. A free alternative would be [VirtualBox](https://www.virtualbox.org/).

# Enabling Hyper-V (Windows 11 Pro)

1.  Open **Settings**.
2.  Open **Apps**.
3.  Click on **Optional features**.
4.  Open **More Windows features** at the bottom of the page.
5.  Check the **Hyper-V** option.
6.  Click on **OK** and restart the PC when prompted.

# Creating an Ubuntu virtual machine (VM) using Hyper-V Manager

1.  Open **Hyper-V Manager**.
2.  On the left, select your PC by clicking it.
3.  Click **Quick Create** in the **Action** menu on the top toolbar.
4.  Select the operating system (I have used Ubuntu 22.04 LTS) and optionally specify a custom name for the VM.
5.  Click **Create Virtual Machine**.
6.  Once done, click **Connect**, followed by **Start**.
7.  Go through the system configuration. Select your language, keyboard layout, and location.
8.  Choose a username and password.
9.  Once prompted, select a display configuration/resolution and click **Connect**.
10. Log in using your previously created credentials.
11. Skip through the menus by clicking **Next** and finally **Done**.
12. After a minute or so, the Software Updater will pop up with the option to update software. Click **Install Now**.
13. Enter your password and click **Authenticate**.
14. Click **Restart Now** when prompted, enter your password again, and click **Authenticate**.
15. Close the small Virtual Machine Connection window or click **Exit** if prompted.
16. In Hyper-V Manager, right-click your VM and click **Turn Off...**.

# Expanding VM storage space

1.  In Hyper-V Manager, select your VM by left-clicking it and check if there are any checkpoints.
2.  Right-click a checkpoint tree if any is present, and click **Delete Checkpoint Subtree...**, followed by **Delete**. If there are no checkpoints this step can be skipped.
3.  Select your VM again, right-click and open **Settings...**.
4.  Select the Hard Drive by left-clicking it and click **Edit** in the right-hand screen.
5.  Click **Next \>**, check **Expand**, and click **Next \>** again.
6.  Enter the new (maximum) size for the virtual hard disk. I would use 25 GB as a minimum, with (much) larger sizes being a better choice if you expect to work with (much) larger datasets.
7.  Click **Next \>** to review the changes. Note that we are using a dynamically expanding virtual hard disk. The size we specified in the last step is the maximum, but it will only use as much space as required.
8.  Click **Finish**, followed by **OK** in the settings window.
9.  Right-click your VM and click **Start**.
10. Right-click it again and click **Connect...**
11. After logging in, open up the terminal by pressing Ctrl + Alt + T.
12. Run `sudo apt install cloud-guest-utils` and enter your password when prompted.
13. Run `sudo growpart /dev/sda 1` followed by `sudo resize2fs /dev/sda1`.

# Installing R and RStudio

1.  R can be installed by following the instructions at <https://cloud.r-project.org/bin/linux/ubuntu/> under **Install R** (running the lines in the terminal).
2.  RStudio can be installed by simply downloading the Ubuntu 22 installer from <https://posit.co/download/rstudio-desktop/> and opening the file with Software Install (right-click the file, click **Open With Other Application**, and double-click **Software Install**).
3.  Open up the terminal again and run `sudo apt-get install build-essential`.
4.  We can now open up RStudio and see what BLAS and LAPACK libraries we are using by running `sessionInfo()` in the console:

    <pre>> sessionInfo()
    R version 4.2.3 (2023-03-15)
    Platform: x86_64-pc-linux-gnu (64-bit)
    Running under: Ubuntu 22.04.2 LTS
    
    Matrix products: default
    <b>BLAS:   /usr/lib/x86_64-linux-gnu/blas/libblas.so.3.10.0</b>
    <b>LAPACK: /usr/lib/x86_64-linux-gnu/lapack/liblapack.so.3.10.0</b>
    
    locale:
     [1] LC_CTYPE=en_US.UTF-8       LC_NUMERIC=C               LC_TIME=nl_NL.UTF-8       
     [4] LC_COLLATE=en_US.UTF-8     LC_MONETARY=nl_NL.UTF-8    LC_MESSAGES=en_US.UTF-8   
     [7] LC_PAPER=nl_NL.UTF-8       LC_NAME=C                  LC_ADDRESS=C              
    [10] LC_TELEPHONE=C             LC_MEASUREMENT=nl_NL.UTF-8 LC_IDENTIFICATION=C       
    
    attached base packages:
    [1] stats     graphics  grDevices utils     datasets  methods   base     
    
    loaded via a namespace (and not attached):
    [1] compiler_4.2.3 tools_4.2.3   
    ></pre>

# Installing Intel oneMKL

As mentioned, oneMKL can be installed as part of the oneAPI Base Toolkit. However, this toolkit is quite large at ~20 GB. We only need oneMKL and installing it as a standalone component (~3 GB) saves a lot of space.

1.  Open up the terminal in your VM and run the following lines (you might have to press Enter and enter your password after running the first line):

        wget -O- https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB | gpg --dearmor | sudo tee /usr/share/keyrings/oneapi-archive-keyring.gpg > /dev/null
        
        echo "deb [signed-by=/usr/share/keyrings/oneapi-archive-keyring.gpg] https://apt.repos.intel.com/oneapi all main" | sudo tee /etc/apt/sources.list.d/oneAPI.list
        
        sudo apt update

2.  We can now look at what versions of oneMKL are available by running the following line:

        sudo -E apt-cache pkgnames intel | grep intel-oneapi-mkl | grep -v intel-oneapi-runtime
  
    The latest version at the time of writing is `intel-oneapi-mkl-2023.0.0`.
  
3.  To install this version we run:

        sudo apt install intel-oneapi-mkl-2023.0.0

4. Now run the following lines:

        sudo update-alternatives --install /usr/lib/x86_64-linux-gnu/libblas.so libblas.so-x86_64-linux-gnu /opt/intel/oneapi/mkl/2023.0.0/lib/intel64/libmkl_rt.so 50
        
        sudo update-alternatives --install /usr/lib/x86_64-linux-gnu/libblas.so.3 libblas.so.3-x86_64-linux-gnu /opt/intel/oneapi/mkl/2023.0.0/lib/intel64/libmkl_rt.so 50
        
        sudo update-alternatives --install /usr/lib/x86_64-linux-gnu/liblapack.so liblapack.so-x86_64-linux-gnu /opt/intel/oneapi/mkl/2023.0.0/lib/intel64/libmkl_rt.so 50
        
        sudo update-alternatives --install /usr/lib/x86_64-linux-gnu/liblapack.so.3 liblapack.so.3-x86_64-linux-gnu /opt/intel/oneapi/mkl/2023.0.0/lib/intel64/libmkl_rt.so 50

        echo "/opt/intel/oneapi/mkl/2023.0.0/lib/intel64" | sudo tee /etc/ld.so.conf.d/mkl.conf
        
        sudo ldconfig
        
        echo "MKL_THREADING_LAYER=GNU" | sudo tee /etc/environment -a
        
5. If we now open RStudio and run `sessionInfo()` in the console we should see that we are using the Intel oneMKL BLAS and LAPACK libraries:

    <pre>> sessionInfo()
    R version 4.2.3 (2023-03-15)
    Platform: x86_64-pc-linux-gnu (64-bit)
    Running under: Ubuntu 22.04.2 LTS
    
    Matrix products: default
    <b>BLAS/LAPACK: /opt/intel/oneapi/mkl/2023.0.0/lib/intel64/libmkl_rt.so.2</b>
    
    locale:
     [1] LC_CTYPE=en_US.UTF-8       LC_NUMERIC=C               LC_TIME=nl_NL.UTF-8       
     [4] LC_COLLATE=en_US.UTF-8     LC_MONETARY=nl_NL.UTF-8    LC_MESSAGES=en_US.UTF-8   
     [7] LC_PAPER=nl_NL.UTF-8       LC_NAME=C                  LC_ADDRESS=C              
    [10] LC_TELEPHONE=C             LC_MEASUREMENT=nl_NL.UTF-8 LC_IDENTIFICATION=C       
    
    attached base packages:
    [1] stats     graphics  grDevices utils     datasets  methods   base     
    
    loaded via a namespace (and not attached):
    [1] compiler_4.2.3 tools_4.2.3   
    ></pre>

6. Switching back between the default and oneMKL libraries is possible using the following commands:

        # For the BLAS library:
        sudo update-alternatives --config libblas.so.3-x86_64-linux-gnu
        
        # For the LAPACK library:
        sudo update-alternatives --config liblapack.so.3-x86_64-linux-gnu
        
    Running these lines opens a menu where a number can be entered to select the desired library:

        There are 2 choices for the alternative libblas.so.3-x86_64-linux-gnu (providing /usr/lib/x86_64-linux-gnu/libblas.so.3).

          Selection    Path                                                     Priority   Status
        ------------------------------------------------------------
          0            /opt/intel/oneapi/mkl/2023.0.0/lib/intel64/libmkl_rt.so   50        auto mode
        * 1            /opt/intel/oneapi/mkl/2023.0.0/lib/intel64/libmkl_rt.so   50        manual mode
          2            /usr/lib/x86_64-linux-gnu/blas/libblas.so.3               10        manual mode
        
        Press <enter> to keep the current choice[*], or type selection number:




















