# MOCN CU/DU split：selectedPLMN-Identity 解释修复开发方案（B1）

状态：**待审核，未实施**（修订 2，2026-09-28）。本文件位于仓库外；编写与修订过程中未修改任何源码、测试、构建文件、配置或 Git 状态。修订内容见 §8。

---

## 0. 当前真实状态（2026-09-28 核实）

| 项 | 值 |
|---|---|
| 仓库 | `/users/Yuanhao/Duranta-openairinterface5g` |
| 分支 / HEAD | `pr/pre-guido/oai-2026-09-16` / `80cb69d6469880d3ccb96dbe4b734f1b0c37cb96` |
| 工作区 / index | 干净；无 stash |
| 已验证 monolithic 配置 | `targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf`（sha256 前缀 `8e0008876a62d7ef`） |
| split 测试配置 | `/users/Yuanhao/mocn-split-test/cu.mocn-split.conf`、`du.mocn-split.conf`：内容与创建时一致（与源文件 diff 未变），mtime 为 09-27 16:52（晚于创建时间，内容未变） |
| 二进制 | `cmake_targets/ran_build/build/nr-softmodem`、`nr-uesoftmodem`（09-21 17:44） |
| 规范原文 | 本机访问 ETSI/3GPP 返回反爬页面，**无法下载**；下文规范条目采用另一方核实的页码/摘录，本方未独立核对原文 |

---

## 1. 问题与链路

### 1.1 修改前（当前代码）

1. DU：`plmn_list` → `set_plmn_config()` → `info.served_plmn_list` → `get_SIB1_NR()` 生成 SIB1（一个外层 `PLMN-IdentityInfo`，内层按配置顺序）；同一数组也编码为 F1 Served PLMNs。
2. UE：从 SIB1 外层第 0 项读取 PLMN，按 IMSI 匹配得到 1-based `selectedPLMN-Identity`，放入 RRCSetupComplete。
3. DU → F1 UL RRC Message → CU `rrc_gNB_decode_dcch()` → `handle_rrcSetupComplete()` → `rrc_gNB_process_RRCSetupComplete()` → `rrc_gNB_send_NGAP_NAS_FIRST_REQ()`。
4. CU（[rrc_gNB_NGAP.c:232-242](../Duranta-openairinterface5g/openair2/RRC/NR/rrc_gNB_NGAP.c)）：
   `req->plmn = rrc->configuration.plmn[selectedPLMN_Identity - 1]` —— **使用 CU 自己的配置顺序**。
5. NGAP `select_amf()` 按 `req->plmn` 选择 AMF。

后果：CU/DU PLMN 顺序或数量不同时，UE 选择会被解释成另一个 PLMN 并送往另一个 AMF；越界时仅 `LOG_E` 返回，且已分配的 ITTI 消息和 NAS 副本泄漏，UE 悬挂。monolithic 中两侧列表来自同一配置，因此问题不可见。

### 1.2 修改后

步骤 3 在 `handle_rrcSetupComplete()` 入口增加事务检查：只有匹配本 UE 上下文中待完成 RRCSetup 事务的 SetupComplete 才被处理，且每个上下文至多一次。

步骤 4 改为：取 UE 的 PCell → 从 CU 保存的**该小区已解码 SIB1**按 1-based index 取得实际广播 PLMN → `get_serving_plmn()` 确认 CU 支持 → 成功才分配并发送 Initial UE Message；任何一步失败返回 `false`，由调用方执行“NGAP 前本地释放”，不向任何 AMF 发送。AMF 选择算法不变。

---

## 2. 规范要求 / 代码事实 / 工程选择 / 保留问题

### 2.1 规范要求（另一方核实，本方未能访问原文）
- TS 38.331 V17.15.0 p.411：`selectedPLMN-Identity` 指 UE 从 SIB1 PLMN/NPN 列表中选择的索引。
- p.526：PLMN index 跨外层 `PLMN-IdentityInfo`、内层 `plmn-IdentityList` 累计。
- p.741–742：MCC 三位；MNC 两位或三位；MCC 省略时沿用紧邻前一个 `PLMN-Identity` 的 MCC；列表首项必须带 MCC。
- TS 38.473 V17.15.0 §8.2.3.2（p.30）：NG-RAN 场景 F1 Setup 中 DU 应携带 gNB-DU System Information。
- §8.2.4.2（p.33）：DU Update 省略的配置项视为未改变，继续使用已有配置。
- §8.2.4.3（p.35）：CU 无法接受 Update 时应回复 gNB-DU CONFIGURATION UPDATE FAILURE，并给出原因。
- TS 38.331 V17.15.0 §5.1.2（p.39）：UE 回复消息中的 `rrc-TransactionIdentifier` 与触发该回复的网络消息相同。该条说明正常 UE 会回填相同事务 ID，但不规定 CU 内部如何防重复。

