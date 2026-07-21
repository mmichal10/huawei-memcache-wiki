# 使能A5 UB/UBoE 方法

## 1. 前置条件

宿主机执行如下命令，关闭验签：

```
此命令执行完，需要手动全部确认：Y
for i in {0..7}; do npu-smi set -t custom-op-secverify-enable -i $i -d 1; done;

for i in {0..7}; do npu-smi set -t custom-op-secverify-mode -i $i -d 0; done;
```

## 2. 协议选择

```

```