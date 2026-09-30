# SLPI/SDSP crash loop on SM8150 (Xiaomi vayu) 

## 0. Environment
Mobian forky on Linux stable 7.0.1

## 1. Initial observations
On Xiaomi vayu device (sm8150 SoC) using Linux mainline, the SLPI firmware doesn't finish its initialization process.

Instead, it enters in a crash loop cycle every 40 seconds.

### 1-1. Without hexagonrpcd daemon
In the initial tests, without the hexagonrpcd daemon running, the symthoms in the logs were:
``` text
mobian kernel: qcom_q6v5_pas 2400000.remoteproc: fatal error received: err_qdi.c:964:EF:sensor_process:0x1:TMR_CLNT_1:0x80:dog_virtual_user.c:240:USER-PD DOG detects stalled initialization, triage with IMAGE OWNER
mobian kernel: remoteproc remoteproc0: crash detected in slpi: type fatal error
mobian kernel: remoteproc remoteproc0: handling crash #498 in slpi
mobian kernel: remoteproc remoteproc0: recovering slpi
mobian kernel: remoteproc remoteproc0: stopped remote processor slpi
mobian kernel: remoteproc remoteproc0: remote processor slpi is now up
```

This cycle of restartings is repeated every 40 seconds.

This cycle is also visible by checking the remoteproc(0) state:
```bash
cat /sys/class/remoteproc/remoteproc0/state
```

```text
running-->crashed-->offline-->running-->...
```

### 1-2. With hexagonrpcd daemon
After installing hexagonrpcd, the crash sequence persists with the same symthoms in the logs.

#### 1-2-1. Check hexagonrpcd status
```text
mobian systemd[1]: Started hexagonrpcd.service - Hexagon DSP sensors daemon.
mobian hexagonrpcd[3433]: Could not attach to FastRPC node: Broken pipe
mobian hexagonrpcd[3433]: Starting /usr/libexec/hexagonrpc/hexagonrpcd (INIT_ATTACH_SNS) on /dev/fastrpc-sdsp
mobian systemd[1]: hexagonrpcd.service: Main process exited, code=exited, status=4/NOPERMISSION
```

#### 1-2-2. Journal from hexagonrcd restart
Output from journal -f on hexagonrcd daemon restart: [Link to log](logs/journal-hexagonrpcd-restart_baseline.log)


## 2. Analysis
The main clue might be this message logged in the journal, when the hexagonrpc daemon tries to attach to the fastrpc-sdsp device (look al the linked log in the 1-2-2 section):
```text
mobian kernel: arm-smmu 15000000.iommu: Unhandled context fault: fsr=0x402, iova=0x1fffff000, fsynr=0x780001, cbfrsynra=0x5a1, cb=18
```

This suggests that the SDSP tries to access to an unallocated IOVA in the SMMU.

### 2-1. DMA adresses in fastrpc driver
There are two different concepts in the fastrpc driver related to the DMA addresses.

#### 2-1-1. Raw IOVA
It's the standar DMA virtual address used for the operations over the SMMU.

There are two different types:
- The coherent raw IOVA, related to the dma buffer allocation.
- The sg raw IOVA, related to the dma buffer mapping.
 
Each type has its dma mask, an integer value which determines the number of bits for the address range:
- The coherent_dma_mask, which is used to determine the coherent IOVA. 
- The dma_mask, which is used to determine the sg IOVA.

#### 2-1-2. Computed IOVA
The fastrpc driver also defines a computed (or consolidated) DMA address, by adding the SID offset bits to the raw IOVA in the higher address bits, and it's the address sent to the DSP.
The SID offset is used to know the correct context bak.

The next step would be to add some extra debug to the fastrpc driver for tracing the IOVA addresses and DMA masks.

### 2-2. Extended debug in mainline fastrpc driver
A new branch is created in the kernel tree for the issue debug and test: [mobian-sm8150-7.1.0-vayu-slpi-crash-debug](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/tree/mobian-sm8150-7.1.0-vayu-fix-slpi-WIP)

