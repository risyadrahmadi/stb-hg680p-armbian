# STB HG680P — Armbian & RTL8189FS WiFi

This repository contains the technical documentation for transforming an **HG680P Android TV Box (STB)** into a Linux-based mini server using **Armbian**.

The project covers the complete initial setup, including installing Armbian to an SSD, configuring the operating system, enabling SSH access, identifying the internal WiFi chipset, compiling the required **RTL8189FS (`8189fs`) driver**, and connecting the STB to a WiFi network.

The implementation is based on an HG680P using an Amlogic S905X platform.

---

## Project Overview

The HG680P is originally designed as an Android TV box. With Armbian, the device can be repurposed as a low-power Linux mini server.

The initial setup in this project includes:

* Installing Armbian Server
* Booting Armbian from an SSD
* Configuring the Linux system
* Configuring the initial user
* Configuring timezone and shell
* Connecting the device through Ethernet
* Enabling SSH access
* Identifying the internal WiFi chipset
* Installing the required build dependencies
* Compiling the RTL8189FS driver
* Loading the `8189fs` kernel module
* Verifying the WiFi interface
* Connecting to a wireless network

This setup becomes the foundation for the next stages of the mini-server project, such as Docker, CasaOS, Portainer, monitoring, and application deployment.

---

## Hardware

The hardware used in this project includes:

* HG680P Android TV Box
* Amlogic S905X ARM64 platform
* SATA SSD
* SATA-to-USB enclosure
* Ethernet cable
* HDMI capture device
* Laptop/PC for installation and monitoring
* STB power adapter

The SSD is used as the main storage device for the Armbian installation.

---

## Software Environment

The system uses:

* Armbian Server
* Ubuntu Noble userspace
* Linux 6.12.x
* Bash
* Git
* GCC
* GNU Make
* DKMS
* NetworkManager
* SSH

The Armbian image used for the project can be obtained from the Armbian Amlogic release repository:

https://github.com/ophub/amlogic-s9xxx-armbian/releases

---

# 1. Flash Armbian to the SSD

The first step is preparing the SSD with the Armbian image.

The SSD is connected to the computer through a SATA-to-USB enclosure.

A tool such as **balenaEtcher** can be used to flash the Armbian image to the SSD.

Before flashing, verify the target storage device carefully. Selecting the wrong disk can overwrite existing data.

The general process is:

1. Download the Armbian image.
2. Connect the SSD to the computer.
3. Open balenaEtcher.
4. Select the Armbian image.
5. Select the correct SSD.
6. Start the flashing process.
7. Wait until verification is completed.
8. Safely disconnect the SSD.

The SSD is then connected to the HG680P.

---

# 2. Boot the HG680P

After the SSD has been prepared, connect the required hardware:

* SSD
* Ethernet cable
* HDMI capture device
* Laptop/PC
* STB power adapter

The HDMI capture device can be used to monitor the boot process from a laptop.

For example, **OBS Studio** can be used to display the HDMI capture input.

When the Armbian boot process appears, the STB has successfully started Linux from the prepared storage.

---

# 3. Initial Armbian Configuration

On the first boot, Armbian performs the initial system configuration.

The configuration includes:

* Root password
* Default shell
* User creation
* User password
* Region
* Timezone

For the shell, Bash is used in this project.

A dedicated user named `stb` is created for normal system administration.

After the initial setup is completed, the system can be checked from the terminal.

---

# 4. Verify the Linux System

Basic system information can be checked using:

```bash
uname -a
```

The command can be used to verify that the system is running Linux on the expected architecture and kernel.

Network interfaces can be inspected using:

```bash
ip a
```

At this stage, Ethernet is used as the primary network connection.

---

# 5. Configure Network and SSH

The HG680P can obtain an IP address from the router using DHCP.

Check the assigned address with:

```bash
ip a
```

Once the IP address is known, SSH can be used for remote administration.

The SSH service uses the standard port:

```text
22
```

A client such as PuTTY can be used from Windows to connect to the STB.

After SSH access is available, most of the remaining configuration can be performed remotely without requiring a monitor and keyboard.

---

# 6. Identify the Internal WiFi

The internal WiFi adapter of the HG680P is not immediately available with the default Armbian kernel configuration.

The WiFi hardware uses an SDIO interface.

The wireless chipset used by this device is:

```text
Realtek RTL8188F
```

The hardware identification reported for the adapter is:

```text
024C:F179
```

The required Linux driver is:

```text
8189fs
```

The driver source used in this project is:

```text
https://github.com/jwrdegoede/rtl8189ES_linux
```

The required branch is:

```text
rtl8189fs
```

Before compiling the driver, the system can be checked with:

```bash
modinfo 8189fs
```

If the driver is not installed, the system returns an error similar to:

```text
modinfo: ERROR: Module 8189fs not found
```

This confirms that the required kernel module is not currently available.

---

# 7. Install Driver Build Dependencies

Update the package index:

```bash
sudo apt update
```

Install the required build tools:

```bash
sudo apt install -y git build-essential bc dkms make gcc wget curl
```

These packages provide the compiler, build utilities, Git, DKMS, and other tools required for compiling the kernel module.

---

# 8. Clone the RTL8189FS Driver

Clone the required driver branch:

```bash
git clone -b rtl8189fs https://github.com/jwrdegoede/rtl8189ES_linux.git
```

Enter the driver directory:

```bash
cd rtl8189ES_linux
```

Check the contents:

```bash
ls
```

The repository contains the source code and Makefile required to compile the driver.

---

# 9. Initial Driver Compilation