### 2.2 当前代码事实（本方逐项核实）
- ASN.1（`nr-rrc-17.3.0.asn1`）：`selectedPLMN-Identity INTEGER (1..maxPLMN)`；`PLMN-IdentityInfoList`、`plmn-IdentityList` 均 `SIZE (1..maxPLMN)`；`PLMN-Identity.mcc OPTIONAL -- Cond MCC`。解码器只约束每层长度，**不约束总数 ≤12**。
- 生成常量 `NR_maxPLMN = 12`（`ran_build/build/openair2/RRC/NR/MESSAGES/NR_asn_constant.h:517`）；`PLMN_LIST_MAX_SIZE`、`F1AP_MAX_NB_PLMNS` 均为 6，不可用于 SIB1。
- F1AP ASN.1：`GNB-DU-System-Information` 容器 OPTIONAL，但其中 `sIB1-message` 必选；OAI 未实现 `BPLMN-ID-Info-List`。
- CU 只在 [rrc_gNB_du.c:761-762](../Duranta-openairinterface5g/openair2/RRC/NR/rrc_gNB_du.c)（F1 Setup）和 [1065-1076](../Duranta-openairinterface5g/openair2/RRC/NR/rrc_gNB_du.c)（DU Update）写 `cell->mib/sib1`；运行时**无任何读取者**，只在 `rrc_free_cell_container()`（[rrc_cell_management.c:242-249](../Duranta-openairinterface5g/openair2/RRC/NR/rrc_cell_management.c)）释放。原始字节随 F1 消息在 [rrc_gNB.c:3715/3749](../Duranta-openairinterface5g/openair2/RRC/NR/rrc_gNB.c) 释放，运行时只能遍历已解码结构。
- `extract_sys_info()`（[rrc_gNB_du.c:476-504](../Duranta-openairinterface5g/openair2/RRC/NR/rrc_gNB_du.c)）失败时 `ASN_STRUCT_FREE` 后不置 NULL；DU Update 路径在调用前已释放 `cell->mib`（及 `cell->sib1`），失败后二者悬空，释放小区时 double free。
- DU Update（[rrc_gNB_du.c:1033-1080](../Duranta-openairinterface5g/openair2/RRC/NR/rrc_gNB_du.c)）先 `update_cell_info()` 再解码；`get_cell_by_cell_id()` 结果未判 NULL（1045-1046）。
- `update_cell_info()`（926-983）先做 PCI/cell ID 冲突检查再修改，失败时状态未变。
- `gNB-DU CONFIGURATION UPDATE FAILURE`：仅有 ITTI 类型和不完整结构体（缺 transaction_id）；无编码/解码、无 CU 发送、无 DU handler（`f1ap_handlers.c:28` 失败槽为 0）。
- `rrc_gNB_generate_RRCRelease()`（[rrc_gNB.c:3919-3940](../Duranta-openairinterface5g/openair2/RRC/NR/rrc_gNB.c)）忽略 `nr_pdcp_data_req_srb()` 返回值；PDCP 失败（无 SRB 或处理失败）时回调不执行，F1 Release Command 不发出。PDCP 回调同步执行（`nr_pdcp_oai_api.c:765`），栈上 `deliver_ue_ctxt_release_data_t` 安全。
- Release Complete（[rrc_gNB.c:2831-2848](../Duranta-openairinterface5g/openair2/RRC/NR/rrc_gNB.c)）只在 `an_release` 时清理，并发送 NGAP Release Complete。`an_release` 由 `rrc_gNB_process_NGAP_UE_CONTEXT_RELEASE_COMMAND()`（`rrc_gNB_NGAP.c:1695-1741`，第 1711 行）置位。该命令可能来自 AMF，也可能由 NGAP 在本地合成（见下一条）。这个 handler 在 DU 在线时再次调用 `rrc_gNB_generate_RRCRelease()` 并等待 Complete；DU 离线时发送 NGAP Release Complete 并立即 `rrc_remove_ue()`。
- **NGAP 收到未知 UE 的 UE Context Release Request 时不只是记录日志**：`ngap_ue_context_release_req()`（`ngap_gNB_context_management_procedures.c:126-135`）找不到 NGAP UE 上下文时，会经 ITTI 向 RRC 发送一条本地合成的 `NGAP_UE_CONTEXT_RELEASE_COMMAND`（132-134 行），异步进入上一条的 handler。因此，RRC 为没有 NG 上下文的 UE 发出 Release Request，会异步把该 UE 置为 `an_release` 并触发第二次释放。相比之下，Uplink NAS 对未知 UE 只记录日志并返回（`ngap_gNB_nas_procedures.c:373-377`），不会回送消息。
- `rrc_gNB_send_NGAP_UE_CONTEXT_RELEASE_REQ()`（`rrc_gNB_NGAP.c:1407`）目前的调用方：
  - DU 发起的 Release Request（`rrc_gNB.c:2799`）；
  - 重建回退时释放旧上下文（1707-1708）；
  - F1 断链（`rrc_gNB_du.c:900`）；
  - E1/CU-UP 丢失（`rrc_gNB_cuup.c:209`，发送后立即 `rrc_remove_ue()`）；
  - N2 切换失败（`rrc_gNB_mobility.c:532`）；
  - SRB2-only 清理（`rrc_gNB_NGAP.c:1909`）。
- `rrc_remove_ue()`（[rrc_gNB.c:2819-2828](../Duranta-openairinterface5g/openair2/RRC/NR/rrc_gNB.c)）= `nr_pdcp_remove_UE()` + `rrc_delete_ue_data()` + `rrc_gNB_remove_ue_context()`（后者在 `rrc_gNB_UE_context.c:145-148` 移出 UE 树、释放 UID、`cu_remove_f1_ue_data()`）。
- `cu_get_f1_ue_data()`（`f1ap_ids.c:82-89`）直接解引用哈希结果，**映射不存在时会崩溃**；必须先 `cu_exists_f1_ue_data()`。
- `RETURN_IF_INVALID_ASSOC_ID`（`rrc_gNB_du.h:39-45`）内部为裸 `return;`，只能用于 void 函数；判断 `assoc_id == 0`（DU 离线），`-1` 为 monolithic 合法值。
- DU 处理 Release Command：无 RRC 容器时立即释放 UE 并触发 Release Complete；DU 不认识该 UE 时也回 Release Complete（`mac_rrc_dl_handler.c:977-994`）。这是代码支持的回复，**不保证**断链/传输失败时送达。
- monolithic：Release Command 直接调用 MAC（`mac_rrc_dl_direct.c:69-72`），Release Complete 经 ITTI 回 RRC（`mac_rrc_ul_direct.c:95-100`），不会同步重入 RRC。
- F1 断链 `invalidate_du_connections()`（[rrc_gNB_du.c:875-914](../Duranta-openairinterface5g/openair2/RRC/NR/rrc_gNB_du.c)）：
  - SA 下先把映射的 `du_assoc_id` 置 0，再发 NGAP UE Context Release Request。对没有 NG 上下文的 UE，NGAP 合成的释放命令到达后，handler 因为 `du_assoc_id == 0` 走 DU 离线分支：发送没有对端上下文的 NGAP Release Complete，然后 `rrc_remove_ue()`。也就是说，NAS 前的 UE 在 F1 断链时**会被清理，不会泄漏**。
  - 非 SA 放入待删列表，遍历结束后调用 `rrc_remove_nsa_user_context()`。
- UE 上下文由 `calloc` 分配（`rrc_gNB_UE_context.c:52`），新增 bool 初值为 false。
- 所有 UL-DCCH 经唯一入口 `rrc_gNB_decode_dcch()`（2268）；分发返回后只释放 `ul_dcch_msg` 并返回（2382-2383），不再访问 UE。
- 事务 ID：
  - `rrc_gNB_get_next_transaction_identifier()`（`rrc_gNB.c:422-430`）是一个全局计数器，对 4 取模；每个 UE 上下文各有 `xids[4]`。
  - `RRC_SETUP` 只在 `rrc_gNB_generate_RRCSetup()` 的第 624 行写入。这个函数只有两个调用点，而且都针对新建的上下文：`rrc_handle_RRCSetupRequest()` 中新建的上下文（1422），以及重建回退中新建的上下文（1713-1716，旧上下文另行请求 NGAP 释放）。因此每个上下文中 `RRC_SETUP` 至多出现一次。
  - `handle_rrcSetupComplete()`（2111-2114）目前不做检查，直接把事务清为 `RRC_ACTION_NONE`。
  - OAI UE 会回填 RRCSetup 的事务 ID（`rrc_UE.c:2449`，编码见 `asn1_msg.c:833`）。
  - `NGAP_NAS_FIRST_REQ` 只在 `rrc_gNB_NGAP.c:217` 产生；N2 切换目标侧的上下文不经过 RRCSetup。

