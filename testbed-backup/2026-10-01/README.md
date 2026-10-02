# MOCN testbed recovery snapshot - 2026-10-01

This is a personal recovery branch, not additional content for PR #271.
Branch: `backup/after-two-approvals-2026-10-01` in
`kevin8023c/Duranta-openairinterface5g`.

## Code and configuration provenance

- Code baseline: `5e0c0ed9f60062a553c76ae4bcc30e8974d361bd` (the final six-layer PR revision).
- The tested monolithic configuration files were read from the **older checkout**
  `/users/Yuanhao/Duranta-openairinterface5g/targets/PROJECTS/GENERIC-NR-5GC/CONF/`,
  while the final tests used binaries from `/users/Yuanhao/mocn-layered-candidate-20260929`.
- This branch restores those exact `gnb.sa.band78.fr1.106PRB.usrpb210.conf`, `ue.conf`,
  and `ue2.conf` files at their original repository-relative paths.
- Both UE files include `channelmod_rfsimu_LEO_satellite.conf`. This dependency is
  already tracked in the code baseline and matches the tested copy byte for byte.
- The sibling `cu.mocn-split.conf`, `du.mocn-split.conf`, and `mocn-plmn-fix-plan.md`
  are preserved from `/users/Yuanhao/mocn-split-test/` without editing their contents.

The CU/DU plan is **unimplemented historical design**, based on `80cb69d6469880d3ccb96dbe4b734f1b0c37cb96`.
Its line numbers, paths, assumptions, and proposed changes must be re-audited against
the code used for any future implementation. Archiving it does not certify the design.

## Before restoring

The old testbed used Ubuntu 22.04.2 LTS and kernel `5.15.0-187-generic`.
Install build dependencies using the checked-out project's instructions. Recreate the
two cores, DNs, namespaces, routing and ARP separately; none of their state is backed up here.

The saved gNB uses N2 `155.98.36.17/22`, N3 `10.10.3.1/24`, and AMFs
`155.98.36.18` / `155.98.36.20`. Update these for a new allocation. Recheck interface
names, IP/MAC addresses, PCI/DPDK mapping, and subscriber provisioning on the relevant nodes.
UE credentials in these historical lab configurations must not be reused for production.

Expected test mapping:

| UE namespace | PLMN | SST / SD | UE IP | DN IP | RFsim server address |
| --- | --- | --- | --- | --- | --- |
| ue1 | 20893 | 1 / 010203 | 10.60.100.1 | 10.10.1.2 | 10.201.1.100 |
| ue2 | 20894 | 1 / 010204 | 10.60.100.2 | 10.10.2.2 | 10.202.1.100 |

The RFsim addresses refer to the host ends of the namespace veth pairs, not namespace-local `127.0.0.1`.
Use the tracked `tools/scripts/multi-ue.sh` as a reference when recreating the namespaces.

## Build and monolithic startup

From the repository root, after installing dependencies, the following direct CMake
commands reproduce the options and target set recorded in the L6 build log. They are
documented for recovery; no new build was run while creating this backup.

```bash
source oaienv
cmake -S . -B cmake_targets/ran_build/build -DOAI_SIMU=ON -DENABLE_TELNETSRV=ON
cmake --build cmake_targets/ran_build/build --parallel 8 --target \
  nr-softmodem nr-cuup nr-uesoftmodem rfsimulator telnetsrv \
  params_libconfig coding dfts params_yaml vrtsim rf_emulator
```

Start both cores separately. In three separate terminals, enter this checkout's
`cmake_targets/ran_build/build` directory, then run the appropriate command below.
Do not run a monolithic gNB and split CU/DU simultaneously.

```bash
# gNB
sudo ./nr-softmodem \
  -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf \
  '--gNBs.[0].min_rxtxtime' 6 --rfsim

# UE1; namespace ue1 must already exist
sudo ip netns exec ue1 ./nr-uesoftmodem \
  -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf \
  -r 106 --numerology 1 --band 78 -C 3619200000 --ssb 516 --rfsim \
  --uicc0.imsi 208930000000003 --rfsimulator.serveraddr 10.201.1.100 \
  --telnetsrv --telnetsrv.listenport 9095

# UE2; namespace ue2 must already exist
sudo ip netns exec ue2 ./nr-uesoftmodem \
  -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/ue2.conf \
  -r 106 --numerology 1 --band 78 -C 3619200000 --ssb 516 --rfsim \
  --uicc0.imsi 208940000000004 --rfsimulator.serveraddr 10.202.1.100 \
  --telnetsrv --telnetsrv.listenport 9096
```

## Optional historical split setup

The two archived split configs use matching PLMN order `20893, 20894`.
The user previously verified that setup on the older `80cb69d6...` code with two UEs
and successful ping. **It was not revalidated on the final six-layer revision.**
After reviewing compatibility and environment addresses, use separate CU and DU terminals
instead of the monolithic command, from the same build directory:

```bash
# CU
sudo ./nr-softmodem -O ../../../testbed-backup/2026-10-01/cu.mocn-split.conf
# DU
sudo ./nr-softmodem -O ../../../testbed-backup/2026-10-01/du.mocn-split.conf \
  '--gNBs.[0].min_rxtxtime' 6 --rfsim
```

The UE commands above remain the reference commands. The CU/DU PLMN-order mismatch
fix is not included in this backup's source code.

## Functional checks

After provisioning subscribers and restoring networking, verify the expected UE IPs
and run each UE's test in a separate terminal:

```bash
sudo ip netns exec ue1 ip -4 addr show dev oaitun_ue1
sudo ip netns exec ue2 ip -4 addr show dev oaitun_ue1
sudo ip netns exec ue1 ping -n -I oaitun_ue1 -c 60 10.10.1.2
sudo ip netns exec ue2 ping -n -I oaitun_ue1 -c 60 10.10.2.2
```

For optional TCP downlink testing, run `iperf3 -s -p 5201` on DN1 and
`iperf3 -s -p 5202` on DN2, then run the respective clients on Node0:

```bash
sudo ip netns exec ue1 iperf3 -c 10.10.1.2 -p 5201 -B 10.60.100.1 -M 1300 -t 30 -i 1 -R
sudo ip netns exec ue2 iperf3 -c 10.10.2.2 -p 5202 -B 10.60.100.2 -M 1300 -t 30 -i 1 -R
```

`-M 1300` requests a TCP MSS of 1300 bytes; `-R` makes the DN send data to the UE.
The user reported successful monolithic attachment/IP allocation and ping for both UEs
on the final revision, plus 30-second TCP downlink receiver rates of 54.3 / 54.0 Mbit/s
with sender-reported zero retransmissions. These are historical results, not fresh
tests of this backup or a guarantee of performance on a new allocation.
