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
- Phase 1 Hi3861 + LiteOS_m + v6.0.0
- Phase 2 OpenHarmony + c910
- Phase 3 VirtIO + IOMMU + AIA
- Phase 4 CUDA

## OHOS setup
- [archived source code](https://gitee.com/link?target=https%3A%2F%2Frepo.huaweicloud.com%2Fopenharmony%2Fos%2F4.1-Beta1%2Fcode-v4.1-Beta1.tar.gz)
- official docker
```
sudo docker pull swr.cn-south-1.myhuaweicloud.com/openharmony-docker/docker_oh_standard:3.2
```
- build in docker
```
cd OpenHarmony-v4.1-Beta1/OpenHarmony
sudo docker run -it -v $(pwd):/home/openharmony swr.cn-south-1.myhuaweicloud.com/openharmony-docker/docker_oh_standard:3.2
python3 -m pip install --user build/hb
./build.sh --product-name qemu_riscv_mini_system_demo
./build.sh --product-name hispark_pegasus_mini_system
```