### 2.3 本次工程选择
- **B1**：运行时从 PCell 已解码的 SIB1 取 PLMN，不增加派生缓存。只有一个 helper `nr_rrc_sib1_get_plmn()`，每次查找都遍历完整 PLMN 列表，只校验索引解析所需的字段（结构、指针、长度、数字、MCC 继承、总数）。这**不是完整的 SIB1 语义验证**。任何失败都不会修改输出。
- F1 Setup 不新增 PLMN 校验，也不新增拒绝分支。能解码、但 PLMN 列表通不过上述检查的 SIB1 照常保存：运行时查找会失败，新 UE 被本地释放，效果等同于失效，同时不改变 F1 Setup 的行为。
- DU Update：能解码就替换为新结构；**无法解码**时拒绝本次修改，并让该小区的旧 SIB1 失效。这是工程选择，不是规范规定：解码失败时 CU 无法知道 DU 实际广播的内容，继续按旧顺序解释可能把 UE 送错 PLMN。`mac_rrc_dl_handler.c:158-176` 只说明 OAI DU 是先更新自己的 SIB1、再发 Update，并不能证明 PLMN 会被重新排序。
- SetupComplete 必须匹配本上下文中待完成的 `RRC_SETUP` 事务才会被处理，通过检查后立即清除，保证至多处理一次。
- NAS 首条消息失败时，走新增的“NGAP 前本地释放”（`local_release`），不复用 `an_release`。正在本地释放的 UE，不会再因 DU Release Request 而被绕进 NGAP 释放流程。
- DU Update 在 SIB1 解码失败时不回 Acknowledge。这是最小方案中的明确取舍，**不符合** 38.473 §8.2.4.3（应回复 Failure 并给出原因），见 §7。

### 2.4 明确保留的已有问题
见 §7。

---

## 3. 文件与函数改动（7 个生产文件 + 1 个测试文件）

行号均为 HEAD `80cb69d6` 的当前行号。

### 3.1 `openair2/RRC/NR/rrc_cell_management.h`
- 新增一个声明（`NR_SIB1_t` 已通过 `nr_rrc_defs.h:482` 可见）：
  ```c
  bool nr_rrc_sib1_get_plmn(const NR_SIB1_t *sib1, long selected_plmn_identity, plmn_id_t *out);
  ```
  不再提供单独的 `plmn_list_valid` API。
- 必要性：CU 当前无任何 SIB1 PLMN 解析代码（UE 侧 `plmn_from_asn1()` 为 `rrc_UE.c` 内 static，不可复用）。

### 3.2 `openair2/RRC/NR/rrc_cell_management.c`
- 新增 `#include "NR_asn_constant.h"`（当前未包含；`rrc_gNB_du.c:30` 已有先例）。
- 新增 static 遍历函数 `sib1_walk_plmns()` 和公开函数 `nr_rrc_sib1_get_plmn()`（见 §4.1）。
- 放在此文件的理由：已包含 `NR_SIB1.h`，已被 `rrc_gNB_NGAP.c`/`rrc_gNB_du.c` 包含，且被现有单测链接。

### 3.3 `openair2/RRC/NR/rrc_gNB_NGAP.h`
- 第 23 行：`void rrc_gNB_send_NGAP_NAS_FIRST_REQ(...)` → `bool rrc_gNB_send_NGAP_NAS_FIRST_REQ(...)`。
- 必要性：显式返回失败，不用标志副作用。唯一调用点 `rrc_gNB.c:688`（已 grep 确认）。

### 3.4 `openair2/RRC/NR/rrc_gNB_NGAP.c`
- `rrc_gNB_send_NGAP_NAS_FIRST_REQ()`（215-278）：
  - 修改前：先 `itti_alloc_new_message` + `create_byte_array(nas)`，再按 CU 配置索引；越界 `return` 泄漏；`DevAssert(cell)`。
  - 修改后：先 PCell → SIB1 → `get_serving_plmn()`，全部成功再分配；`req->plmn` 与 `UE->serving_plmn` 取 `get_serving_plmn()` 返回的 CU 配置项（保留 MNC 位数）；无 PCell 时返回 false 替代 `DevAssert`；其余字段（cause、5G-S-TMSI、registeredAMF/GUAMI、`nr_cell_id`、日志）原样保留。
- `get_serving_plmn()`（101-113）复用，不改。它比较 mcc、mnc、mnc_digit_length，未命中时 `LOG_W`。

### 3.5 `openair2/RRC/NR/nr_rrc_defs.h`
- `gNB_RRC_UE_t` 中 `an_release`（251）旁新增：
  ```c
  bool local_release; // CU-initiated release before any NGAP context exists
  ```

### 3.6 `openair2/RRC/NR/rrc_gNB.c`
| 函数（当前行） | 改动 |
|---|---|
| `handle_rrcSetupComplete()`（2111-2114） | 取得 xid 后，先检查 `xid < 4 && UE->xids[xid] == RRC_SETUP`；不满足就记录日志并返回；满足就清为 `RRC_ACTION_NONE`，然后继续原逻辑（§4.2）。**必需** |
| `rrc_gNB_process_RRCSetupComplete()`（682-689） | 检查 NAS 首条消息的返回值，为 false 时调用本地释放 |
| `rrc_gNB_generate_RRCRelease()`（3919） | 函数体抽成 `static bool rrc_gNB_send_RRCRelease()`；公开的 `void` wrapper 调用它并忽略返回值，AMF 发起的释放行为、日志和 E2 `#ifdef` 都保持不变 |
| 新增 `static void rrc_gNB_release_ue_before_ngap()` | 本地释放（§4.3） |
| `rrc_gNB_decode_dcch()`（2268） | 在 SRB 检查（2276-2279）之后、`uper_decode` 之前：`local_release` 则记录并丢弃。一行代码，不是正确性所必需（§4.3） |
| `rrc_CU_process_ue_context_release_request()`（2735） | 找到上下文之后：`local_release` 则记录并返回，不转发给 NGAP。**必需**（§4.3） |
| `rrc_CU_process_ue_context_release_complete()`（2831） | `an_release` 分支不变；新增 `else if (UE->local_release) rrc_remove_ue()`，不发 NGAP。**必需** |

`rrc_gNB_send_RRCRelease()` 不得使用 `RETURN_IF_INVALID_ASSOC_ID`（裸 `return;`），改为显式检查并 `return false`；且先 `cu_exists_f1_ue_data()` 再 `cu_get_f1_ue_data()`。

### 3.7 `openair2/RRC/NR/rrc_gNB_du.c`
| 函数（当前行） | 改动 |
|---|---|
| `extract_sys_info()`（476-504） | 每处 `ASN_STRUCT_FREE(*mib/*sib1)` 之后，把 `*mib` / `*sib1` 置 NULL |
| `rrc_gNB_process_f1_du_configuration_update()`（1033-1080） | 补上 cell 的 NULL 检查；先解码到局部变量，再调用 `update_cell_info()`，最后提交；只有解码失败才让旧 SIB1 失效（§4.4） |

以下两个函数**不修改**：
- `rrc_gNB_process_f1_setup_req()`：理由见 §2.3。
- `invalidate_du_connections()`：见 §4.5。F1 断链时，现有 SA 路径会通过 NGAP 合成的释放命令清理 NAS 前的 UE（包括 `local_release` UE）。修订 1 中“不改就会泄漏”的判断是错的。

