# Rubik Pi 3 (QCS6490) — mainline kernel with NPU + GPU + Venus + NVMe

Branch base: linux-msm/laptops-kernel 7.1.0-rc6 (ddd664bbf).
Goal: run the Hexagon NPU (QNN/HTP), Adreno GPU (freedreno/OpenCL), Venus video
decode, and the NVMe SSD — all on ONE mainline kernel (no downstream 6.8 BSP).

## Changes in this branch
1. `drivers/misc/fastrpc.c` + `include/uapi/misc/fastrpc.h` — the QCS6490 6.8 BSP
   fastrpc (from rubikpi-ai/linux-ubuntu) ported to 7.1: adds FASTRPC_IOCTL_MMAP
   handling of `req.vaddrin` (user-allocated / dma-buf pages) for unsigned PDs,
   which QNN/libcdsprpc needs to load the HTP skel onto the cDSP. Stock mainline
   rejects it with EINVAL ("adding user allocated pages is not supported").
   API-drift fixes vs 6.8: `fastrpc_cb_remove` returns void (platform_driver.remove
   signature change); `MODULE_IMPORT_NS("DMA_BUF")` string-literal form.
2. `rubikpi3-mainline-npu-gpu.config` — working .config. KEY: `CONFIG_DMABUF_HEAPS=y`
   + `_SYSTEM=y` + `_CMA=y` (gives /dev/dma_heap/system; libcdsprpc allocates the
   HTP skel-load buffer from it — without it QNN fails INVALID_CONFIG / qnn_open 0x68).

## Build
```
cp rubikpi3-mainline-npu-gpu.config .config
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc) Image modules dtbs
```
DTB: `arch/arm64/boot/dts/qcom/qcs6490-thundercomm-rubikpi3.dtb` (build with
`DTC_FLAGS=-@` so it keeps `__symbols__`, required by the Qualcomm UEFI dtb overlay).

## Runtime / boot
- Userspace: ORT QNN EP + QAIRT 2.43 (`ADSP_LIBRARY_PATH=/usr/lib/rfsa/adsp;/usr/lib/dsp/cdsp`).
- GPU: Mesa with rusticl (freedreno). For zero-copy Venus->GPU prep, rusticl needs a
  `cl_khr_external_memory_dma_buf` patch (small frontend addition).
- Venus firmware: /lib/firmware/qcom/vpu-2.0/venus.mbn (decompress vpu20_p1.mbn.zst).
- Kernel cmdline (NVMe D3cold + flaky bits):
  `pcie_aspm=off pcie_port_pm=off nvme_core.default_ps_max_latency_us=0`
  `modprobe.blacklist=coresight,...,lpassaudiocc_sc7280`
- GPU dtb via GRUB `devicetree <rubikpi3.dtb>`. **CRITICAL: `sync` the disk before a
  COLD power-cycle** — otherwise the Qualcomm UEFI ufdt overlay fixup reads a
  partially-written dtb and aborts ("alloc magic is broken").

## Verified (2026-06-07)
NPU: QNN det ~12 ms/batch (HTP LIVE). GPU: rusticl FD643 OpenCL 3.0. Venus: v4l2h264dec
NV12. NVMe: ADATA 931G. All on uname 7.1.0-rc6 simultaneously.
