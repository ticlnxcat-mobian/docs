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
