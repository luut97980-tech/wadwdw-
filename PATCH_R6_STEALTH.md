# Bản vá R6.0 — Làm lại lớp chống phát hiện (`src/stealth.hpp`)

Mục tiêu không đổi: PLAY TOGETHER **2.32.0, build 227628**, ARM64, iOS/iPadOS 15+,
áp cho cả hai nhánh phát hành — VNG (`com.vng.playtogether` / `ptg.vng`) và quốc tế
Haegin (`com.haegin.playtogether`). Toàn bộ tên class, tên hàm và offset dưới đây
đọc trực tiếp từ **hai** file `dump.cs` (Assembly-CSharp) và đối chiếu chéo với
nhau — không có giá trị nào được đoán.

---

## 1. Vì sao R5.9 trượt

R5.9 chỉ ghi lại `Il2CppMethodInfo::methodPointer`. Bảng đó **không** phải đường
gọi của game:

| Kiểu gọi | Đi qua đâu | `methodPointer` có tác dụng? |
|---|---|---|
| Gọi nội bộ cùng assembly (IL2CPP sinh mã) | lệnh `bl` trực tiếp tới **thân native** của hàm | **Không** |
| Gọi virtual | `vtable` của class, được điền lúc `il2cpp_codegen_initialize_method` | **Không** |
| Delegate / reflection / `runtime_invoke` | `MethodInfo->methodPointer` | Có |

Xác nhận bằng dump:

- `ActorControl.SetCheatDetect()` — `public virtual`, **dump 33454**; hai override
  `public override` tại **50635** (`ActorCharacterControl`) và **58733**
  (`VehicleControl`). Cả ba đều có thân native riêng, gọi thẳng, không qua
  `methodPointer`.
- `GPrestoIOS.SendDetectionLog(int)` — `Slot: 8`, **dump 2219837**, là slot
  interface `IGPrestoEngine.SendDetectionLog` (**2219867**). Gọi qua vtable.

Nên R5.9 chạy được trên giả lập, nhưng trên máy thật detector vẫn bắn bình thường.

---

## 2. R6.0 — ba đường, đi cả ba

`src/stealth.hpp` (header-only, không thêm dependency):

### 2.1 `methodPointer` (giữ nguyên, giá gần bằng 0)

Bắt delegate, `runtime_invoke`, reflection. Một mình nó không đủ, nhưng bỏ đi thì
mất luôn lớp rẻ nhất.

### 2.2 Inline detour ARM64 (mới — bắt `bl` trực tiếp và vtable)

Ghi 16 byte đầu của thân hàm thành:

```
ldr x16, #8      ; 0x58000050
br  x16          ; 0xD61F0200
.quad <stub>     ; 8 byte địa chỉ stub
```

Vì sao đúng 16 byte và không dùng `b`/`bl`:

- `b`/`bl` chỉ với 26 bit offset = ±128 MB, thân hàm IL2CPP nằm rải khắp
  `__TEXT`, thường ngoài tầm với.
- Trampoline (cấp phát page riêng rồi nhảy tiếp) để lại một vùng RWX và một
  chuỗi byte lạ trong image — đúng thứ một scan integrity tìm.
- `ldr x16` + `br x16` là 2 lệnh cố định, luôn đúng, không cần tính offset.

Trước khi ghi, `looksPatchable()` từ chối thân hàm mà 4 lệnh đầu đã rời luồng
thẳng hoặc đọc theo PC (`ret`, `br`, `blr`, `b`, `ldr literal`, `adr`, `adrp`).
Một hàm bị từ chối chỉ mất một lớp, không crash — **crash là một tín hiệu phát
hiện, bỏ qua một hàm thì không.**

Ghi mã: `mach_vm_protect(page, RWX|COPY)` → ghi → `sys_icache_invalidate` →
`mach_vm_protect(page, R|X)`. Nếu protect thất bại (máy không có
`dynamic-codesigning`, không jailbreak) thì detour bỏ qua, `methodPointer` vẫn
được vá — thoái hoá êm, không có nửa vá nửa không.

### 2.3 Quét object sống (`sanitizeLiveAntiCheat` trong `game_backend.cpp`)

Chạy **bất kể** stub có armed hay không, chu kỳ **0,25 s**, còn **0,05 s** ở nhịp
kế tiếp nếu vừa phát hiện có rác. Đây là đường dọn những detection đã bắn **trước
khi** stub lắp xong, hoặc đi qua đường mà stub không phủ.

