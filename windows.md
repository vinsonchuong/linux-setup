# Running a Windows 11 VM on Arch Linux

```
pacman -S qemu-full swtpm edk2-ovmf
```

## Enabling Secure Boot
Windows 11 (by default) will not run except with
[Secure Boot over UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface#Secure_Boot).

This requires that all UEFI drivers and OS boot loaders to be loaded into the
virtual machine be signed with known keys.

Not all Linux distributions ship signed UEFI firmware. Arch Linux does not.
[Ubuntu provides signed firmware](https://packages.ubuntu.com/jammy/ovmf).

Such packages usually provide several variants of `OVMF_CODE.fd` and
`OVMF_VARS.fd`. I've found that my Windows VM will boot with some but not all
of the variants.

Provide the firmware to a VM like so:

```sh
qemu-system-x86_64 \
  -drive if=pflash,format=raw,unit=0,readonly=on,file="$PWD/OVMF_CODE_4M.secboot.fd" \
  -drive if=pflash,format=raw,unit=1,file="$PWD/OVMF_VARS_4M.fd" \
  -global driver=cfi.pflash01,property=secure,value=on \
  -global ICH9-LPC.disable_s3=1

```

### References
- [Unified Extensible Firmware Interface/Secure Boot](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Creating_keys)
- [QEMU, OVMF and Secure Boot](https://github.com/rhuefi/qemu-ovmf-secureboot)
- [OVMF OVERVIEW](https://github.com/tianocore/edk2/blob/master/OvmfPkg/README)
- [SecureBoot: VirtualMachine](https://wiki.debian.org/SecureBoot/VirtualMachine)
- [Can't enable Secure Boot in qemu](https://bbs.archlinux.org/viewtopic.php?id=275691)

## Simulating TPM Version 2.0
Windows 11 requires that a TPM be available.

For a VM, [SWTPM](https://github.com/stefanberger/swtpm) can be used to simulate
it.

To start a simulated TPM, run:

```sh
mkdir "$PWD/tpm"
swtpm socket --tpmstate dir="$PWD/tpm" --ctrl type=unixio,path="$PWD/tpm/socket" --tpm2 --log level=20

```

Then, provide it to a VM like so:

```sh
qemu-system-x86_64 \
  -chardev socket,id=chrtpm,path="$PWD/tpm/socket" \
  -tpmdev emulator,id=tpm0,chardev=chrtpm -device tpm-tis,tpmdev=tpm0
```

### References
- [Trusted Platform Module](https://en.wikipedia.org/wiki/Trusted_Platform_Module)
- [How to Create a Windows 11 Virtual Machine in QEMU](https://www.tecklyfe.com/how-to-create-a-windows-11-virtual-machine-in-qemu/)

## General System Requirements
Windows 11 requires at minimum:

- 2 CPU cores
- 4GB of memory
- 64GB of storage

These can be provided like so:

```sh
# This disk image will only consume as much actual disk space as is used by the
# VM.
qemu-img create -f qcow2 storage.img 128G

qemu-system-x86_64 \
  -cpu host \
  -m 4G \
  -smp cores=2,threads=1
```

### References
- [Windows 11 System Requirements](https://support.microsoft.com/en-us/windows/windows-11-system-requirements-86c11283-ea52-4782-9efd-7674389a7ba3)
- [QEMU / KVM CPU model configuration](https://qemu.readthedocs.io/en/latest/system/qemu-cpu-models.html#qemu-command-line)

## Installing Windows 11
```sh
qemu-system-x86_64 \
  -cdrom windows11.iso
```

### References
- [Download Windows 11](https://www.microsoft.com/en-us/software-download/windows11)

## Running the VM
```sh
qemu-img create -f qcow2 storage.img 128G

swtpm socket --tpmstate dir="$PWD/tpm" --ctrl type=unixio,path="$PWD/tpm/socket" --tpm2 --log level=20
cp /usr/share/edk2-ovmf/x64/{OVMF_CODE.secboot,OVMF_VARS}.fd .

qemu-system-x86_64 \
  -enable-kvm \
  -cpu host \
  -m 4G \
  -machine q35,smm=on,accel=kvm \
  -smp cores=2,threads=1 \
  -hda storage.img \
  -chardev socket,id=chrtpm,path="$PWD/tpm/socket" \
  -tpmdev emulator,id=tpm0,chardev=chrtpm -device tpm-tis,tpmdev=tpm0 \
  -drive if=pflash,format=raw,unit=0,readonly=on,file="$PWD/OVMF_CODE_4M.secboot.fd" \
  -drive if=pflash,format=raw,unit=1,file="$PWD/OVMF_VARS_4M.fd" \
  -global driver=cfi.pflash01,property=secure,value=on \
  -global ICH9-LPC.disable_s3=1 \
  -cdrom windows11.iso
```



```sh
sleep 1; swtpm socket --tpmstate dir="$PWD/tpm" --ctrl type=unixio,path="$PWD/tpm/socket" --tpm2 --log level=20 & qemu-system-x86_64 \
  -enable-kvm \
  -cpu host \
  -m 4G \
  -machine q35,smm=on,accel=kvm \
  -smp cores=2,threads=1 \
  -hda storage.img \
  -chardev socket,id=chrtpm,path="$PWD/tpm/socket" \
  -tpmdev emulator,id=tpm0,chardev=chrtpm -device tpm-tis,tpmdev=tpm0 \
  -drive if=pflash,format=raw,unit=0,readonly=on,file="$PWD/OVMF_CODE_4M.secboot.fd" \
  -drive if=pflash,format=raw,unit=1,file="$PWD/OVMF_VARS_4M.fd" \
  -global driver=cfi.pflash01,property=secure,value=on \
  -global ICH9-LPC.disable_s3=1
```





## Setting Up a Development Environment

### Installing WSL
[ArchWSL](https://github.com/yuk7/ArchWSL)
WSL vs MinGW vs Cygwin??

In an Administrator PowerShell, run:

```PowerShell
wsl --install
```

This will install Ubuntu 20.04, which can be upgraded to 22.04. After the
installation is complete, open the Ubuntu app and run:

```sh
sudo apt update
sudo apt upgrade
sudo apt dist-upgrade
sudo do-release-upgrade -d
```

#### References
- [Installing Linux on Windows with WSL](https://docs.microsoft.com/en-us/windows/wsl/install)
- [How to Upgrade WSL 2 or 1 Ubuntu 20.04 to 22.04 LTS](https://www.how2shout.com/linux/how-to-upgrade-wsl-2-or-1-ubuntu-20-04-to-22-04-lts/)