### 3.8 `openair2/RRC/NR/tests/rrc_cell_management_test.c`（444 行）
- 新增 `test_sib1_plmn_lookup()`，在 `main()`（433-444）中调用。
- 新增 `#include "common/utils/oai_asn1.h"`（`asn1cCalloc`/`asn1cSequenceAdd`）以及 SIB1 相关头文件。
- **不需要 CMake 修改**：`rrc_cell_management_test` 链接 `rrc_cell_management`（`tests/CMakeLists.txt:14-23`），而后者 `PUBLIC` 链接 `asn1_nr_rrc`（`RRC/NR/CMakeLists.txt:6-8`），`asn_DEF_NR_SIB1` 可用。若实际编译发现缺符号，再说明原因后调整。

---

## 4. API 与关键伪代码

### 4.1 SIB1 PLMN helper

```c
/* returns total PLMN count (1..NR_maxPLMN) if the PLMN list passes the checks below, -1 otherwise;
 * fills *out only when 1 <= target <= count */
static int sib1_walk_plmns(const NR_SIB1_t *sib1, long target, plmn_id_t *out)
{
  if (!sib1) return -1;
  const list = &sib1->cellAccessRelatedInfo.plmn_IdentityInfoList.list;
  if (!list->array || list->count < 1 || list->count > NR_maxPLMN) return -1;
  int total = 0;
  for (o = 0; o < list->count; o++) {
    info = list->array[o];               if (!info) return -1;
    inner = &info->plmn_IdentityList.list;
    if (!inner->array || inner->count < 1 || inner->count > NR_maxPLMN) return -1;
    bool have_mcc = false; uint16_t mcc = 0;          // MCC inheritance scope = this inner list only
    for (i = 0; i < inner->count; i++) {
      p = inner->array[i];               if (!p) return -1;
      if (p->mcc) {
        if (!digits_ok(p->mcc, 3)) return -1;         // count==3, non-NULL, 0..9
        mcc = digits_value(p->mcc); have_mcc = true;
      } else if (!have_mcc) return -1;                // first entry of each inner list needs MCC
      if (!digits_ok(&p->mnc, 2) && !digits_ok(&p->mnc, 3)) return -1;
      if (++total > NR_maxPLMN) return -1;           // total across outer entries
      if (out && total == target)
        *out = (plmn_id_t){.mcc = mcc, .mnc = digits_value(&p->mnc), .mnc_digit_length = p->mnc.list.count};
    }
  }
  return total;
}

bool nr_rrc_sib1_get_plmn(const NR_SIB1_t *sib1, long sel, plmn_id_t *out)
{
  if (!out || sel < 1 || sel > NR_maxPLMN) return false;
  plmn_id_t tmp = {0};
  int n = sib1_walk_plmns(sib1, sel, &tmp);
  if (n < 0 || sel > n) return false;               // *out untouched on failure
  *out = tmp;
  return true;
}
```
- 索引：输入保持 1-based；外层 → 内层顺序累计；不截断、不压缩、不跳过槽位。
- 始终遍历完整列表：目标之后出现非法项也按失败处理（广播格式异常时保守拒绝）。
- 校验范围仅限索引解析所需字段（§2.3），**不是完整 SIB1 语义验证**。
- 有效条件：整个 PLMN 列表通过上述检查，且 `1 ≤ sel ≤ total`；只在成功时写 `*out`。
- MCC 继承只在同一内层 `plmn-IdentityList` 内有效，不跨外层条目；每个内层列表首项必须带 MCC。
- 同一 MNC 数值、不同位数（93 vs 093）以 `mnc_digit_length` 区分；后续 `get_serving_plmn()` 按位数比较。

### 4.2 SetupComplete 事务检查

```c
/* handle_rrcSetupComplete(), replaces 2113-2114 */
uint8_t xid = setup_complete->rrc_TransactionIdentifier;
if (xid >= NR_RRC_TRANSACTION_IDENTIFIER_NUMBER || UE->xids[xid] != RRC_SETUP) {
  LOG_W(NR_RRC, "UE %d: RRCSetupComplete xid %d does not match a pending RRCSetup, ignoring\n", UE->rrc_ue_id, xid);
  return;
}
UE->xids[xid] = RRC_ACTION_NONE;   // consumed: at most once per UE context
/* rest unchanged (criticalExtensions check, 5G-S-TMSI handling, E2 signal, process) */
```
顺序：先检查，通过后立即清除，再处理消息体；即使消息体格式错误，事务也已被消耗，保证每个上下文至多进入一次后续流程。ASN.1 已把事务 ID 限制在 0–3，这里的范围判断只防数组越界。

必要性：没有这个检查时，首条 SetupComplete 成功送 NGAP 后，再来一条 index 无效的 SetupComplete 会进入新的失败分支，把已有或正在建立 NGAP 上下文的 UE 当作 NAS 前 UE 本地释放。修改前同样的输入只在越界时记录日志、UE 不受影响，所以这是**本次新引入的回归，必须在本次修复**；“重复 SetupComplete 无防护”的已有缺陷也随之消除。

| 输入 | 修改前 | 修改后 |
|---|---|---|
| 首条、事务 ID 匹配 | 处理 | 处理 |
| 重复或迟到（事务已消耗） | 再次送 NGAP，或越界时只记录日志 | 丢弃 |
| 事务 ID 与 RRCSetup 不一致 | 处理 | 丢弃（行为变化；对符合 §5.1.2 的 UE 无影响） |

**“调用本地释放时尚未向 NGAP 发出首条 NAS”的保证**：
1. `NGAP_NAS_FIRST_REQ` 只有 `rrc_gNB_send_NGAP_NAS_FIRST_REQ()` 一个发送点；
2. 它只在事务检查通过后被调用，而每个上下文中 `RRC_SETUP` 至多设置一次、只出现在新建上下文里，所以每个上下文最多调用一次；
3. 它只有在分配消息之前失败才返回 false，返回 false 就意味着没有发出任何消息。

N2 切换目标侧的上下文永远没有 `RRC_SETUP`；重建回退使用新建上下文。

**验证**：
- 代码审查：grep 确认 `RRC_SETUP` 写入点和 `NGAP_NAS_FIRST_REQ` 发送点各只有一处；
- E2E：正常接入时每个 UE 仍有 `Selected PLMN in the NG Initial UE Message`，且不出现新增的 `LOG_W`；
- 重复或事务 ID 不匹配的 SetupComplete 无法用 OAI UE 触发，只能靠代码审查。

### 4.2b NAS 首条消息