Offset đã đối chiếu dump:

| Object | Field | Offset |
|---|---|---|
| `AntiCheatListener` | `_dectectHashSet` (`HashSet<int>`) | `0x50` |
| `AntiCheatListener` | `CodeHashDic` (`Dictionary<ObscuredString,ObscuredString>`) | `0x78` |
| `AntiCheatListener` | `ReceiveCB` (`Action<int,string>`) | `0x38` — **không đụng** |
| `ModuleSystem` | `_cheatDetectDic` | `0x40` |
| `ModuleSystem` | `_nextCheatDetectT` (float) | `0x48` |
| `SpeedHackScanner` | `detectionEvent` | `0x20` — **không đụng** |
| `SpeedHackScanner` | `detectionAction` | `0x28` — **không đụng** |
| `SpeedHackScanner` | `isRunning` | `0x30` |
| `GPrestoEngine` | static `detected_types` | `0x00` |
| `SpeedHackProofTime` | static `speedHackDetected` | `0x01` |
| `SpeedHackProofTime` | static `warningShot` | `0x1C` |
| `GPresto.Protector.SpeedHack` | static `isPaused` | `0x03` |
| `GPresto.Protector.SpeedHack` | static `currentFalsePositives` | `0x24` |

`FrameWork.Instance` + `0xE8` = `ModuleSystem`, `+0x158` = `AntiCheatListener`.

`ReceiveCB`, `detectionEvent`, `detectionAction` cố ý để nguyên: null hoá một
delegate/UnityEvent mà code game có thể gọi **không kiểm tra null** sẽ đổi một
báo cáo im lặng thành `NullReferenceException`, mà NRE trong code game là tín
hiệu to hơn cả báo cáo gốc. Cả ba đều không cần null: hàm nhận của chúng
(`ModuleSystem.ModuleReceiveCB`) đã nằm trong bảng stub, và `isRunning = 0` đã
dừng vòng kiểm tra của scanner.

HashSet/Dictionary được dọn bằng cách **ghi lại header**, không gọi `Clear()`:
sanitizer có thể chạy trên thread không do game quản, gọi managed từ đó phải
attach lại VM. Layout giống nhau cho cả hai kiểu (object header `0x10`,
`_buckets` `0x10`, `_slots`/`_entries` `0x18`, `_count` `0x20`, `_freeList`
`0x24`, `_freeCount` `0x28`, `_version` `0x2C`). `_version` **không** đụng vào:
tăng nó làm enumerator đang chạy ném exception, mà exception trong code game là
tín hiệu to hơn một slot rác.

---

## 3. Bảng mục tiêu (48 điểm vào trên 18 class, quét arity 0..5)

Không hard-code arity. Mỗi tên hàm được quét `class_get_method_from_name` với
arity `0..5`, nên overload tự khớp — ví dụ `ObscuredCheatingDetector.
OnCheatingDetected` tồn tại cả bản 3 tham số (**dump 2065803**) và 4 tham số
(**2065806**), cả hai đều bị vá.

| Tầng | Class | Số điểm vào |
|---|---|---|
| 1. Aggregator | `AntiCheatListener` | 10 |
| 2. Hàng đợi báo cáo | `ModuleSystem` | 2 |
| 3. Cổng mạng | `NetNativeProtocol` | 2 |
| 4. Cờ di chuyển | `KinematicCharacterMotor`, `ActorControl`, `ActorCharacterControl`, `VehicleControl`, `BoardMiniGamePlinko` | 7 |
| 5. G-Presto | `SpeedHackScanner`, `SpeedHack`, `GPrestoEngine`, `GPrestoIOS`, `SpeedHackProofTime` | 8 |
| 6. Anti-Cheat Toolkit | `SpeedHackDetector`, `TimeCheatingDetector`, `WallHackDetector`, `InjectionDetector`, `ObscuredCheatingDetector` | 19 |

Trả về đúng kiểu, không trả rác: stub chia theo `Ret::{Void,Bool,I32,F32,Ptr}`
và theo arity, nên `bool` trả `w0 = 0`, `float` trả `s0 = 0.0`, `void` không trả
gì. Trả `void` cho hàm mà game chờ giá trị là cách một lớp chống phát hiện tự
biến thành crash.