The first commit [1f0507153e1e0b972182c3baabda825a640562d3](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/1f0507153e1e0b972182c3baabda825a640562d3) adds extra debug to the fastrpc driver as info prints (FASTRPC-INFO) to see the effective DMA addresses and masks when hexagonrpcd attaches to the fastrpc-sdsp device.

#### 2-2-1. Analyis from collected clues from logs on mainline baseline
When the hexagonrpc daemon tries to attach to the fastrpc-sdsp device, the extra debug in the fastrpc driver shows this information in the journal log:
```text
mobian kernel: qcom,fastrpc-cb 2400000.remoteproc:glink-edge:fastrpc:compute-cb@1: FASTRPC-INFO: coherent-dma-addr=0x00000000fffff000 coherent-dma-mask=0xffffffff dma-mask=0xffffffff
mobian kernel: qcom,fastrpc-cb 2400000.remoteproc:glink-edge:fastrpc:compute-cb@1: FASTRPC-INFO: computed-coherent-dma-addr=0x00000001fffff000
```

The coherent (raw) IOVA "0xfffff000" (the virtual address for the allocated DMA buffer in the SMMU) is in the 32 bits range. This matches with its printed 32 bits coherent dma mask "0xffffffff".

By other hand, the computed address (by fastrpc driver) is in the 33 bits range "0x1fffff000", which is the coherent raw IOVA with the SID offset prepended (0x100000000), the offset for the context bank 1 (compute-cb@1 in the extended log).

So the computed address matches exactly with the IOVA in the "Unhandled context fault" from the journal on the hexagonrpcd attach.

### 2-3 Mainline fastrppc driver
The mainline fastrpc driver defines the computed address after the DMA buffer allocation happens.

And the computed address is sent to the DSP (as &msg->addr), which seems neccesary for the DSP to know the context bank.

But the DSP tries to access to the DMA buffer using the computed address as IOVA, not the raw address, and the allocatted address is the raw one.

So there is a mismatch between the allocated buffer's IOVA (32 bits, coherent raw address) and the address that the DSP tries to access (33 bits, the computed address).

Is this supposed to work on other devices?

Does this might suggest that the SDSP firmware should strip the SID offset and use the raw IOVA to access the DMA buffer, but it doesn't?

The second question seems to have an answer in the sm8150/vayu downstream's adsprpc driver.


