# Enable UB and UBoE Pooling on 950DT and 950PR

## 1. Prerequisites

On the host, run the following commands to disable signature verification. Confirm each prompt manually by entering `Y`.

```bash
for i in {0..7}; do npu-smi set -t custom-op-secverify-enable -i $i -d 1; done;

for i in {0..7}; do npu-smi set -t custom-op-secverify-mode -i $i -d 0; done;
```

## 2. Select a Protocol

### 950 Pod

```toml
ock.mmc.local_service.protocol = device_urma
```

This uses the six external ports.

### 950 Server

```toml
ock.mmc.local_service.protocol = device_urma / device_uboe
```

- `device_urma`: uses the eight external ports on die 0.
- `device_uboe`: uses the one scale-up port on die 1.

## FAQ

1. Inference fails with `device_urma launch kernel AclrtSynchronizeStream failed, ret: 507018`.

   ![image.png](https://raw.gitcode.com/user-images/assets/7672915/5e10fbbf-71a7-4131-a113-296895fba2ab/image.png 'image.png')

   Check the plog for the corresponding process (1003). If it contains this error:

   ```
   PrintAicpuErrorInfo:AI CPU kernel execution failed, device_id=1, stream_id=280, task_pos=0, fault op_name=, soName=libcann_hybm_kernel.so, funcName=HybmBatchWrite, kernelName=HybmBatchWrite
   ```

   Manually import the `memfabric_hybrid` package once before starting the inference process:

   ```bash
   python -c "import memfabric_hybrid"
   ```