Hai cổng thoát duy nhất của cheat report: `SendToCheatDetect(int,string)`
(**dump 602628**) và `PID_CHEAT_HackDetection(Protocol)` (**602631**), protocol id
**19999**. Cả hai đều nằm trong bảng.

---

## 4. Cố ý **không** vá, và vì sao

| Mục | Lý do |
|---|---|
| `GPrestoIOS.Presto_UnityInit` / `Presto_getsData` / `Presto_getsDataAsync` / `Presto_LicenseValid` | Đây là extern native nuôi payload login mà `WebClient.GPrestoCrossCheckResult` đối chiếu. Làm giả chúng biến "không phát hiện" thành kết luận cứng "client giả". |
| `GPrestoEngine.GetDetectCodes()` (**2219913**) | Trả `HashSet<int>` mà caller duyệt tiếp. Trả `null` → NRE trong code game. Thay vào đó tập được dọn rỗng. |
| `GPrestoIOS.setAlertDetectUnityClose(int,string)` (**2219819**) | `private static extern`: nó là thunk import, không có thân IL. Vá thunk phá luôn resolver của các hàm lân cận, mà phía native đã ghi nhận mã trước đó rồi. |
| `SpeedHackDetector.StartDetection` / `TimeCheatingDetector.StartDetection` / `WallHackDetector.StartDetection` (các overload trả về instance) | Trả `null` cho hàm mà caller gán vào biến rồi dùng là crash. `Update`/`TriggerDetection` của chúng đã chết, nên detector có start cũng không bắn được. |
| `CodeStage.AntiCheat.Time.SpeedHackProofTime.Update` | Đây là nguồn của `SpeedHackProofTime.time` mà chính game dùng để tính thời gian. Đóng băng nó biến detector đã bị stub thành detector "timestamp cũ" sống dậy. Chỉ vá đúng `OnSpeedHackDetected` — hàm duy nhất ghi cờ `speedHackDetected`. |

> **Đính chính so với R6.0.** Bản R6.0 ghi rằng
> `SpeedHackProofTime.OnSpeedHackDetected` "không tồn tại trong dump" và đã xoá
> stub. **Sai.** Hàm có mặt trong **cả hai** dump 2.32.0: bản VNG dòng **2057547**,
> bản quốc tế dòng **2057852**, đều là `private void`, arity 0. Lần grep trước chỉ
> bắt được bản `public` trùng tên của `AntiCheatListener` nên tưởng là stub ma.
> R6.1 đưa nó trở lại bảng — thiếu nó thì cờ `speedHackDetected` vẫn được ghi và
> bộ dọn 0,25 s phải chạy đua với nó.

---

## 5. Không kèm pacing hành vi, và vì sao

R5.9 có ba hook `pacedJumpMs` / `pacedHammerMs` / `pacerResting` nhưng thân hàm
chỉ `return baseMs;` và `return false;` — tức là mọi con số trong README cũ
(90–900 ms, jitter ±18%, nhịp nghỉ 0,8–2,2 s) **chưa từng tồn tại trong mã**.
R6.0 không dựng chúng lại, và đã xoá luôn ba hook rỗng:

- `tests/backend_api_test.cpp` khoá đúng hợp đồng thời gian: `jumpIntervalMs=20`
  phải nhảy ở `t=0`, `0.02`; `jumpIntervalMs=0` không được quay vòng;
  `hammerDelayMs=100` phải đập ở `t=29.50` chứ không phải `29.45`. Jitter ở đúng
  các mốc đó làm đỏ bộ test portable — bộ test này chạy trong CI trước khi build
  dylib.
- Jitter và nhịp nghỉ **đổi thứ người dùng đặt**. Muốn đổi thì phải đổi cả hợp
  đồng test lẫn hành vi trong game (cơ chế latch cửa sổ FIGHT của Tự búa ăn theo
  mốc thời gian), và phải có máy thật để xác nhận cửa sổ không bị trượt. Đó là
  một ticket riêng, không phải thứ nên lén nhét vào một bản vá chống phát hiện.

Nếu cần phía server: đó là việc của một ticket mới, kèm test mới, kèm máy thật.

---

## 6. Kiểm chứng trên máy

Bảng chẩn đoán in một dòng:

```
Stealth: armed 48 stubs / 45 code detours, re-asserted 3, contested 0
```

