# QEMU + Buildroot Setup for Kernel Testing
The following is a simple guide on how I set up a testing framework for testing new libbpf-tools. QEMU boots using buildroot and a modified buildroot configuration.

## Workspace setup
Create a workspace inside your home directory. Buildroot and the kernel you choose to build will be on the same directory level inside this workspace.

```zsh
mkdir libbpf-test && cd libbpf-test
```

## General setup for QEMU and its dependencies
```zsh
sudo apt update
sudo apt install make gcc flex bison libncurses-dev libelf-dev libssl-dev
sudo apt install qemu-system-x86
```
## Building kernel source code
Choose a kernel version to test, tar and cd into the directory. Copy the configuration of your current system to the .config file. Use `oldefconfig` to set all the defaults.

Reference: https://www.maketecheasier.com/build-custom-kernel-ubuntu/
```zsh
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.16.19.tar.xz
tar xavf linux-5.16.19.tar.xz && cd ./linux-5.16.19
cp /boot/config-`uname -r` .config
make olddefconfig
make menuconfig
```
### menuconfig options
**QEMU**
- Device drivers → Network device support → Virtio network driver <*>
- Device drivers → Block devices → Virtio block driver <*>

**Custom Kernel**
- Binary Emulations → x32 ABI for 64-bit mode, turn this OFF [ ]
- Enable loadable modules support → Module unloading - Forced module unloading [*]

```zsh
make clean
scripts/config --disable SYSTEM_TRUSTED_KEYS
scripts/config --disable SYSTEM_REVOCATION_KEYS
make -j `getconf _NPROCESSORS_ONLN`
```
Apply the pahole patch only if you get the following error while trying to run `make`.  
`FAILED: load BTF from vmlinux: Invalid argument`
```zsh
cd scripts/
cp pahole-flags.sh pahole-flags.sh.bak
vim pahole-flags.patch
patch pahole-flags.sh < pahole-flags.patch
```
If pahole-flags.sh is not available within the scripts directory, check if link-vmlinux.sh exists and make the following change under `gen_btf()`
```diff 
-LLVM_OBJCOPY="${OBJCOPY}" ${PAHOLE} -J ${PAHOLE_FLAGS} ${1}
+LLVM_OBJCOPY="${OBJCOPY}" ${PAHOLE} -J --skip_encoding_btf_enum64 ${PAHOLE_FLAGS} ${1}
```
Run make, check if bzImage has been generated within linux-x.x.x/arch/x86/boot
```zsh
make -j `getconf _NPROCESSORS_ONLN`
```
Create `run.sh` to boot QEMU with your kernel configuration. This should be done outside your kernel directory.
```zsh
vim run.sh
```
Edit run.sh to include the following, change the location of the bzImage if needed. 
```zsh
sudo qemu-system-x86_64 \
  -kernel linux-5.16.19/arch/x86_64/boot/bzImage \
  -nographic \
  -drive format=raw,file=buildroot/output/images/rootfs.ext4,if=virtio \
  -append "root=/dev/vda console=ttyS0 nokaslr other-paras-here-if-needed" \
  -m 4G \
  -smp 2 \
  -net nic,model=virtio \
  -net user,hostfwd=tcp::10022-:22 \
```
Get Buildroot. For the menuconfig options, there is a patch file that I've included with the changes I made. Use `chmod +x` to make the script executable, run and login as root. Use `scp` to copy the binary from your system to QEMU. Run and test.
```zsh
git clone https://github.com/buildroot/buildroot.git buildroot
cd buildroot
make menuconfig
make -j$(nproc)
cd ..
vim run.sh
chmod +x run.sh
./run.sh
```

@kerennaomidev
Comment
