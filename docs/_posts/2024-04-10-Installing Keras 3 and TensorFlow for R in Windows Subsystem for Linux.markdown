---
layout: post
title: "Installing Keras 3 and TensorFlow for R in Windows Subsystem for Linux"
date: 2024-04-09 17:15:32 +0200
categories: guide
---

APIs for [Keras](https://keras.posit.co/) 3 with the [TensorFlow](https://www.tensorflow.org/) backend can be installed in RStudio (Server) to enable easy access to deep learning from within RStudio. This guide assumes access to a WSL2 environment with R and RStudio Server installed.

# Prerequisites

1.  Open a WSL Ubuntu terminal and run `sudo apt install -y libcurl4-openssl-dev libfontconfig1-dev libharfbuzz-dev libfribidi-dev libfreetype6-dev libpng-dev libtiff5-dev libjpeg-dev libxml2-dev` to install a number of packages required for the devtools R package.
2.  Run `sudo rstudio-server start` and navigate to **localhost:8787** to start RStudio Server, followed by installing the devtools R package.
3.  Close RStudio Server by running `q()` in the console, closing the browser tab, and running `sudo rstudio-server stop` in the Ubuntu terminal. Exit the Ubuntu terminal and run `wsl --shutdown` in a Powershell window to stop WSL.
4.  Ensure that the latest Nvidia GPU driver is installed in Windows. Checking the current version can be done by opening Nvidia Control Panel and clicking System Information in the bottom left. The installed driver version should be visible in the window that opens.
5.  The latest drivers can be found at <https://www.nvidia.com/download/index.aspx?lang=en-us>. **Do not install any GPU driver in WSL itself!**
6.  Open an Ubuntu WSL terminal and run `nvidia-smi`. You should see your GPU listed as device 0 (assuming you have a single GPU), as well as the driver version. ![nvidia-smi](/assets/nvidia-smi.png)
7.  Running `nvcc --version` in the terminal should indicate that no CUDA toolkit is installed.

# Installing the CUDA toolkit and cuDNN

1. The information at <https://www.tensorflow.org/install/source#gpu> can be used to check which combinations of TensorFlow, Python, cuDNN, and CUDA are among the tested configurations. The most up-to-date config at the time of writing is TensorFlow 2.16.1, Python 3.9-3.12, cuDNN 8.9, and CUDA 12.3.

2. The instructions for installing CUDA 12.3 for WSL can be found at <https://developer.nvidia.com/cuda-12-3-2-download-archive?target_os=Linux&target_arch=x86_64&Distribution=WSL-Ubuntu&target_version=2.0&target_type=deb_network>:

    ```
    wget https://developer.download.nvidia.com/compute/cuda/repos/wsl-ubuntu/x86_64/cuda-keyring_1.1-1_all.deb
    sudo dpkg -i cuda-keyring_1.1-1_all.deb
    sudo apt-get update
    sudo apt-get -y install cuda-toolkit-12-3
    rm cuda-keyring_1.1-1_all.deb
    ```
    
    We run the `rm` command because we need a different .deb file with the same name to install cuDNN later.
    
3.  Run `sudo nano ~/.profile` and add the following lines to the file (note the CUDA version number):
    ```
    export PATH=/usr/local/cuda-12.3/bin:$PATH
    export LD_LIBRARY_PATH=/usr/local/cuda-12.3/lib64:$LD_LIBRARY_PATH
    ```
    
4.  Close the Ubuntu terminal using `exit` and run `wsl --shutdown` in a Powershell window to stop WSL.

5.  Reopen the Ubuntu terminal and run `nvcc --version` to check whether the CUDA toolkit was correctly installed. ![nvcc](/assets/nvcc.png)

6.  Now we'll install cuDNN by following the instructions at <https://docs.nvidia.com/deeplearning/cudnn/archives/cudnn-897/install-guide/index.html#package-manager-ubuntu-install>. Substitute `$distro/$arch` with `ubuntu2204/x86_64` when enabling the network repository, so run:
    ```
    wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
    sudo dpkg -i cuda-keyring_1.1-1_all.deb
    ```
    
    in the terminal, followed by:
    
    ```
    sudo apt update
    sudo apt install libcudnn8 libcudnn8-dev libcudnn8-samples
    rm cuda-keyring_1.1-1_all.deb
    ```

# Installing Python, Keras, and TensorFlow

1.  Start RStudio Server by running `sudo rstudio-server start` and navigating to **localhost:8787** in a browser window.

2.  Install the keras3 R package, followed by installing Python 3.10 using `reticulate::install_python(version = "3.10")`.

3.  Install Keras and the GPU capable tensorflow backend using `keras3::install_keras(envname = "r-tensorflow", gpu = TRUE)`.

4.  After the installation is finished the R session will be restarted. At this point the installed versions of TensorFlow and Python can be checked by running `tensorflow::tf_config()` in the console. At the time of writing this shows Python 3.10 and TensorFlow 2.16.1, which, when combined with the CUDA 12.3 and cuDNN 8.9 that we installed earlier, matches the tested configuration from <https://www.tensorflow.org/install/source#gpu>. ![tf_config](/assets/tf_config.png)

5.  Lastly, we can check whether the GPU was correctly configured by running `tensorflow::tf_gpu_configured()`. This should show a bunch of output, but most importantly it lists some of the GPU specifications at the bottom. ![tf_gpu_config](/assets/tf_gpu_config.png)