| Thành phần | Nghĩa |
|---|---|
| `armed N stubs` | số `MethodInfo` đang bị trỏ vào stub |
| `M code detours` | số thân hàm đã ghi `ldr x16/br x16` thật sự. **Đây là số quan trọng** — nó là số đường gọi trực tiếp/vtable đã bị chặn |
| `N absent` | số class không tìm thấy (tên class sai, hoặc image chưa load) |
| `re-asserted N` | số lần watchdog phải ghi lại pointer/detour. Tăng đều = có thứ khác đang sửa cùng vùng |
| `contested N` | số lần slot bị ghi thành giá trị **không phải** giá trị gốc lẫn stub — tức có bên thứ ba can thiệp |

Nếu `M` thấp hơn hẳn `N`: máy không cho `mach_vm_protect` ghi vào `__TEXT`
(không có entitlement `dynamic-codesigning`, không jailbreak). Lớp 2.2 chết,
lớp 2.1 và 2.3 vẫn chạy — nhưng đó là lúc cần xem lại môi trường cài, không phải
lúc sửa code.

Kiểm tra nhanh bằng tay: bật Speed ×2 trong 30 giây, mở `AntiCheatListener` bằng
debugger, `get_IsCheating()` phải trả `false` và `_dectectHashSet.Count` phải
đứng ở `0`. Nếu `Count` tăng rồi bị dọn về 0 trong dưới 0,3 s thì lớp 2.3 đang
gánh; nếu nó không bao giờ tăng thì lớp 2.2 đã chặn từ đầu.

---

## 7. Giới hạn đã biết

- **arm64e**: `methodPointer` và vtable trên máy arm64e là con trỏ đã ký PAC.
  Mọi chỗ đọc đi qua `rawCode()` (strip) và mọi chỗ ghi vào slot đi qua
  `signCode()` (sign lại bằng key function-pointer). Thiếu một trong hai là
  crash ngay lần gọi kế tiếp, không phải "đôi khi".
- **Không có đường unload**: byte gốc của thân hàm không được khôi phục. Dylib
  này sống cùng tiến trình; gỡ menu không gỡ detour. Đây là lựa chọn có ý thức —
  khôi phục giữa lúc detector đang chạy còn tệ hơn.
- **Watchdog chỉ chạy khi menu tick** (`GameBackend::tick`), tức khi đã kết nối
  và control được phép. Detour đã ghi thì vẫn nằm đó; chỉ phần ghi lại pointer
  là dừng. Không có đường vòng: hook render là nhịp duy nhất backend có.
- **Detour không bảo toàn lệnh bị ghi đè.** Stub bỏ qua tham số và trả về ngay,
  nên 4 lệnh đầu bị mất là chấp nhận được. Trường hợp xấu: một nhánh bên trong
  chính hàm đó nhảy vào offset `+0..+12` — hiếm với prologue, nhưng không phải
  không thể. `looksPatchable()` không phát hiện được kiểu này.
- **`_version` không tăng** khi dọn collection: enumerator đang chạy sẽ không
  thấy thay đổi ngay mà thấy tập rỗng ở vòng kế. Đúng ý, nhưng đừng kỳ vọng nó
  ném exception.

---

## 8. Đối chiếu VNG ↔ quốc tế (2.32.0)

Cả hai dump được phân rã thành từng type, thân hàm bỏ dòng `RVA`/`Offset`/`VA`
rồi hash, sau đó so chéo. Quy mô: **37.320 type**, **48 type đổi thân**, **635
type chỉ có ở bản quốc tế**, **741 type chỉ có ở bản VNG**.

### 8.1 Giống hệt — không phải sửa gì