```c
bool rrc_gNB_send_NGAP_NAS_FIRST_REQ(rrc, UE, ies)
{
  nr_rrc_cell_container_t *cell = rrc_get_pcell_for_ue(rrc, UE);
  if (!cell) { LOG_E(...no PCell...); return false; }
  plmn_id_t sel;
  if (!cell->sib1 || !nr_rrc_sib1_get_plmn(cell->sib1, ies->selectedPLMN_Identity, &sel)) {
    LOG_E(...cannot resolve selectedPLMN-Identity %ld in cell %lu SIB1...); return false;
  }
  const plmn_id_t *served = get_serving_plmn(rrc, &sel);
  if (!served) { LOG_E(...PLMN not served by CU...); return false; }
  MessageDef *message_p = itti_alloc_new_message(...);      // only now
  ... req->nas_pdu = create_byte_array(...);
  req->plmn = *served; UE->serving_plmn = *served; req->nr_cell_id = cell->info.cell_id;
  ... 5G-S-TMSI / registeredAMF / LOG_I "Selected PLMN in the NG Initial UE Message" unchanged ...
  itti_send_msg_to_task(TASK_NGAP, ...);
  return true;
}
```
调用方（唯一，`rrc_gNB.c:682-689`）：
```c
static void rrc_gNB_process_RRCSetupComplete(rrc, UE, ies)
{
  UE->Srb[1].Active = 1;
  UE->Srb[2].Active = 0;
  if (!rrc_gNB_send_NGAP_NAS_FIRST_REQ(rrc, UE, ies))
    rrc_gNB_release_ue_before_ngap(rrc, UE);   // may free UE: nothing below touches UE
}
```
删除后调用链安全性：`handle_rrcSetupComplete()` 在 2179 调用后直接 `return`；`rrc_gNB_decode_dcch()` 在 2318 `break` 后只执行 2382 `ASN_STRUCT_FREE(ul_dcch_msg)` 并返回；E2 `signal_ue_id()` 在调用前执行。

### 4.3 NGAP 前本地释放

```c
static bool rrc_gNB_send_RRCRelease(gNB_RRC_INST *rrc, gNB_RRC_UE_t *UE)
{
  build RRCRelease buffer (unchanged);
  LOG_UE_DL_EVENT(UE, "Send RRC Release\n");
  if (!cu_exists_f1_ue_data(UE->rrc_ue_id)) return false;
  f1_ue_data_t d = cu_get_f1_ue_data(UE->rrc_ue_id);
  if (d.du_assoc_id == 0) { LOG_E(...DU offline...); return false; }   // -1 (monolithic) is valid
  f1ap_ue_context_rel_cmd_t cmd = {..., .srb_id = &srbid};
  deliver_ue_ctxt_release_data_t data = {...};
  bool ok = nr_pdcp_data_req_srb(..., rrc_deliver_ue_ctxt_release_cmd, &data);
#ifdef E2_AGENT
  E2_AGENT_SIGNAL_DL_DCCH_RRC_MSG(...);   // same position as today
#endif
  return ok;
}

void rrc_gNB_generate_RRCRelease(gNB_RRC_INST *rrc, gNB_RRC_UE_t *UE)
{
  (void)rrc_gNB_send_RRCRelease(rrc, UE);   // AMF-initiated path: behaviour unchanged
}

static void rrc_gNB_release_ue_before_ngap(gNB_RRC_INST *rrc, gNB_RRC_UE_t *UE)
{
  UE->local_release = true;
  rrc_gNB_ue_context_t *ctx = rrc_gNB_get_ue_context(rrc, UE->rrc_ue_id);
  DevAssert(ctx);
  if (!cu_exists_f1_ue_data(UE->rrc_ue_id) || cu_get_f1_ue_data(UE->rrc_ue_id).du_assoc_id == 0) {
    rrc_remove_ue(rrc, ctx);                         // deleter: this function
    return;
  }
  if (rrc_gNB_send_RRCRelease(rrc, UE))
    return;                                          // deleter: Release Complete / F1 loss
  f1_ue_data_t d = cu_get_f1_ue_data(UE->rrc_ue_id);
  f1ap_ue_context_rel_cmd_t cmd = {.gNB_CU_ue_id = UE->rrc_ue_id, .gNB_DU_ue_id = d.secondary_ue,
                                   .cause = F1AP_CAUSE_RADIO_NETWORK, .cause_value = 10};  // no RRC container
  rrc->mac_rrc.ue_context_release_command(d.du_assoc_id, &cmd);
  // deleter: Release Complete / F1 loss
}
```
与原有 AMF 路径的差异：原公开 `void` 接口在 DU 离线时 `LOG_E` 并返回；新内部函数除此之外还在映射不存在时返回 false（原接口在此情况下会解引用空指针，但现有唯一调用者 `rrc_gNB_NGAP.c:1729` 已先检查存在性，故 AMF 路径行为不变）。

其余 `rrc_gNB.c` 检查：
```c
/* rrc_gNB_decode_dcch(), after srb_id check */
if (UE->local_release) { LOG_W(... dropping UL-DCCH, local release pending ...); return 0; }

/* rrc_CU_process_ue_context_release_request(), after context lookup, before any forwarding */
if (UE->local_release) { LOG_I(... local release already commanded, not forwarding to NGAP ...); return; }

/* rrc_CU_process_ue_context_release_complete() */
if (UE->an_release) { ...unchanged... }
else if (UE->local_release) rrc_remove_ue(RC.nrrrc[0], ue_context_p);
```

**Release Request 检查必需（完整链路）**：
1. 不加检查时，`local_release` UE 收到 DU Release Request 会调用 `rrc_gNB_send_NGAP_UE_CONTEXT_RELEASE_REQ()`（2799）。
2. NGAP 找不到该 UE 的上下文，异步合成 `NGAP_UE_CONTEXT_RELEASE_COMMAND` 发回 RRC（`ngap_gNB_context_management_procedures.c:132-134`）。
3. RRC 处理该命令（`rrc_gNB_NGAP.c:1695-1741`）：置位 `an_release`，DU 在线时再发一次 RRCRelease 和 F1 Release Command。
4. 后果：
   - Release Complete 走 `an_release` 分支，向不存在的 NGAP 上下文发 Release Complete；
   - 出现两次 F1 Release Command、两次 Complete；
   - 若合成命令在 UE 已被删除、其 CU UE ID 被新 UE 复用之后才到达，会对**新 UE** 置位 `an_release` 并将其释放。

该检查在源头阻止正在本地释放的 UE 被绕回 NGAP 释放流程，不需要改 NGAP 模块。

**UL-DCCH 入口检查**：有了事务检查后，重复 SetupComplete 已被挡住；UL NAS 到 NGAP 后对未知 UE 只记录日志、不回送消息（`ngap_gNB_nas_procedures.c:373-377`）。因此这一行不是正确性所必需，保留它是为了让正在本地释放的 UE 不再进入任何 UL-DCCH 处理。

### 4.4 DU Configuration Update（cell_to_modify 分支）

