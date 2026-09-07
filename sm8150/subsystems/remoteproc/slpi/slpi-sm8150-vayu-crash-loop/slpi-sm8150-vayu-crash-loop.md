# SLPI/SDSP crash loop on SM8150 (Xiaomi vayu) 

## 0. Environment
Mobian forky on Linux stable 7.0.1

## 1. Initial observations
On Xiaomi vayu device (sm8150 SoC) using Linux mainline, the SLPI firmware doesn't finish its initialization process.

Instead, it enters in a crash loop cycle every 40 seconds.

## 1-1. Without hexagonrpcd daemon
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

## 1-2. With hexagonrpcd daemon
After installing hexagonrpcd, the crash sequence persists with the same symthoms in the logs.

## 1-2-1. Check hexagonrpcd status
```text
mobian systemd[1]: Started hexagonrpcd.service - Hexagon DSP sensors daemon.
mobian hexagonrpcd[3433]: Could not attach to FastRPC node: Broken pipe
mobian hexagonrpcd[3433]: Starting /usr/libexec/hexagonrpc/hexagonrpcd (INIT_ATTACH_SNS) on /dev/fastrpc-sdsp
mobian systemd[1]: hexagonrpcd.service: Main process exited, code=exited, status=4/NOPERMISSION
```

## 1-2-2. Journal from hexagonrcd restart
Output from journal -f on hexagonrcd daemon restart: [Link to log](logs/journal-hexagonrpcd-restart_baseline.log)


## 2. Analysis
The main clue might be this message logged in the journal, when the hexagonrpc daemon tries to attach to the fastrpc-sdsp device (look al the linked log in the 1-2-2 section):
```text
mobian kernel: arm-smmu 15000000.iommu: Unhandled context fault: fsr=0x402, iova=0x1fffff000, fsynr=0x780001, cbfrsynra=0x5a1, cb=18
```

This suggests that the SDSP tries to access to an unallocated IOVA in the SMMU.

### 2-1. DMA adresses in fastrpc driver
There are two different concepts in the fastrpc driver related to the DMA addresses.

### 2-1-1. Raw IOVA
It's the standar DMA virtual address used for the operations over the SMMU.

There are two different types:
- The coherent raw IOVA, related to the dma buffer allocation.
- The sg raw IOVA, related to the dma buffer mapping.
 
Each type has its dma mask, an integer value which determines the number of bits for the address range:
- The coherent_dma_mask, which is used to determine the coherent IOVA. 
- The dma_mask, which is used to determine the sg IOVA.

### 2-1-2. Computed IOVA
The fastrpc driver also defines a computed (or consolidated) DMA address, by adding the SID offset bits to the raw IOVA in the higher address bits, and it's the address sent to the DSP.
The SID offset is used to know the correct context bak.

The next step would be to add some extra debug to the fastrpc driver for tracing the IOVA addresses and DMA masks.

## 2-2. Extended debug in mainline fastrpc driver
A new branch is created in the kernel tree for the issue debug and test: [mobian-sm8150-7.1.0-vayu-slpi-crash-debug](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/tree/mobian-sm8150-7.1.0-vayu-slpi-crash-debug)

The first commit [1f0507153e1e0b972182c3baabda825a640562d3](https://github.com/ticlnxcat-mobian/linux-mobian-sm8150-stable/commit/1f0507153e1e0b972182c3baabda825a640562d3) adds extra debug to the fastrpc driver as info prints (FASTRPC-INFO) to see the effective DMA addresses and masks when hexagonrpcd attaches to the fastrpc-sdsp device.

### 2-2-1. Analyis from collected clues from logs on mainline baseline
When the hexagonrpc daemon tries to attach to the fastrpc-sdsp device, the extra debug in the fastrpc driver shows this information in the journal log:
```text
mobian kernel: qcom,fastrpc-cb 2400000.remoteproc:glink-edge:fastrpc:compute-cb@1: FASTRPC-INFO: coherent-dma-addr=0x00000000fffff000 coherent-dma-mask=0xffffffff dma-mask=0xffffffff
mobian kernel: qcom,fastrpc-cb 2400000.remoteproc:glink-edge:fastrpc:compute-cb@1: FASTRPC-INFO: computed-coherent-dma-addr=0x00000001fffff000
```

The coherent (raw) IOVA "0xfffff000" (the virtual address for the allocated DMA buffer in the SMMU) is in the 32 bits range. This matches with its printed 32 bits coherent dma mask "0xffffffff".

By other hand, the computed address (by fastrpc driver) is in the 33 bits range "0x1fffff000", which is the coherent raw IOVA with the SID offset prepended (0x100000000), the offset for the context bank 1 (compute-cb@1 in the extended log).

So the computed address matches exactly with the IOVA in the "Unhandled context fault" from the journal on the hexagonrpcd attach.

## 2-3 Mainline fastrppc driver
The mainline fastrpc driver defines the computed address after the DMA buffer allocation happens.

And the computed address is sent to the DSP (as &msg->addr), which seems neccesary for the DSP to know the context bank.

But the DSP tries to access to the DMA buffer using the computed address as IOVA, not the raw address, and the allocatted address is the raw one.

So there is a mismatch between the allocated buffer's IOVA (32 bits, coherent raw address) and the address that the DSP tries to access (33 bits, the computed address).

Is this supposed to work on other devices?

Does this might suggest that the SDSP firmware should strip the SID offset and use the raw IOVA to access the DMA buffer, but it doesn't?

The second question seems to have an answer in the sm8150/vayu downstream's adsprpc driver.


## 2-4. Downstream adsprpc driver: SDSP hardware bug workaround
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

The next step is try to implement the downtream's SDSP bug workaround in the mainline fastrpc driver.
