---
layout: post
title:  "Accelerating R using oneMKL."
date:   2023-03-28 17:42:46 +0200
categories: guide
---

The default BLAS and LAPACK libraries used by R are rather slow, meaning installing and switching to alternative libraries can result in a significant increase in performance. One such alternative is Intel's oneAPI Math Kernal Library (oneMKL) which is available as a standalone component or part of the [Intel oneAPI](https://www.intel.com/content/www/us/en/developer/tools/oneapi/overview.html) Base Toolkit. MKL used to be part of Microsoft R Open, which has unfortunately been discontinued. As a result, installation of and using oneMKL on Windows does not seem to be straight forward at this time. As a workaround, Windows users can set up a simple virtual machine running Ubuntu and install oneMKL there. Here I have combined information provided by [Carlos Santillan](https://csantill.github.io/RPerformanceWBLAS/), [Dirk Eddelbuettel](http://dirk.eddelbuettel.com/blog/2018/04/15/) and [Anton Karl Ingason](http://linguist.is/2020/08/12/expand-ubuntu-disk-after-hyper-v-quick-create/) to easily set up an Ubuntu virtual machine with an R installation using Intel's oneMKL BLAS and LAPACK libraries. I have used Hyper-V, which requires Windows 10/11 Pro. A free alternative would be [VirtualBox]https://www.virtualbox.org/.

# Enabling Hyper-V (Windows 11 Pro)

1. Open **Settings**.
2. Open **Apps**.
3. Click on **Optional features**.
4. Open **More Windows features** at the bottom of the page.
5. Check the **Hyper-V** option.
6. Click on **OK** and restart the PC when prompted.

# Creating an Ubuntu virtual machine (VM) using Hyper-V Manager

1. Open **Hyper-V Manager**.
2. On the left, select your PC by clicking it.
3. Click **Quick Create** in the **Action** menu on the top toolbar.
4. Select the operating system (I have used Ubuntu 22.04 LTS) and optionally specify a custom name for the VM.
5. Click **Create Virtual Machine**.