```c
/* existing PLMN any-match check unchanged (returns without response, pre-existing) */
nr_rrc_cell_container_t *cell = get_cell_by_cell_id(&rrc->cells, old_nci);
if (!cell) { LOG_W(...); return; }                                   // new
/* existing assoc_id check unchanged */
const f1ap_gnb_du_system_info_t *si = conf_up->cell_to_modify[0].sys_info;
bool has_si = si && si->mib && !(si->sib1 == NULL && IS_SA_MODE(...)); // existing condition
NR_MIB_t *new_mib = NULL; NR_SIB1_t *new_sib1 = NULL;
if (has_si) {
  if (!extract_sys_info(si, &new_mib, &new_sib1)) {      // extract NULLs *mib/*sib1 on failure
    ASN_STRUCT_FREE(asn_DEF_NR_SIB1, cell->sib1);
    cell->sib1 = NULL;                        // engineering choice: new UEs on this cell refused
    LOG_E(... "gNB-DU Configuration Update not accepted: undecodable system information; "
              "GNB-DU CONFIGURATION UPDATE FAILURE not implemented, no response sent; "
              "new UEs on cell %lu refused until a decodable SIB1 is received" ...);
    return;
  }
}
cell = update_cell_info(rrc, old_nci, new_ci);
if (!cell) {                                   // PCI / cell-ID conflict: keep old state entirely
  ASN_STRUCT_FREE(asn_DEF_NR_MIB, new_mib);
  ASN_STRUCT_FREE(asn_DEF_NR_SIB1, new_sib1);
  LOG_W(... existing message ...);
  return;
}
if (has_si) {
  ASN_STRUCT_FREE(asn_DEF_NR_MIB, cell->mib);  cell->mib = new_mib;
  if (new_sib1) { ASN_STRUCT_FREE(asn_DEF_NR_SIB1, cell->sib1); cell->sib1 = new_sib1; }
  LOG_I(... existing "update system information" ...);
}
/* no system information: keep existing MIB/SIB1 (38.473 §8.2.4.2) */
```
- 所有权：局部 `new_mib/new_sib1` 在提交前归本函数；提交后归 `cell`，旧对象立即释放；`cell` 的对象最终由 `rrc_free_cell_container()` 释放。`extract_sys_info()` 失败时自行释放并把输出置 NULL，调用方不再重复释放。`ASN_STRUCT_FREE` 对 NULL 安全（现有 `rrc_free_cell_container()` 已依赖这一点）。
- 能解码但 PLMN 列表通不过检查的 SIB1 照常提交（它就是 DU 实际广播的内容）；该小区的新 UE 会在运行时查找失败并被本地释放。
- SIB1 解码失败时 `cell->mib` 保持原值（没有运行时读取者，不影响路由）。
- 失效只影响新的 RRCSetupComplete，前提是 §4.2 的事务检查：已连接的 UE 不会再进入 SIB1 解析；切换和重建流程也不读 SIB1。没有事务检查时，这一点不成立。
- 恢复条件：后续收到带可解码系统信息的 DU Update 并成功提交，或该 DU 重新 F1 Setup。
- 行为变化：现有代码在 SIB1 解码失败后记录日志、继续执行并发送 Acknowledge（1095-1096）。新方案在这种情况下 `return`，不发送 Acknowledge。这**不是**标准的 Failure 回复（见 §7）。代码观察到的只有：OAI DU 收到 Ack 只记录一行日志（`mac_rrc_dl_handler.c:205-209`），DU 侧未搜到等待 Ack 的定时器。这些观察**不能代替实际的故障测试**。
- 与最初版本的区别：PLMN 校验失败不再触发失效。

### 4.5 F1 断链：不修改

`invalidate_du_connections()` 在 SA 下先把映射的 `du_assoc_id` 置 0，再调用 `rrc_gNB_send_NGAP_UE_CONTEXT_RELEASE_REQ()`（`rrc_gNB_du.c:893-902`）。对没有 NG 上下文的 UE（包括 `local_release` UE）：
1. NGAP 合成一条释放命令；
2. handler 置位 `an_release`，因为 `du_assoc_id == 0` 走 DU 离线分支；
3. 向不存在的 NGAP 上下文发 Release Complete（NGAP 记录错误日志后返回），然后 `rrc_remove_ue()`。

也就是说，正常的断链路径会通过现有的异步释放命令清理这类 UE，因此本次不改源码。修订 1 中“待删列表按标志分派”的改动已撤销，NSA 清理不受影响。

这里**不保证**原来的 Release Complete 一定不会再到达，也不保证 UE 只被删除一次。以下情况仍属于已记录的局限（§4.6、§7）：
- 断链前已在途的 F1 消息，或在合成命令到达之前被处理的消息；
- 合成命令是异步到达的，它和其他消息的先后顺序不固定；
- CU UE ID 被复用后，迟到的消息可能匹配到新 UE。

这些局限与所有已有的 NAS 前 UE 相同，本次不为此增加源码改动。

### 4.6 删除责任与状态

| 状态 | 删除者 | 时机 |
|---|---|---|
| NAS 首条失败，无 F1 映射或 `du_assoc_id == 0` | `rrc_gNB_release_ue_before_ngap()` | 立即 |
| RRCRelease 已发出（PDCP 成功） | `rrc_CU_process_ue_context_release_complete()` 的 `local_release` 分支 | 收到 Complete |
| PDCP 失败，已发不带容器的 Release Command | 同上 | 收到 Complete |
| 等待期间 F1 断链 | 现有路径：NGAP 合成命令 → DU 离线分支 `rrc_remove_ue()`（§4.5） | 合成命令到达时（异步） |
| 等待期间 DU 发 Release Request | 不删除、不转发 NGAP | 继续等 Complete |
| 等待期间 E1/CU-UP 丢失 | 现有 `rrc_gNB_cuup.c:209-210`：发 NGAP 请求后立即 `rrc_remove_ue()` | 立即 |
| 既无 Complete 也无断链 | 无（已知局限，没有定时器） | — |

**等待 Complete 的收益**：只要 DU 仍持有该 UE，它的 CU UE ID 就一直被占用；DU 发来的该 UE 的消息（Release Request、UL RRC、Complete）都会落到这个 `local_release` 上下文上，并按上表处理。

**剩余局限**：CU UE ID 由线性分配器回收并优先复用（`rrc_gNB_UE_context.c:146`）。上下文被删除之后（不论是因为 Complete、断链还是 E1 丢失）再到达的消息，包括迟到的 Complete、DU 消息，以及 E1 丢失路径引出的异步合成命令，**不保证找不到上下文**，可能匹配到复用了该 ID 的新 UE。这是已有释放路径共同的局限，本次不修。

已考虑但未采用的更小方案：像 `rrc_gNB_generate_RRCReject()`（646-676）那样发出后立即 `rrc_remove_ue()`。未采用原因：ID 会被立即复用，DU 对旧 UE 的迟到消息更容易匹配到新 UE。

---

## 5. 保留的现有行为

- **monolithic**：SIB1 同样经 `RC_read_F1Setup()` 和直连路径的深拷贝进入 `cell->sib1`（`gnb_config.c:1124-1127`、`f1ap_interface_management.c:936-955`），因此 B1 同样生效；同序时结果与现状相同。Release Complete 经 ITTI 返回，不同步重入。
- **NSA / phy-test / do-ra**：`IS_SA_MODE` 为假，不走 RRCSetupComplete 的 NGAP 路径；F1 Setup 和 F1 断链处理都没有改动。
- **AMF 发起的释放**：`rrc_gNB_generate_RRCRelease()` 的签名、行为和 E2 信号不变；`an_release` 分支不变。
- **NGAP / AMF 选择**：`select_amf()`、nnsf 以及 NGAP 模块都没有改动，只是输入的 PLMN 变成了 UE 实际选择的 PLMN。
- **切换、重建、UL NAS**：没有改动。N2 切换目标侧的上下文不带 `RRC_SETUP`；切换回滚只清理 `RRC_DEDICATED_RECONF`（`rrc_gNB.c:1608-1611`）；UL-DCCH 新增的检查只对 `local_release` UE 起作用。
- **正常接入的事务检查**：OAI UE 会回填事务 ID（`rrc_UE.c:2449`），所以 monolithic 和 split 的正常接入都不受影响。

