# Rubik Pi 3 (QCS6490) — mainline kernel with NPU + GPU + Venus + NVMe

Base: linux-msm/laptops-kernel 7.1.0-rc6 (ddd664bbf). Runs Hexagon NPU (QNN/HTP),
Adreno GPU (freedreno/OpenCL), Venus decode, and NVMe — all on ONE mainline kernel.

## Changes
1. `drivers/misc/fastrpc.c` + uapi — QCS6490 6.8 BSP fastrpc (rubikpi-ai/linux-ubuntu)
   ported to 7.1: FASTRPC_IOCTL_MMAP handling of `req.vaddrin` (user/dma-buf pages)
   for unsigned PDs — QNN/libcdsprpc needs it to load the HTP skel onto the cDSP
   (stock mainline returns EINVAL "adding user allocated pages is not supported").
   7.1 API fixes: `fastrpc_cb_remove` returns void; `MODULE_IMPORT_NS("DMA_BUF")`.
2. `rubikpi3-mainline-npu-gpu.config` — working .config; KEY `CONFIG_DMABUF_HEAPS=y`
   (+ _SYSTEM/_CMA) → /dev/dma_heap/system, where libcdsprpc allocates the skel-load
   buffer (without it: QNN INVALID_CONFIG / qnn_open 0x68).

## Build
    cp rubikpi3-mainline-npu-gpu.config .config
    make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig
    make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc) Image modules dtbs
DTB qcs6490-thundercomm-rubikpi3.dtb: build with DTC_FLAGS=-@ (keep __symbols__).

## Boot notes
- cmdline: `pcie_aspm=off pcie_port_pm=off nvme_core.default_ps_max_latency_us=0`
  `modprobe.blacklist=coresight,...,lpassaudiocc_sc7280`
- GPU dtb via GRUB `devicetree`; **`sync` before a COLD power-cycle** or the Qualcomm
  UEFI ufdt overlay fixup reads a half-written dtb and aborts ("alloc magic is broken").
- Venus fw: /lib/firmware/qcom/vpu-2.0/venus.mbn. GPU OpenCL: Mesa rusticl
  (+ cl_khr_external_memory_dma_buf patch for zero-copy Venus->GPU prep).

## Verified 2026-06-07 (uname 7.1.0-rc6, simultaneously)
NPU QNN ~12 ms/batch HTP LIVE; GPU rusticl FD643 OpenCL 3.0; Venus v4l2h264dec NV12;
NVMe ADATA 931G.
