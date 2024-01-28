# reference repositories:
1. Official v8.2.0
    - T-Head vendor extensions
    - Multi-threded TCG
    - KVM support
    - RISCV AIA
    - RISCV VirtIO Board
2. XThead uncommited
    - xuantie-iommu-v1
    - xuantie-qemu-6.1.0
3. dependencies and guest SW
    - wujian100_open
    - riscv-aosp
    - tvm
    - xuantie-gnu-toolchain
    - openc910
    - xuantie-yocto
    - u-boot

# target RV non-ISA extensions:
1. IO-MMU
2. CHI
3. UCIe
4. Semi-hosting

# target SW stacks:
1. ROS2
2. FreeRTOS
3. OPtee
4. Android 14/AOSP
5. CUDA
6. OpenHarmony

# focus plan
- Phase 1 qemu v8.2.0 rv32 + qemu_riscv_mini_system_demo
- Phase 2 qemu_riscv_mini_system_demo + thead llvm + debug/logging
- Phase 3 dayu800/visonFive2 + rv64
- Phase 4 VirtIO + IOMMU + AIA
- Phase 5 CUDA

## OHOS setup
- [archived source code](https://gitee.com/link?target=https%3A%2F%2Frepo.huaweicloud.com%2Fopenharmony%2Fos%2F4.1-Beta1%2Fcode-v4.1-Beta1.tar.gz)
- official docker
```
sudo docker pull swr.cn-south-1.myhuaweicloud.com/openharmony-docker/docker_oh_standard:3.2
```
## Phase 1
- build in docker
```
cd OpenHarmony-v4.1-Beta1/OpenHarmony
sudo docker run -it -v $(pwd):/home/openharmony swr.cn-south-1.myhuaweicloud.com/openharmony-docker/docker_oh_standard:3.2
./build.sh --product-name qemu_riscv_mini_system_demo
./build.sh --product-name hispark_pegasus_mini_system
```
- qemu run
```
hb set; hb build
cp out/ohos_config.json
./qemu-run
.qemu-system-riscv32 -M virt -m 128M -bios none -kernel out/riscv32_virt/qemu_riscv_mini_system_demo/OHOS_Image -global virtio-mmio.force-legacy=false -device virtio-gpu-device,xres=800,yres=480 -device virtio-tablet-device -vnc :20 -serial mon:stdio -drive if=pflash,file=vendor/ohemu/qemu_riscv32_mini_system_demo/fs-storage.img,format=raw,index=1 -append root="/dev/vda or console=ttyS0"
```
## Phase 2
- prepare new hb set for build and qemu-run
```
vi /home/openharmony/qemu-run
-> replace "ohos_config.json" to "out/ohos_config.json"

cp /home/openharmony/vendor/ohemu/qemu_riscv32_mini_system_demo
-> replace */"qemu_riscv32_mini_system_demo" to */"{target_name}"
```
- replace rv32 compiler
0. download toolchain from https://www.xrvm.cn/community/download?id=4267734522939904000 (reference https://www.xrvm.cn/document?temp=bhr332&slug=xuantie-cpu-userguide)
    1. /home/kkt/codes/lazyoung/prja/OpenHarmony-v4.1-Beta1/OpenHarmony/device/qemu/riscv32_virt/liteos_m/config.gni `board_toolchain_prefix = "board_arch = "rv32imafdcv_zihintpause_zfh_zba_zbb_zbc_zbs_xtheadc riscv64-unknown-elf-" "-mcpu=c908v-rv32",`
    2. prja/OpenHarmony-v4.1-Beta1/OpenHarmony/kernel/liteos_m/kal/libc/newlib/porting/include/sys/_pthreadtypes.h `//typedef __uint32_t pthread_key_t;`
    3. prja/OpenHarmony-v4.1-Beta1/OpenHarmony/kernel/liteos_m/kal/posix/src/pthread.c `int     pthread_setschedparam (pthread_t __pthread, int __policy,
                               const struct sched_param *__param);`
    4. prja/OpenHarmony-v4.1-Beta1/OpenHarmony/foundation/ability/ability_lite/services/abilitymgr_lite/src/slite/ability_mgr_service_slite.cpp
    `extern "C" { void *__dso_handle = 0; }`
- replace thead qemu