---

## 6. 测试方案

### 阶段 A：文档审查
由用户和另一方对照真实代码复核本文。通过前不实施。

### 阶段 B：未修改代码的 split 基线（用户执行）

配置：`/users/Yuanhao/mocn-split-test/cu.mocn-split.conf`、`du.mocn-split.conf`（两侧均为 20893 / 0x010203、20894 / 0x010204）。以下命令由用户手动执行，本方不操作测试环境。

```bash
# 0) 准备：在原终端用 Ctrl-C 停止旧 monolithic gNB 和两个 UE；确认没有残留进程
pgrep -a -f 'nr-softmodem|nr-uesoftmodem' || echo none
ip netns list                      # 应有 ue1、ue2；已存在就不要再执行 multi-ue.sh -c
# 两个 Core 先起来；记录版本
cd /users/Yuanhao/Duranta-openairinterface5g && git rev-parse HEAD
ls -l --time-style=full-iso cmake_targets/ran_build/build/nr-softmodem cmake_targets/ran_build/build/nr-uesoftmodem

# 终端 1：CU（不需要 --rfsim）
cd /users/Yuanhao/Duranta-openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O /users/Yuanhao/mocn-split-test/cu.mocn-split.conf

# 终端 2：DU（等 CU 与两个 AMF 完成 NG Setup 后）
cd /users/Yuanhao/Duranta-openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O /users/Yuanhao/mocn-split-test/du.mocn-split.conf --gNBs.[0].min_rxtxtime 6 --rfsim

# 终端 3：UE1
cd /users/Yuanhao/Duranta-openairinterface5g/tools/scripts && sudo ./multi-ue.sh -o1
# 在进入的 ue1 shell 中：
cd /users/Yuanhao/Duranta-openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-uesoftmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf -r 106 --numerology 1 --band 78 -C 3619200000 --ssb 516 --rfsim --uicc0.imsi 208930000000003 --rfsimulator.serveraddr 10.201.1.100 --telnetsrv --telnetsrv.listenport 9095

# 终端 4：UE2
cd /users/Yuanhao/Duranta-openairinterface5g/tools/scripts && sudo ./multi-ue.sh -o2
# 在进入的 ue2 shell 中：
cd /users/Yuanhao/Duranta-openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-uesoftmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/ue2.conf -r 106 --numerology 1 --band 78 -C 3619200000 --ssb 516 --rfsim --uicc0.imsi 208940000000004 --rfsimulator.serveraddr 10.202.1.100 --telnetsrv --telnetsrv.listenport 9096
```
- RFsim 地址依据：`multi-ue.sh` 为 ue1/ue2 分配 10.201.1.1 / 10.202.1.2，根 namespace 侧为 10.201.1.100 / 10.202.1.100；DU 的 RFsim 服务端在根 namespace 监听 `[::]:4043`。namespace 内 `127.0.0.1` 是该 namespace 自身的回环，不可用。
- 预期：CU `Accepting DU`、`Added cell 12345678`、无 `PLMN mismatch`；DU `F1-C DU IPaddr 127.0.0.4, connect to F1-C CU 127.0.0.3`；两个 AMF NG Setup 成功；每个 UE：CU `Selected PLMN in the NG Initial UE Message: MCC=208 MNC=93/94` 与 `Selected AMF '<name>' (assoc_id N) through selected PLMN`；两个 UE 取得预期 IP 并同时 ping 各自 DN。

**修复前反序复现**（另存副本，不改基线文件）：只把 CU 副本的 `plmn_list` 两项互换，**slice 随 PLMN 一起移动**：
```
plmn_list = (
  { mcc = 208; mnc = 94; mnc_length = 2; snssaiList = ({ sst = 1; sd = 0x010204; }) },
  { mcc = 208; mnc = 93; mnc_length = 2; snssaiList = ({ sst = 1; sd = 0x010203; }) }
);
```
一次只起一个 UE。预期（代码推导）：UE1 的 CU 日志为 `MNC=94`，被选中的是 Core2 的 AMF；UE2 相反。要记录实际选中的 PLMN/AMF，不能只凭 ping 失败下结论。

### 阶段 C：实现及单元测试（获批后）
`rrc_cell_management_test.c` 新增用例（手工构造 `NR_SIB1_t`，结束时 `ASN_STRUCT_FREE`），全部通过 `nr_rrc_sib1_get_plmn()` 覆盖：
1. 单外层，取第 1 项、中间项、最后一项；
2. 两个及以上外层，跨外层累计索引；
3. 合法的 MCC 继承；第二个内层列表首项缺 MCC → 任何 index 都失败；
4. MNC 两位与三位；同数值不同位数（93 / 093）得到不同的 `mnc_digit_length`；
5. index 为 0、负数、`total+1`、`NR_maxPLMN+1`；
6. `sib1 == NULL`、外层为空、内层为空、NULL 元素、MCC 位数不是 3、MNC 位数不在 {2,3}、数字大于 9、NULL 数字指针；
7. 7–12 个合法 PLMN（超过 6 个）；
8. 总数 13（如 7 + 6）→ 任何 index 都失败；
9. 目标项合法、但其后有非法项 → 失败；
10. 失败时 `*out` 保持不变。

编译与运行（开启测试会写入现有 build cache，如需隔离可另设 build 目录）：
```bash
cd /users/Yuanhao/Duranta-openairinterface5g/cmake_targets/ran_build/build
cmake -DENABLE_TESTS=ON . && make rrc_cell_management_test && ctest -R rrc_cell_management_test --output-on-failure
```

### 阶段 D：编译、split E2E、monolithic 回归
统一编译：`cd cmake_targets && ./build_oai --gNB --nrUE -w SIMU --build-lib telnetsrv`

| # | CU plmn_list | DU plmn_list | 预期 |
|---|---|---|---|
| 1 | 93, 94 | 93, 94 | 同基线 |
| 2 | 94, 93 | 93, 94 | UE1→93 / Core1，UE2→94 / Core2 |
| 3 | 93, 94 | 94, 93 | 同上（UE index 互换，CU 仍正确） |
| 4 | 93, 94 | 95, 93, 94（95 用独立 slice，仅 DU） | UE1 index 2→93，UE2 index 3→94；修复前 UE1 会错选 94、UE2 会越界 |
| 5 | 93 | 93, 94 | UE1 正常；UE2 本地释放 |
| 6 | monolithic 原配置 | — | 双 UE IP + 同时 ping |

所有配置变更中 PLMN 与其 `snssaiList` 必须成对移动。CU 仅配置 93 时，AMF2 的 NG Setup 可能失败，属预期。