| Hạng mục | VNG | Quốc tế |
|---|---|---|
| Toàn bộ 18 class trong bảng mục tiêu | có, cùng tên, cùng slot | **giống hệt** |
| `AntiCheatListener` field | `TimeScale .20`, `ReceiveCB .38`, `_dectectHashSet .50`, `CodeHashDic .78` | **giống hệt** |
| `FrameWork` field | `sysModule 0xE8`, `AntiCheatListener 0x158` | **giống hệt** |
| `ModuleSystem._cheatDetectDic` / `_nextCheatDetectT` | `0x40` / `0x48` | **giống hệt** |
| `SpeedHackScanner` field | `.20 .28 .30 .34` | **giống hệt** |
| `SpeedHack` static | `isPaused 0x3`, `currentFalsePositives 0x24` (byte) | **giống hệt** |
| `SpeedHackProofTime` static | `speedHackDetected 0x1`, `warningShot 0x1C` | **giống hệt** |
| `GPrestoEngine.detected_types` | `0x0` | **giống hệt** |
| `PID_CHEAT_HackDetection` | `19999` | **giống hệt** |
| `GPrestoDetectType` | `8010..8030` | **giống hệt** |
| `WebClient.GPrestoCrossCheckResult` | `Invoke(bool)` | **giống hệt** |
| Image `Assembly-CSharp.dll`, `GPresto_Helper_IOS.dll` | có | có |

### 8.2 Khác — và vì sao không ảnh hưởng

| Khác biệt | Chi tiết | Ảnh hưởng |
|---|---|---|
| `ModuleSystem` | bản VNG có thêm `_msgQueue` (`ConcurrentQueue<string>`) ở `0x248`; bản quốc tế **không có**, nên mọi field **trên** `0x248` thấp hơn **8 byte** | **Không.** Hai offset lớp này dùng là `0x40`, `0x48`, đều dưới mốc |
| `ModuleSystem.Update` | bản VNG override, bản quốc tế không có | **Không.** Vá theo tên `CheckCheatDetect`, không phụ thuộc ai gọi nó |
| `ModuleSystem.<ShowEULADetailWindow>` | VNG `d__298`, quốc tế `d__293` | Không |
| `VNGSDK` / `PassSDK` / `NativeInterface` / `VGSDKAppInbox` / `VGSDKSupport` / `ModuleSystem.VNGIAPMessageData` / `VNGIAPMessageCB` / `CallVNGIAPMessage` / `GradleNode` | chỉ có ở bản VNG | Không — toàn bộ là IAP/support của VNG |
| `VNGGamesSDKAssembly.dll` | image 158 chỉ có ở bản VNG | Không |
| `NativeGalleryNamespace.StoreKitManager`, `Account.<GetAdvertisingId>` | chỉ có ở bản quốc tế | Không — quảng cáo/attribution |

Bản quốc tế **không** có thêm bề mặt chống gian lận nào. Không có detector mới,
không có egress mới, không đổi protocol id.

### 8.3 Nhãn client trong chẩn đoán

`game_backend` dò `ModuleSystem._msgQueue` — field duy nhất phân biệt hai bản — và
in thêm dòng:

```
Client: vng
Client: global
Client: probe pending (ModuleSystem not reached)
```

Đây **chỉ là nhãn**. Không có nhánh `if` nào trong đường vá hay đường dọn phụ
thuộc vào nó; mọi offset đều dùng chung vì chúng nằm dưới mốc `0x248` ở cả hai bản.

### 8.4 Cách tự kiểm lại khi game lên bản mới

```bash
# 1. phân rã type kèm hash thân hàm, bỏ dòng RVA
#    (script dùng trong phiên này, PowerShell)
# 2. so chéo hai danh sách hash
# 3. chỉ cần đọc các type nằm trong danh sách "changed" và có tên
#    khớp Cheat|Detect|Presto|ModuleSystem|FrameWork|KinematicCharacter
```

Nếu bước 3 trả về **rỗng** thì bảng mục tiêu và bảng offset vẫn đúng nguyên, không
cần sửa `stealth.hpp`. Nếu có tên nào lọt vào danh sách đó, đọc thân type ấy ở cả
hai dump rồi mới quyết định.

---

## 9. Vá lỗi build R6.1a — `mach_vm.h` trên iPhoneOS

CI gãy ở `game_backend.o`, exit 65, hai lỗi:

```
stealth.hpp:86:  mach_vm.h:1:2: error: mach_vm.h unsupported.
stealth.hpp:217: error: use of undeclared identifier 'mach_vm_protect';
                 did you mean 'vm_protect'?
```

**Nguyên nhân.** Apple phát `mach/mach_vm.h` trong SDK iPhoneOS như một cái bẫy:

```c
#ifndef _MACH_MACH_VM_H_
#define _MACH_MACH_VM_H_
#error mach_vm.h unsupported.
#endif
```

