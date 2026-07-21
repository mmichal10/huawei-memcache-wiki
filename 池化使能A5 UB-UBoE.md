# 使能A5 UB/UBoE 方法

## 1. 前置条件

宿主机执行如下命令，关闭验签：

```bash
此命令执行完，需要手动全部确认：Y
for i in {0..7}; do npu-smi set -t custom-op-secverify-enable -i $i -d 1; done;

for i in {0..7}; do npu-smi set -t custom-op-secverify-mode -i $i -d 0; done;
```

## 2. 协议选择

### 在 950 Pod机型中

```toml
ock.mmc.local_service.protocol = device_urma
```
使用的是6个出框口

### 在 950 Server机型中

```toml
ock.mmc.local_service.protocol = device_urma / device_uboe
```

device_urma：使用的是die0的8个出框口

device_uboe：使用是die1的1个参数面口



