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