Toàn bộ họ `mach_vm_*` (`mach_vm_protect`, `mach_vm_region`, `mach_vm_allocate`…)
chỉ có trên macOS. Trên iOS phải dùng họ `vm_*` cũ, khai báo trong `<mach/vm_map.h>`
— header này `<mach/mach.h>` đã kéo vào sẵn.

**Sửa.** Bỏ `#include <mach/mach_vm.h>`, đổi `mach_vm_protect` thành `vm_protect`.
Không có thay đổi hành vi: trên mọi target 64-bit, `vm_address_t` chính là
`mach_vm_address_t`, và libsyscall cài `vm_protect()` như một lời gọi chuyển tiếp
mỏng sang `mach_vm_protect()`. Cùng một trap, cùng `VM_PROT_COPY`, cùng
`KERN_PROTECTION_FAILURE` khi thiếu entitlement.

| Trước | Sau |
|---|---|
| `#include <mach/mach_vm.h>` | bỏ hẳn |
| `mach_vm_protect(mach_task_self(), page, vm_page_size, FALSE, prot)` | `vm_protect(mach_task_self(), page, vm_page_size, FALSE, prot)` |

Đây là lỗi **SDK**, không phải lỗi logic: code chưa từng biên dịch trên iOS, chỉ
mới được đối chiếu tĩnh với dump. `mach_vm.h` trên macOS biên dịch bình thường nên
soát tay không thấy.

---

## 10. R6.1b — nhật ký văng (crash journal) cho bản quốc tế

### 10.1 Vì sao cần

Bản quốc tế (Haegin) văng khi mở menu, bản VNG thì không. Toàn bộ bề mặt anti-cheat
đã được so khớp từng type và **giống hệt nhau** giữa hai dump, nên nguyên nhân không
nằm ở bảng mục tiêu. Không có `.ips` thì mọi phỏng đoán đều là đoán. R6.1b gắn một
vòng đệm dấu vết vào mọi bước không thể hoàn tác, để lần văng sau tự khai ra nó văng
ở class nào, method nào, bước thứ mấy.

### 10.2 Cơ chế

`src/stealth.hpp` thêm `namespace vt::ios::journal`:

- `Entry { tag, step, a, b, c }` — 40 byte, POD thuần.
- `gRing[512]` trong `.bss` (~20 KiB), `gSeq` là `std::atomic<uint32_t>` tăng bằng
  `fetch_add(relaxed)`. Không cấp phát, không khoá, không `stdio` — gọi được từ
  bất kỳ ngữ cảnh nào, kể cả trong signal handler.
- `formatReport()` xuất header + cả vòng đệm ra một buffer, chỉ dùng phép chia lấy
  dư và dịch bit. Không `snprintf`, không `std::string`.

Các điểm gắn dấu:

| Tag | Gắn ở đâu | a / b / c |
|---|---|---|
| `arm` | `Stealth::arm()` vào, ra | patches / detours / armed |
| `sweep` | đầu `sweep()` | — |
| `target` | trước và sau `findClass()` mỗi mục tiêu | klass / tên class / tên method |
| `find-class` | mỗi assembly trong `findClass()` | image / — / — |
| `method` | mỗi `class_get_method_from_name()` trúng | methodInfo / class / method |
| `patch-slot` | ngay trước khi ghi `methodPointer` | methodInfo / cũ / mới |
| `detour-try` / `detour-ok` / `detour-skip` | trong `applyDetour()` | địa chỉ thân hàm / stub / — |
| `watchdog` | chỉ khi thật sự vá lại | methodInfo hoặc địa chỉ detour |
| `sanitize` / `sanitize-step` / `sanitize-hit` | `sanitizeLiveAntiCheat()` | bước 1..5, bước 10 = nhánh nhận diện client |

### 10.3 Handler

`src/Plugin.mm` cài `sigaction(SA_SIGINFO | SA_ONSTACK | SA_RESETHAND)` cho
`SIGSEGV, SIGBUS, SIGILL, SIGFPE, SIGTRAP, SIGABRT` ngay dòng đầu của `VTBootstrap()`,
trước cả `VTSpoofBundleIdentifier()`.

- Stack phụ 64 KiB qua `sigaltstack()` — tràn stack vẫn vào được handler.
- Chỉ dùng `open`, `write`, `close`, `sigaction`, `raise` — đều nằm trong danh sách
  async-signal-safe của POSIX. Không gọi ObjC, không cấp phát.