### 2-4. Downstream adsprpc driver: SDSP hardware bug workaround
To understand what should be the fastrpc driver behaviour, the equivalent driver in downstream was analyzed: [adsprc.c from Xiaomi vayu downstream](https://github.com/MiCode/Xiaomi_Kernel_OpenSource/blob/vayu-r-oss/drivers/char/adsprpc.c)

It's especially revealing a workaround in the downstream driver for the sdsp domain to deal with a "HW bug" in the SMMU interconnect:
```text
static int fastrpc_cb_probe(struct device *dev)
{
    ...
	dma_addr_t start = 0x80000000;
    ...
	/* Software workaround for SMMU interconnect HW bug */
	if (cid == SDSP_DOMAIN_ID) {
		sess->smmu.cb = iommuspec.args[0] & 0x3;
		VERIFY(err, sess->smmu.cb);
		if (err)
			goto bail;
		start += ((uint64_t)sess->smmu.cb << 32);
		dma_set_mask(dev, DMA_BIT_MASK(34));
	} else {
		sess->smmu.cb = iommuspec.args[0] & 0xf;
	}
```

For the SDSP domain, the sid offset is added to the raw IOVA before the buffer allocation, and the dma mask is set to 34, so the raw IOVA contains the SID offset (equivalent to the "computed address" in the mainline fastrpc driver).

### 2-5. Relevant conclusons from the analysis
Before proceed to port the fix there are some relevat points to think about.

#### 2-5-1. Context banks and DMA mask
There are three context banks for the SDSP domain, defined in the sm8150 DT for both downtream and mainline: cb@1, cb@2 and cb@3.

##### 2-5-1-1. Context banks
There are three context banks for the SDSP domain defined in the sm8150 DT for both downtream and uptream: cb@1, cb@2 and cb@3.

In the upstream (Mobian) logs:
```text
[   44.647655] platform 2400000.remoteproc:glink-edge:fastrpc:compute-cb@1: Adding to iommu group 26
[   44.659891] platform 2400000.remoteproc:glink-edge:fastrpc:compute-cb@2: Adding to iommu group 27
[   44.668652] platform 2400000.remoteproc:glink-edge:fastrpc:compute-cb@3: Adding to iommu group 28
```

##### 2-5-1-2. DMA mask
The hexagonrpc daemon uses the cb@1 to attach to the fastrpc-sdsp device, and the computed virtual address (by fastrpc driver) is on top of the 33 bits range.

For cb@1, only the bit 32 is relevant (dma mask 33).

For cb's 2 and 3, the bit 33 (dma mask = 34) is required, which is the hardcoded mask in the downstream's exception for the SDSP domain.

In the upstream scenario, the address start range for cb@1 is 0x100000000 (bit 32), and for cb@2 and cb@3 should be 0x200000000 and 0x300000000 respectively (bits 33:32).


## 3. SDSP bug workaround in mainline (WIP Branch: Initial version)
The next step is try to implement the downtream's SDSP bug workaround in the upstream fastrpc driver.

The branch for the initial implementation and tests is [mobian-sm8150-7.1.0-vayu-fix-slpi-WIP](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/tree/mobian-sm8150-7.1.0-vayu-fix-slpi-WIP)

### 3-1. Required mechanisms
The necessary pieces are:

#### 3-1-1. Guarded soc_data
A mechanism in the fastrpc driver to set a custom soc_data table guarded for specific SoC (sm8150) and speific domain (sdsp).

The guard mechanism implemenation is based on the existing multiple soc_data architecture in the fastrpc driver.

The idea is taken from the Kaanapali SoC workaround in the upstream commits:
- 1d94ce8996d71d77e2d649db9e5c205f423e2c17 "misc: fastrpc: Add support for new DSP IOVA formatting"
- 8314d2c28d3369bc879af8e848f810292b16d0af "misc: fastrpc: Update dma_bits for CDSP support on Kaanapali SoC"

A new soc_data table is created to use custom soc data values only for the sm8150 SoC and sdsp domain.

This also requires to change/override the fastrpc compatible (qcom,sm8150-sdsp-fastrpc) in the slpi remoteproc node in the DT.

Implemented in commit [3906ad8b8191db72a26256e090fbef1320a1cb94](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/3906ad8b8191db72a26256e090fbef1320a1cb94)

#### 3-1-2. Custom coherent dma mask
A mechanism in the fastrpc driver to modify the coherent dma mask, since it affects to the allocation IOVA.
The idea is to force a 33 or 34 bits IOVA for the buffer allocation.

The approach used is based on the already existing dma_bits and dma_set_mask implementation. The main involved components are:
- dma_addr_coherent_bits_default: New constant equivalent to dma_addr_bits_default. It's defined in the soc_data with the mask value.
 - dma_coherent_bits: New variable equivalent to dma_bits.
- dma_set_coherent_mask: Function call to set the coherent mask using the dma_coherent_bits value for the dma buffer allocation.

The dma_bits could be used for the dma_set_coherent_mask instead the extra dma_coherent_bits introduction, but i guess is preferred to have them with independent assignements, at least for now.

Implemented in commit [99c07ab0efaf84c5535095340f21d63ef69d7ca9](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/99c07ab0efaf84c5535095340f21d63ef69d7ca9)

#### 3-1-3. Skip computed iova
Mechanism to allow not adding the sid offset to the dma address sent to the DSP.

The implemntation consists on return the raw IOVA for the coherent and sg addresses as computed addresses, by skipping the add sid offset in the "fastrpc_buf_alloc" and "fastrpc_compute_dma_addr" functions when the introduced variable "no_sid_offset" is set to "true" in the soc_data.

Also the reversed IOVA is skipped in the "dma_addr_t fastrpc_ipa_to_dma_addr" function, returning the IOVA as-is, also based on the no_sid_offset variable. This is required for some operations like free the address.

Implemented in commit [230ceac790b33091fdd3c2e8a218e1b670dcb186](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/230ceac790b33091fdd3c2e8a218e1b670dcb186)

#### 3-1-4. Shifted IOVA allocation in arm-smmu
Add support for shifted IOVA allocation workaround.

This workaround solves the problem in the Test 3 (Section 3-2-4), where the dma mask is set o 34 (like downstream) to cover the three context banks.

As mentioned above, the downstream's approach for the HW bug (in the adsprpc driver) is to shift the start address by adding the sid offset to it before the dma buffer allocation.

This workaround adds an equivalent approach in the uptream's arm-smmu driver.

In upstream, the allocation only uses aperture_end, so the resulting IOVA is in the higher range allowed by the dma mask.

For this reason, the dma mask 33 (Test 2, section 3-2-3) works for the context bank 1, but not for the 2 and 3.

By other hand, the dma mask 34 sould work only for he context bank 3, and doesn't work for context banks 1 and 2 (only confirmed that fails for CB1, but should fail with  CB2 too).

The implemented approach adds a new address range definition guarded by the new qcom,shifted-iova DT property (in the compute-cb@X nodes), using aperture_start = reg value from the DT (sid offset numeric value) + 32, and aperture_end = aperture_start + 32 (limiting the range to the expected for the context bank).

This approach should use the correct buffer allocation IOVA for each context bank.

The qcom,shifted-iova property is required in each compute-cb fastrpc subnode.

Implemented in commit [978e76a7f235071d2d6dd21d5fd0da755e3f7849](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/978e76a7f235071d2d6dd21d5fd0da755e3f7849)

### 3-2. Tests
This sub-section documents the tests based in the implemented mechanisms.

#### 3-2-1. Testing methods

##### 3-2-1-1. Attach and free equivalents from Linux terminal
For some tested scenarios, the hexagonrpc attach caused a hard system hang.

If the hexagonrpcd daemon is enabled when thiis scenario is found, a bootloop happens whhen the daemon is started at boot.

To deal with it in a more confortable way, the daemon can be masked and do the equivalent to attach and free to the fastrpc-sdsp device uing the next bash commands sequence:

- Attach equivalent:
```bash
exec 3<>/dev/fastrpc-sdsp
```
- Check that device is "attached":
```bash
ls -l /proc/self/fd/3
```
- Free equivalent:
```bash
exec 3>&-
```

#### 3-2-2. Test-1: Enable the skip computed iova mechanism
This test just enables the "no_sid_offset" variable in the "sm8150_sdsp_soc_data" to test the implementation and observe the dma addresses with the default coherent and sg dma masks (32 bits).

Implemented in commit [6e025cb0bd1f7695dae4d0ac13fee11be9315068](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/6e025cb0bd1f7695dae4d0ac13fee11be9315068)

This test just enables the "no_sid_offset" variable in the "sm8150_sdsp_soc_data" to test the implementation and observe the dma addresses with the default coherent and sg dma masks (32 bits).

The involved commits are:
- 3-1-1. Guarded soc_data [3906ad8b8191db72a26256e090fbef1320a1cb94](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/3906ad8b8191db72a26256e090fbef1320a1cb94)
- 3-1-3. Skip computed iova [230ceac790b33091fdd3c2e8a218e1b670dcb186](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/230ceac790b33091fdd3c2e8a218e1b670dcb186)
- 3-2-2. Enable no_sid_offset variable [6e025cb0bd1f7695dae4d0ac13fee11be9315068](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/6e025cb0bd1f7695dae4d0ac13fee11be9315068)

##### 3-2-2-1. Result Test-1
This test confirms that custom sm8150_sdsp_soc_data and the "no_sid_offset" mechanism are working.

The coherent address for the dma allocation now matches to the computed address (the one sent to the DSP).

However, the hexagonrpcd attachment to the fastrpc-sdsp device, causes a hard system hang.

In the pstore log:
```text
[   55.673398] qcom,fastrpc-cb 2400000.remoteproc:glink-edge:fastrpc:compute-cb@1: FASTRPC-INFO: coherent-dma-addr=0x00000000fffff000 coherent-dma-mask=0xffffffff dma-mask=0xffffffff
[   55.673540] qcom,fastrpc-cb 2400000.remoteproc:glink-edge:fastrpc:compute-cb@1: FASTRPC-INFO: computed-coherent-dma-addr=0x00000000fffff000
```

This might suggest that the dma address range for the context bank 1 (cb@1) expected by the SDSP should be in the 33 bits, the computed range (0x00000001fffff000).

For the next test, the dma masks will be set to 33 bits. This would cover only the context bank 1, but probably it's enough to test the theory.

#### 3-2-3. Test-2: DMA masks 33 + Enable the skip computed iova mechanism
This test combines "3-2-2. Test-1" with dma mask to 33 intead the defauls 32.

The involved commits are:
- 3-1-1. Guarded soc_data [3906ad8b8191db72a26256e090fbef1320a1cb94](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/3906ad8b8191db72a26256e090fbef1320a1cb94)
- 3-1-2. Custom coherent dma mask [99c07ab0efaf84c5535095340f21d63ef69d7ca9](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/99c07ab0efaf84c5535095340f21d63ef69d7ca9)
- 3-1-3. Skip computed iova [230ceac790b33091fdd3c2e8a218e1b670dcb186](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/230ceac790b33091fdd3c2e8a218e1b670dcb186)
- 3-2-2. Enable no_sid_offset variable [6e025cb0bd1f7695dae4d0ac13fee11be9315068](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/6e025cb0bd1f7695dae4d0ac13fee11be9315068)
- 3-2-3. Set dma masks to 33 bits [fef4b800c14e6900cadd34cf4daacab889432c45](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/fef4b800c14e6900cadd34cf4daacab889432c45)

##### 3-2-3-1. Result Test-2 (FIXED cb@1)
First good news.

This test fixed the attach/free problem.

Now the hexagorpcd completes succesfully for the context bank 1, and the reverse tunnel is working as exected, serving files to the SDSP.

Also te SLPI crash loop is fixed: no more reemoteproc crashes in logs for SLPI.

As additional information, the extended debug shows:
```text
mobian kernel: qcom,fastrpc-cb 2400000.remoteproc:glink-edge:fastrpc:compute-cb@1: FASTRPC-INFO: coherent-dma-addr=0x00000001ffed6000 coherent-dma-mask=0x1ffffffff dma-mask=0x1ffffffff
mobian kernel: qcom,fastrpc-cb 2400000.remoteproc:glink-edge:fastrpc:compute-cb@1: FASTRPC-INFO: computed-coherent-dma-addr=0x00000001ffed6000
```

There arn't errors for context banks 2 and 3, which suggests that they are not used/required by hexagonrcd.

Also there arn't more errors for "Unhandled context"

I don't know if the CBs 2 and 3 might be required for other scenarios, but they should require a 34 bts mask (the mask used by the downstream adsprpc driver) to use the correct dma address ranges (0x200000000 and 0x300000000).

The next test will be setting the dma masks to 34 to observe the behaviour.


#### 3-2-4. Test-3: DMA masks 34 + Enable the skip computed iova mechanism
This test combines "3-2-2. Test-1" with dma masks set to 34.

The involved commits are:
- 3-1-1. Guarded soc_data [3906ad8b8191db72a26256e090fbef1320a1cb94](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/3906ad8b8191db72a26256e090fbef1320a1cb94)
- 3-1-2. Custom coherent dma mask [99c07ab0efaf84c5535095340f21d63ef69d7ca9](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/99c07ab0efaf84c5535095340f21d63ef69d7ca9)
- 3-1-3. Skip computed iova [230ceac790b33091fdd3c2e8a218e1b670dcb186](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/230ceac790b33091fdd3c2e8a218e1b670dcb186)
- 3-2-2. Enable no_sid_offset variable [6e025cb0bd1f7695dae4d0ac13fee11be9315068](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/6e025cb0bd1f7695dae4d0ac13fee11be9315068)
- 3-2-4. Set dma masks to 34 bits [9e6f95933b590703c3156053172aac627cdd6ba1](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/9e6f95933b590703c3156053172aac627cdd6ba1)

##### 3-2-4-1. Result Test-3 (FAIL cb@1)
The test result shows that the address range used for the context bank 1 is the expected for the context bank 3 (0x300000000).

This suggests that the smmu allocates the buffer in the higher virtual address range allowed by the dma mask.

The log shows:
```text
mobian kernel: qcom,fastrpc-cb 2400000.remoteproc:glink-edge:fastrpc:compute-cb@1: FASTRPC-INFO: coherent-dma-addr=0x00000003fffff000 coherent-dma-mask=0x3ffffffff dma-mask=0x3ffffffff
mobian kernel: qcom,fastrpc-cb 2400000.remoteproc:glink-edge:fastrpc:compute-cb@1: FASTRPC-INFO: computed-coherent-dma-addr=0x00000003fffff000
```

By other hand, the sdsp attach fails again with the "Broen pipe" error. It probably suggests that the DSP expects the 0x100000000 range for the context bank 1.

So an option is to use the "3-2-3. Test-2" workaround for now, but it solves only the cb@1. The CBs 2 and 3 should fail if are required in some other scenario.
For the hexagonrpcd it seems to be enough.

#### 3-2-5. Test-4: Shifted-iova support for qcom DSP HW bug
This test combines "3-2-4. Test-3" with the shifted IOVA workaround (in arm-smmu driver)

For the test, the DT property is implemented at device level (sm8150-xiaomi-vayu.dtsi) in commit [1259f09c8af9ddd6070eb0cd71e78fcaed09e834](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/1259f09c8af9ddd6070eb0cd71e78fcaed09e834)

The involved commits are:
- 3-1-1. Guarded soc_data [3906ad8b8191db72a26256e090fbef1320a1cb94](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/3906ad8b8191db72a26256e090fbef1320a1cb94)
- 3-1-2. Custom coherent dma mask [99c07ab0efaf84c5535095340f21d63ef69d7ca9](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/99c07ab0efaf84c5535095340f21d63ef69d7ca9)
- 3-1-3. Skip computed iova [230ceac790b33091fdd3c2e8a218e1b670dcb186](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/230ceac790b33091fdd3c2e8a218e1b670dcb186)
- 3-2-2. Enable no_sid_offset variable [6e025cb0bd1f7695dae4d0ac13fee11be9315068](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/6e025cb0bd1f7695dae4d0ac13fee11be9315068)
- 3-2-4. Set dma masks to 34 bits [9e6f95933b590703c3156053172aac627cdd6ba1](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/9e6f95933b590703c3156053172aac627cdd6ba1)
- 3-1-4. Implement shifted IOVA allocation in arm-smmu [9e6f95933b590703c3156053172aac627cdd6ba1](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/9e6f95933b590703c3156053172aac627cdd6ba1)
- 3-2-5. Add qcom,shifted-iova prop to DT's slpi fastrpc CB nodes [1259f09c8af9ddd6070eb0cd71e78fcaed09e834](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/1259f09c8af9ddd6070eb0cd71e78fcaed09e834)

##### 3-2-5-1. Result Test-4 (FIXED cb@1)
The test result shows that thee CB1 works as expected using the dma mask 34 in he sm8150_sdsp_soc_data.
The CBs 2 and 3 are untested (not used by hexagonrpcd).

The hexagonrpc daemon works as exected, serving files to the SDSP.

The extended debug show:
```text
mobian kernel: qcom,fastrpc-cb 2400000.remoteproc:glink-edge:fastrpc:compute-cb@1: FASTRPC-INFO: coherent-dma-addr=0x00000001fff24000 coherent-dma-mask=0x3ffffffff dma-mask=0x3ffffffff
mobian kernel: qcom,fastrpc-cb 2400000.remoteproc:glink-edge:fastrpc:compute-cb@1: FASTRPC-INFO: computed-coherent-dma-addr=0x00000001fff24000
```