The initial compilation can be attempted using:

```bash
make ARCH=arm64 KSRC=/lib/modules/$(uname -r)/build
```

The build uses:

```text
ARCH=arm64
```

because the HG680P uses an ARM64 processor.

The kernel source/build directory is obtained from:

```bash
/lib/modules/$(uname -r)/build
```

On the tested environment, the initial build can fail because the compiler used by the kernel and the compiler selected for the module build are not compatible.

---

# 10. Configure the ARM64 Toolchain

To build the driver correctly for the ARM64 platform, an ARM GNU Toolchain is used.

Verify the compiler:

```bash
aarch64-none-linux-gnu-gcc --version
```

The compiler executable is:

```text
aarch64-none-linux-gnu-gcc
```

This compiler is used explicitly during the driver compilation.

---

# 11. Clean the Previous Build

Before rebuilding the driver, remove the previous build artifacts:

```bash
make clean
```

This ensures that the next compilation starts from a clean state.

---

# 12. Compile RTL8189FS with the ARM64 Compiler

Build the driver using:

```bash
make ARCH=arm64 \
CC=aarch64-none-linux-gnu-gcc \
KSRC=/lib/modules/$(uname -r)/build
```

The important parameters are:

* `ARCH=arm64` — specifies the target architecture.
* `CC=aarch64-none-linux-gnu-gcc` — explicitly selects the ARM64 compiler.
* `KSRC=/lib/modules/$(uname -r)/build` — points to the kernel build directory.

After a successful compilation, the kernel module should be generated.

Check the result using:

```bash
ls -lh *.ko
```

The expected module is:

```text
8189fs.ko
```

---

# 13. Install and Load the Kernel Module

After compilation succeeds, update the kernel module dependency database:

```bash
sudo depmod -a
```

Load the driver:

```bash
sudo modprobe 8189fs
```

Verify that the module is loaded:

```bash
lsmod | grep 8189
```

If the module is loaded successfully, the `8189fs` driver should appear in the output.

---

# 14. Verify the WiFi Interface

Check the available network interfaces:

```bash
ip link
```

The wireless interface should now be available.

NetworkManager can also be used to check wireless devices:

```bash
nmcli device
```

Available WiFi networks can be scanned with:

```bash
nmcli device wifi list
```

If nearby wireless networks are displayed, the RTL8189FS driver is working correctly.

---

# 15. Connect to WiFi

Connect to a wireless network using NetworkManager:

```bash
sudo nmcli device wifi connect "WIFI_NAME" password "WIFI_PASSWORD"
```

Replace:

```text
WIFI_NAME
```

with the actual wireless network name.

Replace:

```text
WIFI_PASSWORD
```

with the actual password.

Example:

```bash
sudo nmcli device wifi connect "MyWiFi" password "example-password"
```

Do not commit real WiFi credentials to this repository.

After connecting, verify the interface:

```bash
ip a
```

The wireless interface should have an IP address assigned by the router.

---

# 16. Troubleshooting

## Driver Module Not Found

If:

```bash
modinfo 8189fs
```

returns:

```text
modinfo: ERROR: Module 8189fs not found
```

the driver has not been installed or registered with the kernel.

Check whether the compiled module exists:

```bash
ls -lh *.ko
```

If the module exists, run:

```bash
sudo depmod -a
```

and then:

```bash
sudo modprobe 8189fs
```

---

## Compilation Failure

If the first compilation fails because of a compiler mismatch, verify the ARM64 compiler:

```bash
aarch64-none-linux-gnu-gcc --version
```

Then clean the previous build:

```bash
make clean
```

Recompile using:

```bash
make ARCH=arm64 \
CC=aarch64-none-linux-gnu-gcc \
KSRC=/lib/modules/$(uname -r)/build
```

---

## WiFi Interface Not Available

If the driver is loaded but the WiFi interface is not visible, check:

```bash
lsmod | grep 8189
```

Then:

```bash
ip link
```

Also check NetworkManager:

```bash
nmcli device
```

If necessary, inspect the kernel log:

```bash
dmesg | grep -i 8189
```

---

# 17. Security Considerations

This repository is public, so sensitive information must not be committed.

Do not publish:

* WiFi passwords
* SSH passwords
* Private SSH keys
* API tokens
* Cloud credentials
* Personal credentials
* Private configuration files containing secrets

Use placeholders in documentation, for example:

```text
WIFI_NAME
WIFI_PASSWORD
SERVER_IP
USERNAME
```

The commands in this repository are intended to be adapted to the user's own environment.

---

# 18. Project Result

After completing the configuration, the HG680P can run Linux using Armbian and can be administered remotely through SSH.

The internal RTL8189FS WiFi adapter is also recognized after compiling and loading the required `8189fs` kernel module.

The resulting environment provides:

* Armbian Linux
* ARM64 environment
* Ethernet connectivity
* SSH remote access
* RTL8189FS WiFi support
* NetworkManager WiFi management

This setup turns the original Android TV box into a usable Linux mini-server platform.

---

# 19. Next Stage

This project is the initial foundation for the HG680P mini-server environment.

The next stages include:

* Docker installation
* CasaOS installation
* Portainer
* Monitoring with Prometheus and Grafana
* Bash automation
* Ansible automation
* Docker Compose deployment
* Kubernetes K3s
* ArgoCD and GitOps

Each stage is documented in a separate repository as part of the overall mini-server and application deployment project.

---

## Author

**Muhammad Risyad Rahmadi**

Developer Engineer

Building, automating, and deploying applications with Linux, Docker, Kubernetes, CI/CD, and GitOps.