判定是否选对 Core（不能只看是否拿到 IP）：CU `Selected PLMN`/`Selected AMF` 日志；对应 AMF 日志中 Initial UE Message 的 SUCI MCC/MNC；UE IP 所属地址池；`enp6s0f0` 上 `udp port 2152` 的 GTP 目的地址（10.10.3.2 或 10.10.3.3）；在 namespace 内 ping 对应 DN（10.10.1.2 / 10.10.2.2）并在 DN 侧确认。

用例 5 预期：CU 日志出现无法解析或 PLMN 不被支持的 `LOG_E`，然后 `Send RRC Release`；DU 收到 UE Context Release Command；CU `removed UE CU UE ID …`；两个 AMF 都**没有**收到 UE2 的 Initial UE Message（可在 N2 SCTP 38412 抓包确认）；整个过程中也没有出现针对该 UE 的 NGAP UE Context Release Request / Complete。UE 随后可能重新接入，并重复这一过程。

### 验证覆盖与局限
| 内容 | 验证方式 |
|---|---|
| SIB1 PLMN 解析、累计索引、MCC 继承、容量、失败时不改输出 | helper 单元测试（可执行） |
| 同序、反序、数量不同、CU 不支持该 PLMN、monolithic 回归 | E2E 用例 1–6（可执行） |
| 正常接入不受事务检查影响 | E2E：每个 UE 都有 Initial UE Message，且没有新增的 `LOG_W`（可执行） |
| 本地释放主路径（RRCRelease → Complete → 删除） | E2E 用例 5（可执行） |
| 重复或事务 ID 不匹配的 SetupComplete | 仅代码审查 |
| 本地释放期间收到 DU Release Request、UL-DCCH | 仅代码审查 |
| 本地释放期间 F1 断链或 E1 丢失 | 仅代码审查 |
| PDCP 失败时的兼底路径 | 仅代码审查 |
| DU Update 解码失败、所有权转移、不回 Ack | 仅代码审查；静态配置的 E2E 无法触发 |
| CU UE ID 复用造成的迟到消息竞态 | 未覆盖（已知局限） |

不为这些路径新建 mock 框架。如有需要，可另行用 ASan 构建做人工注入测试（不在本次范围内）。

---

## 7. 明确不做、协议缺口与已知局限

**协议缺口（明确记录，不宣称合规）**
- TS 38.473 §8.2.4.3 要求 CU 在无法接受 Update 时回复 gNB-DU CONFIGURATION UPDATE FAILURE 并给出原因。OAI 目前只有这个 ITTI 类型和一个不完整的结构体，没有编码/解码、没有 CU 发送、也没有 DU 处理。本次 DU Update 解码失败时只记录错误日志并返回，**没有**发送任何协议回复（既没有 Ack，也没有 Failure）。这不是完整合规的拒绝。不回复对 DU 的实际影响没有测试过。
- 本函数中其他已有的拒绝路径（PLMN 不匹配、小区不属于该 DU、`update_cell_info()` 失败）原本就是这样直接 `return`，行为不变。

**明确不做**
- 完整的 gNB-DU CONFIGURATION UPDATE FAILURE 流程；
- 缺少系统信息时提前拒绝 SA F1 Setup；在 F1 Setup 中新增 PLMN 校验或拒绝；
- 派生 PLMN 缓存（C 方案），或按 Served PLMNs 顺序解释（B2 方案）；
- UE 侧支持 12 个 PLMN / 多个外层条目，以及 IMSI 无匹配时回退为 1 的问题；
- AMF 选择算法，包括跨 PLMN 回退到“最高容量 AMF”（`ngap_gNB_nas_procedures.c:113-121`）；
- NGAP 模块对未知 UE 合成释放命令的行为；
- 通用的释放定时器或重试框架；
- F1 Setup 失败时未移除已加入的 DU 容器（`rrc_gNB_du.c:701` 起）。

**已知局限（保留）**
- 没有释放定时器：如果 DU 既不回 Complete、F1 也不断链，`local_release` UE 会一直遗留。
- CU UE ID 复用：上下文删除后到达的消息可能匹配到新 UE（§4.6）。
- 重建回退（`rrc_gNB.c:1707-1708`）会对旧上下文发 NGAP Release Request。`local_release` UE 没有 AS 安全上下文，正常情况下不会发起重建；只有 UE 行为异常时，这条路径才会经 NGAP 合成命令，对该旧上下文触发一次 `an_release` 释放。本次不为此扩展状态。
- E1/CU-UP 丢失时，对未关联 CU-UP 的 UE（包括 `local_release` UE）先发 NGAP Release Request、再立即删除。异步合成的命令在 ID 被复用时可能落到新 UE 上。这是已有行为，本次不修。
- 本地释放后 UE 会重新接入，并再次被释放；是否在 RRCRelease 中携带等待时间不在本次范围内。
- 原先 `DevAssert(cell)` 的地方改为返回 false 并本地释放。
- “OAI DU 生成的 SIB1 PLMN 列表能通过解析”有三个前提：配置合法（MCC 0–999；MNC 的数值与 `mnc_length` 一致，代码对 `mnc_length` 有断言、对 MCC 范围没有检查）；当前 `get_SIB1_NR()` 每一项都带 MCC，且只有一个外层条目；PLMN 数量不超过 6（`set_plmn_config()` 有断言）。本方案不依赖这一点来保证正确性。

---

## 8. 修订记录与待确认项

**修订 2（2026-09-28）相对修订 1 的变化**
1. 新增 SetupComplete 事务检查（§4.2）。修订 1 把“重复 SetupComplete 进入新失败分支”列为已有问题，这是错的：这是本次新引入的回归，必须在本次修复。
2. 只保留 `nr_rrc_sib1_get_plmn()`；删除 `plmn_list_valid`；F1 Setup 不再改动；DU Update 只在解码失败时让 SIB1 失效。
3. **保留** DU Release Request 对 `local_release` 的检查（§4.3）。复审时曾建议删除它，理由是“NGAP 只会丢弃”，这个理由不成立：NGAP 会合成一条释放命令发回 RRC。
4. **撤销** `invalidate_du_connections()` 的改动（§4.5）。同样由于 NGAP 会合成释放命令，F1 断链时 NAS 前的 UE 会被现有路径清理。修订 1 中“非 `local_release` 的 NAS 前 UE 遇到 F1 断链或 DU 释放时必然泄漏”的说法是错的，已删除。
5. 收紧了过度保证的表述：删除后迟到消息可能匹配到复用 ID 的新 UE；不回 Ack 的影响未测试、且不合规；“OAI 列表合法”带前提；“只影响新接入”依赖事务检查；`mac_rrc_dl_handler.c:158-176` 不能证明 PLMN 会被重排。
6. 补充了启动说明：停止旧的 monolithic、保留已有 namespace、分终端并写明工作目录、给出 UE2 的完整命令。

**范围**：7 个生产文件加 1 个既有测试文件不变。`rrc_gNB_du.c` 中只修改 `extract_sys_info()` 和 DU Update 函数。

**待确认项**：在审核中确认接受 §7 所列的协议缺口和已知局限。没有其他阻塞项。

**后续顺序**：确认文档 → 用户手动跑未修改代码的同序 split 基线 → 反序复现 → 获批后实现 → 单元测试与编译 → split 各场景测试 → monolithic 回归。
