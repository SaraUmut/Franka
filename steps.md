# How to set up your workstation to control the Franka Panda robot:

1. **Ubunut:** donwload Ubunut 16.04 (Xenial Xerus). (I tried on ubuntu 18.04 but it didn't work with me)
    1. you can find the download link for 64 bit here:
      http://releases.ubuntu.com/16.04/ubuntu-16.04.5-desktop-amd64.iso
    2. create bootable usb by using RUFUS: https://rufus.akeo.ie/
    3. boot from flash memory and install ubuntu.

2. **update & upgrade packages**:
    1. update: (write in terminal) `sudo apt-get update`
    2. upgrade: (terminal) `sudo apt-get upgrade`

3. **ROS:** Download ROS Kinetic:
    1. follow the steps in the official link: http://wiki.ros.org/kinetic/Installation/Ubuntu
       (use the Desktop-Full Install version).
    2. check the installation by typing (in terminal): `printenv | grep ROS`
        (you should get some information about the ROS path like: ROS_ROOT= /opt/ros/kinetic/share/ros)
        
4. **libfranka:** install libfranka which is the C++ implementation of the client side of the Franka Control Interface (FCI) [ref_1]. 
    1. Installing the dependencies: $ sudo apt install ros-kinetic-libfranka ros-kinetic-franka-ros
    2. Build the libfranka Package:
    ```
    sudo apt install build-essential cmake git libpoco-dev libeigen3-dev
    git clone --recursive https://github.com/frankaemika/libfranka
    cd libfranka
    mkdir build
    cd build
    cmake -DCMAKE_BUILD_TYPE=Release ..
    cmake --build .
    sudo make install
    ```
5. **franka-ros**: Build the franka-ros Package (required to control the robot using ROS)
```
(go to Home by typing in terminal) cd
mkdir -p catkin_ws/src
cd catkin_ws
source /opt/ros/kinetic/setup.sh
catkin_init_workspace src
git clone --recursive https://github.com/frankaemika/franka_ros src/franka_ros
rosdep install --from-paths src --ignore-src --rosdistro kinetic -y --skip-keys libfranka
catkin_make -DCMAKE_BUILD_TYPE=Release -DFranka_DIR:PATH=/path/to/libfranka/build
source devel/setup.sh
```

6. **real-time kernel**: The robot requires the controller program on the workstation PC to run with real-time priority (1 kHz).
    1. Donwnload Dependencies:
    `sudo apt install build-essential bc curl ca-certificates fakeroot gnupg2 libssl-dev lsb-release libelf-dev bison flex`
    
    2. Check the kernel version that you have:(terminal) `uname -r`
    3. Get the kernel patch version that is closest to the one you currently use. 
    (The following commands assume the 4.14.12 kernel version with the 4.14.12-rt10 patch, recommended)
    4. Download the source files using curl as follows:
    ```
    curl -SLO https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.14.12.tar.xz
    curl -SLO https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.14.12.tar.sign
    curl -SLO https://www.kernel.org/pub/linux/kernel/projects/rt/4.14/older/patch-4.14.12-rt10.patch.xz
    curl -SLO https://www.kernel.org/pub/linux/kernel/projects/rt/4.14/older/patch-4.14.12-rt10.patch.sign
    ```
    
    5. **Decompress** them by:
    ```
    xz -d linux-4.14.12.tar.xz
    xz -d patch-4.14.12-rt10.patch.xz
    ```
    
    6. **Verify** the file integrity (th verify that the downloaded files were not corrupted or tampered with)
    `gpg2 --verify linux-4.14.12.tar.sign`
        * If your output is similar to the following:
        ```
        $ gpg2 --verify linux-4.14.12.tar.sign
        gpg: assuming signed data in 'linux-4.14.12.tar'
        gpg: Signature made Fr 05 Jan 2018 06:49:11 PST using RSA key ID 6092693E
        gpg: Can't check signature: No public key
        ```
        * You have to first download the public key of the person who signed the above file. 
        As you can see from the above output, it has the ID 6092693E. You can obtain it from the key server:
        `gpg2  --keyserver hkp://keys.gnupg.net --recv-keys 0x6092693E`

        * Similarly for the patch:
        `gpg2 --keyserver hkp://keys.gnupg.net --recv-keys 0x2872E4CC`
        
        * Now verify the sources:
        ```
        gpg2 --verify linux-4.14.12.tar.sign
        gpg2 --verify patch-4.14.12-rt10.patch.sign
        ```
    7. **Compile** the kernel:
    ```
    tar xf linux-4.14.12.tar
    cd linux-4.14.12
    patch -p1 < ../patch-4.14.12-rt10.patch
    ```
    
    8. **Configure** the kernel:
    `make oldconfig`
    (When menu pops up choose option 5: Fully Preemptible Kernel. For the rest of the options, use default values (just press enter when you get a question))

    9. **Compile**:
    `fakeroot make deb-pkg`
    (if fakeroot doesn't work, use sudo make deb-pkg)
    
    10. `sudo dpkg -i ../linux-headers-4.14.12-rt10_*.deb ../linux-image-4.14.12-rt10_*.deb`
    
    11. **Verify the new kernel**
        1. Restart your system. 
        2. When you get the Grub boot menu choose: 'Advanced options for Ubuntu' 
        3. Choose from the list the new kernel: 'Ubuntu, with Linux 4.14.12.rt10 (generic)' or 'Ubuntu, with Linux 4.14.12.rt10'
        4. After booting, check that it is correct (terminal): `uname -a`
        5. Also check that the path: "/sys/kernel/realtime" exist and contain the number 1.

    12. **Permissions** you need to create then add the user controlling the robot to the group named realtime:
        1. write the following in the Terminal:
        ```
        sudo addgroup realtime
        sudo usermod -a -G realtime $(whoami)
        ```
    
        2. Open "limits.conf" file by doing:
        ```
        cd /etc/security/
        sudo gedit limits.conf
        ```
    
        3. Copy and paste the following lines to the end of the file:
        ```
        @realtime soft rtprio 99
        @realtime soft priority 99
        @realtime soft memlock 102400
        @realtime hard rtprio 99
        @realtime hard priority 99
        @realtime hard memlock 102400
        ```
    
        4. Finally save the file, close it, log out and log in again.
    
    **All Done, congratulations! :)**
    
    Now you can start controlling Franka robot, you can start by running the existing examples: 
    "How to run Franka examples"
    
    
**Useful commands**
------------------
* Check ubunut version (Terminal): $ lsb_release -a
* Check ROS version (Terminal): $ rosversion -d
* Check ROS version (Terminal): $ echo $ROS_DISTRO)

**References:**
---------------
* [ref_1] libfranka: https://frankaemika.github.io/docs/libfranka.html
* [ref_2] turorial: https://frankaemika.github.io/docs/installation_linux.html
* [ref_3] tutorial: https://hiro-group.ronc.one/franka_installation_tutorial.html
* [erf_4] Get Grub menu: https://askubuntu.com/questions/16042/how-to-get-to-the-grub-menu-at-boot-time