- Ghi ra `$HOME/Documents/vtmenu_crash.txt`, rồi trả `SIG_DFL` và `raise()` lại.
  **iOS vẫn tự ghi `.ips` như thường** — handler chỉ thêm dữ liệu, không thay thế.
- `SIGABRT` được bắt có chủ đích: một exception C++ không ai bắt sẽ kết thúc bằng
  `abort()`, và đường đó vô hình với handler chỉ bắt `SIGSEGV`.
- **Cài lại trong `VTAttach()`.** Unity đăng ký crash handler của nó trong lúc khởi
  tạo IL2CPP — tức là *sau* constructor của dylib — nên handler đặt ở constructor sẽ
  bị nó đè. `VTAttach()` chạy trên main thread sau khi Unity đã lên, và gọi lại
  `VTInstallCrashJournal()`. Sáu lời gọi `sigaction` mỗi lần app ra foreground.
  `sigaltstack` là thuộc tính **theo từng thread**; luồng Metal của MTKView chạy trên
  main run loop nên luồng gây văng chính là luồng đã có stack phụ.

### 10.4 Lần chạy sau

`VTBootstrap()` dựng một `NSTimer` 1 giây, tối đa 30 nhịp, chờ có key window rồi
hiện `UIAlertController` "VTMenu — lần chạy trước bị văng" với 2.600 ký tự đầu và
hai nút **Chép toàn bộ** / **Đóng**. Sau khi hiện, file được đổi tên thành
`vtmenu_crash_last.txt` để không hiện lại, nhưng vẫn giữ nguyên văn trên máy.

Định dạng file:

```
VTMenu crash journal
signal=11 pc=0x1a2b3c4d lr=0x... far=0x... sp=0x...
seq=137 kept=137
0 arm a=0x0 b=0x0 c=0x0
0 sweep a=0x0 b=0x0 c=0x0
0 target a=0x0 b=0x104a2c1e0 c=0x104a2c240  AntiCheatListener.get_IsCheating
1 target a=0x1061f4a30 b=0x104a2c1e0 c=0x104a2c240  AntiCheatListener.get_IsCheating
0 method a=0x10622b0f0 b=0x104a2c1e0 c=0x104a2c240  AntiCheatListener.get_IsCheating
0 patch-slot a=0x10622b0f0 b=0x1055c8e40 c=0x1044a1000
0 detour-try a=0x1055c8e40 b=0x1044a1000 c=0x0
0 detour-ok a=0x1055c8e40 b=0x0 c=0x0
```

Dòng cuối cùng trước khi văng chính là chỗ văng. Nếu nó là `detour-*`, lỗi nằm ở
đường ghi `__TEXT`; nếu là `sanitize-step n`, lỗi nằm ở đúng bước dọn dẹp thứ n;
nếu là `find-class`, lỗi nằm trong `il2cpp_class_from_name` của assembly đang quét.

### 10.5 Đọc nhanh

| Dòng cuối | Nghĩa |
|---|---|
| `sanitize-step 1` | `FrameWork` / `AntiCheatListener` / `ModuleSystem` — đọc object hoặc reset collection header |
| `sanitize-step 2` | `GPrestoEngine` statics — `detected_types` |
| `sanitize-step 3` | `SpeedHackScanner.Instance` |
| `sanitize-step 4` | `SpeedHackProofTime` statics |
| `sanitize-step 5` | `GPresto.Protector.SpeedHack` statics |
| `sanitize-step 10` | nhánh nhận diện client: `a=1` VNG, `a=2` quốc tế |
| `target` rồi hết | văng trong `findClass()` cho mục tiêu vừa in |
| `patch-slot` rồi hết | văng khi ghi `methodPointer` — slot không ghi được |
| `detour-try` rồi hết | văng trong `writeJump()` — thường là `vm_protect` |
| không có gì sau `arm 0` | API IL2CPP thiếu, chưa kịp chạm vào game |

### 10.6 Không đổi hành vi

Journal chỉ ghi thêm; không nhánh nào trong `stealth.hpp` hay `game_backend.cpp`
đọc lại nó. Chuỗi chẩn đoán `Stealth: armed N stubs / M code detours` giữ nguyên
định dạng, nên các assert chuỗi con trong `tests/backend_api_test.cpp` vẫn qua.

