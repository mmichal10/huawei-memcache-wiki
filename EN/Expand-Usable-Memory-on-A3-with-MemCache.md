##### 1. Background

The DRAM address range on the HCCS bus is fixed, and some servers' DRAM address space does not fully overlap it. MemCache can use only the portion of DRAM address space that overlaps the HCCS bus range.

Some deployments have the following limitations:

- **A3 server with 2 TB of memory:** The overlap between DRAM and HCCS address ranges is about **680 GB**, leaving much of the physical memory unavailable to MemCache.
- **A3 server with 1 TB of memory:** There is **no overlap**, so MemCache cannot allocate usable DRAM and the MemCache solution cannot be used.

[DRAM address ranges on the HCCS bus](https://gitcode.com/Ascend/memfabric_hybrid/wiki/FAQ.md#2-%E8%B0%83%E7%94%A8halmemcreate%E8%BF%94%E5%9B%9E%E9%94%99%E8%AF%AF%E7%A0%81-6):

- 0x29580000000-0x34000000000
- 0xa9580000000-0xb4000000000
- 0x129580000000-0x134000000000
- 0x1a9580000000-0x1b4000000000

Because the server DRAM layout does not match the HCCS bus address ranges, a large amount of physical memory is unavailable to MemCache.

##### 2. Solution

Adjust the server DRAM address range so that it overlaps as much as possible with the HCCS bus address ranges. This expands the DRAM space that MemCache can allocate.

##### 3. Procedure

###### 3.1 Download the BIOS Patch

- **Atlas 900 A3 SuperPoD:** Download and extract `Atlas-900-A3-SuperPoD-BIOS_32.85.zip` from [Atlas 900 A3 SuperPoD version 1.0.8.1 and patch files](https://support.huawei.com/enterprise/zh/atlas-computing/atlas-900-a3-superpod-pid-261207247/software/269547276?idAbsPath=fixnode01|23710424|251366513|22892968|252309113|261207247).
- **Atlas 800T A3:** Download and extract `Atlas-800T-A3-Atlas-800I-A3-BIOS_32.85.zip` from [Atlas 800T A3 version 1.0.5.1 and patch files](https://support.huawei.com/enterprise/zh/atlas-computing/atlas-800t-a3-pid-261716443/software/269547277?idAbsPath=fixnode01|23710424|251366513|22892968|252309113|261716443).

###### 3.2 Install the BIOS Patch Through iBMC

**Step 1: Sign in to the iBMC management page.**

Open the server's iBMC management interface.

**Step 2: Open the firmware update page.**

Select **Firmware Update**.

![image5.png](https://raw.gitcode.com/user-images/assets/7672915/61ef178d-5a41-4243-af57-01e29f1e4830/image5.png 'image5.png')

**Step 3: Upload and install the BIOS patch.**

Go to **Firmware Update → Upload BIOS Patch → Start Update**.

![image 4.png](https://raw.gitcode.com/user-images/assets/7672915/33c0db8e-e8b7-47d2-a727-1bcf9384811e/image_4.png 'image 4.png')

###### 3.3 Restart the Server

After the BIOS patch is installed, restart the server and set **Advanced → Memory Configuration → MSD Base Address Adjust** to **Enabled** so the new DRAM address configuration takes effect.

##### 4. Verification

After the BIOS patch takes effect, verify the changes as follows.

**Method 1: Check the DRAM address space.**

Run:

```bash
lsmem
```

Compare the server's DRAM address ranges before and after installing the BIOS patch, then calculate the overlap with the HCCS bus address ranges.

**Method 2: Check the memory available to MemCache.**

Get [mem_scan.py](https://gitcode.com/Ascend/memfabric_hybrid/blob/master/src/smem/python/memfabric_hybrid/memfabric_hybrid/mem_scan.py) and scan the DRAM space available to MemCache:

```bash
python3 mem_scan.py
```

The available memory should increase significantly, and the scan result should approach the server's physical memory capacity.

##### 5. Verification Cases

###### Case 1: A3 Server with 1 TB of Memory

**Before installing the BIOS patch**

The server's DRAM address space does not overlap the HCCS bus address ranges, so no memory is available to MemCache.

![1.png](https://raw.gitcode.com/user-images/assets/7672915/a92572ef-63d9-40f1-8b6d-43f509a29d48/1.png '1.png')

**After installing the BIOS patch**

The adjusted DRAM address space is fully covered by the HCCS bus address ranges.

![2.png](https://raw.gitcode.com/user-images/assets/7672915/d6d3d030-d3c3-466a-8b15-34fd067f0819/2.png '2.png')

**Result:**

- The DRAM address space now overlaps the HCCS bus address ranges.
- MemCache can allocate DRAM.

###### Case 2: A3 Server with 2 TB of Memory

**Before installing the BIOS patch**

The overlap between the server's DRAM address space and HCCS bus ranges is limited.

![3.png](https://raw.gitcode.com/user-images/assets/7672915/78e8fb62-95d0-4016-8baa-9f4a116fadfc/3.png '3.png')

**After installing the BIOS patch**

The adjusted DRAM address space is fully covered by the HCCS bus address ranges.

![4.png](https://raw.gitcode.com/user-images/assets/7672915/76025082-d093-4862-9496-703209ac9a79/4.png '4.png')

The `mem_scan.py` result approaches 2 TB, showing that the BIOS patch has increased the memory available to MemCache.

![5.png](https://raw.gitcode.com/user-images/assets/7672915/7fbd1f92-6b80-4c22-bc0c-6d16c48d3c98/5.png '5.png')

##### 6. Summary

Adjusting the server DRAM layout to overlap with the HCCS bus address ranges can significantly increase the DRAM capacity available to MemCache.

The verification results show:

- An A3 server with 1 TB of memory goes from having no usable space to supporting MemCache allocations.
- An A3 server with 2 TB of memory gains usable space approaching the server's full physical memory capacity.

This approach addresses memory waste caused by mismatched DRAM address layouts. It may also apply to similar memory-address issues on other interconnects, such as the Lingqu bus.
