# meta-pass — Bug 报告(当前分支)

[English](BUGS.md) | 简体中文

依据 `docs/assets/meta-pass-design.zh_CN.md` 对当前分支的代码审查结果。
每条:现象 → 根因 → 修复方案。严重级别:**高** = 数据损坏 / 功能失效,
**中** = 真实路径上的错误行为或潜在缺陷,**低** = 健壮性 / 规范性。
已排查并排除的疑点附在文末,附证据。

| ID | 严重级别 | 组件 | 一句话概述 | 状态(本分支已修复) |
|----|---------|------|-----------|---------------------|
| BUG-01 | 高 | `main/main.c` | 电量标签用未初始化栈缓冲构造;SOC 从未渲染 | **已修复** — `main.c:69-71` 加守卫与 `snprintf` |
| BUG-02 | 高 | `main/meta_sign.c` | `HOST_TEST` 变体的彩蛋 magic 判断反了 | **已修复**(上游提交)— `tests/test_meta_net_upload.c` 已加 m1–m4 回归测试 |
| BUG-03 | 高 | `tools/install-slot/`(开发副本) | 与线上安装页脱节:显示名 blob 写到分区外、双写擦掉签名、头部标志位判断漂移 | **已修复 + 结构性修复完成** — 开发页重复文件已删除;`server.mjs` 直接服务规范的 `install-slot/` |
| BUG-04 | 低 | `main/meta_net.c` | 上传成功日志用 `%d` 打印 `size_t` | **已修复** — `meta_net.c:407` 改 `%zu` |
| BUG-05 | 高 | `components/bsp/src/bsp_display.c`、`bootloader_components/meta_boot_hooks/hooks.c` | 子固件自身空闲深睡后按键唤醒落到启动器(面板未重新初始化 → 有背光、黑屏;点亮后仍停在启动器列表页而非子固件) | **已修复** — 从子固件 BSP 移植唤醒恢复 + bootloader 钩子在深睡唤醒时把该槽的 otadata 副本续期为 `VALID` |

---

## BUG-05 — 子固件息屏唤醒:有背光、界面全黑

**实测现象。** 启动槽位 1(一个 30 秒降亮、60 秒无操作进深睡的子固件),等它睡下后
按键唤醒:背光点亮,界面一直是黑的。不烧启动器、直接烧该子固件则不复现。

**根因 —— 两个独立缺陷,都在启动器一侧。**

深睡唤醒对 bootloader 而言是一次完整启动。子固件 BSP 留下的两样东西,启动器都没处理:

1. **唤醒后跑起来的是启动器,不是子固件。** 子固件从不写 `VALID`,其 otadata 副本停在
   `PENDING_VERIFY`。启动器开了 `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y`,IDF
   bootloader 在选择分区前先把它标成 `ABORTED`(`bootloader_utility.c:399-401`),
   `bootloader_common_ota_select_invalid()` 判其无效(`bootloader_common_loader.c:76`),
   两个副本都没有候选,于是打出 "Defaulting to factory image"
   (`bootloader_utility.c:412-415`)—— 即启动器。hook 的深睡早退(`hooks.c:98`)跳过的是
   *策略*,改不了这条 IDF 标准路径。所以唤醒不是"续玩",而是子固件当场退出。
2. **启动器的显示初始化无法唤醒一块已睡的面板。** `bsp_display_init()` 先发 SWRESET
   (`bsp_display.c:118`),SLPOUT 要到 `esp_lcd_panel_init()` 才发。面板此时仍处
   Sleep In(振荡器停振),SWRESET 不仅无效,还可能让命令解码状态机死锁 —— 子固件
   自己的 BSP 就记录了这个坑并提前发 SLPOUT(`tianshang .../bsp_display.c:173-177`)。
   启动器这份还从不释放子固件的 `gpio_hold_en()` / `gpio_deep_sleep_hold_en()`;
   hold 可跨复位保留,引脚一直被锁在休眠电平上,于是 SPI 命令全被吞掉,而背光
   (LEDC 单独一路)照常点亮。净效果:**有背光、黑屏**。

子固件的 BSP 只是更新:上游 commit `8501cb2` 做了加固,启动器这份早于它
(本仓库 `git log -S "prepare_deep_sleep"` 为空)。制品级证据:恢复相关字符串在子固件
镜像里各出现 1 次,在已发布的启动器镜像里 0 次。

**修复(第一版,留档:真机实测失败)。** 启用
`CONFIG_BOOTLOADER_SKIP_VALIDATE_IN_DEEP_SLEEP=y`,唤醒走 fast-boot 路径(按 RTC 保留
内存里的分区引导,不读 otadata、不标 ABORT)。真机上唤醒仍落到启动器。根因(据 IDF
源码):该选项把"上次引导的分区"记在 RTC 快速内存顶部的 `rtc_retain_mem_t`
(`SOC_RTC_DRAM_HIGH - 16`),而链接脚本只在 `CONFIG_BOOTLOADER_RESERVE_RTC_MEM` 打开时
才为它预留空间。子固件的构建没开这一项 —— 它的 `RTC_TIMER_RESERVE_RTC` 区域正好覆盖
那 16 字节 —— 于是子固件一运行就覆盖了记录,CRC 校验失败,
`bootloader_load_image_no_verify` 返回错误,`bootloader_utility_load_boot_image_from_deep_sleep`
打出 "Fast booting is not successful" 后回落到常规路径。该选项要求每个子固件配合,
正是开机策略规则所禁止的依赖。

**修复(最终版)。**(1) bootloader 钩子(`bootloader_after_init`,执行点在
`bootloader_start.c:39` —— flash 已就绪、`select_partition_number` 把 PENDING 标
ABORTED 之前)按 `esp_rom_get_reset_reason(0)` 分流。深睡唤醒时把正在运行的子固件的
`PENDING_VERIFY` 副本续期为 `VALID`,bootloader 随即引导回该槽位,且不需要子固件做任何
配置。该改写安全的前提是 CRC 只覆盖 `ota_seq`(`bootloader_common_loader.c:69-72`);先擦
后写则是因为 flash 只能把 1 写成 0,而 `0x1 -> 0x2` 需要把 bit1 置 1。(2) 把子固件的
唤醒恢复移植进启动器 BSP:在 SPI 接管引脚前先解除全局与单引脚 hold,再在面板复位前补发
`0x11` SLPOUT + 120ms。(2) 兜住仍回退到启动器的路径,对任何使用深睡的子固件都有效。

**验证。** `tests/test_display_wake_contract.py`(6 例)钉住顺序与续期接线;
`tests/test_meta_boot_policy.c` 覆盖 `must_resume` 全状态,并断言两条规则互斥。构建制品
已核对:`bsp_display_init` 反汇编顺序为 `gpio_deep_sleep_hold_dis` →
`gpio_config`/`gpio_hold_dis` → `spi_bus_initialize` → `esp_lcd_new_panel_io_spi` →
`esp_lcd_panel_io_tx_param(0x11)` → `vTaskDelay` → `esp_lcd_panel_reset`;bootloader ELF
里的 `bootloader_after_init` 调用了 `esp_rom_get_reset_reason`、
`bootloader_common_ota_select_crc`、`bootloader_flash_read`、
`bootloader_flash_erase_sector` 与 `bootloader_flash_write`,且 bootloader 镜像同时含有
两条策略字符串("resuming ota_%u … PENDING -> VALID" 与 "in VALID state -> erasing"),
fast-boot 字符串已消失。真机验证（2026-09-24）：子固件自行息屏进深睡后按键唤醒可回到该
固件且屏幕正常点亮，再走一轮息屏/唤醒仍能续玩。未复测：冷复位回滚到启动器、其他子固件、
其他板卡版本。

---

## PASS-RADAR "仍然提示未签名" — 根因与结论

实测现象:用修复后的签名工具签出的 pass-radar 固件,设备引导时仍提示未签名
(launcher 日志 `signature: unsigned`;当时还会在屏幕上弹出 "Unsigned firmware!" 警告页,
该页已于 2026-09-23 移除)。

排查过程(全部在本机可复现):

1. **签名链本身是正确的。** `tools/signing/run-verify-tests.sh` 直接编译
   固件真实验签代码(`main/meta_sign.c`,不含 `HOST_TEST`),用固件内嵌的
   `meta_sign_pubkey.h` 验签:最新签名镜像
   (`../pass-radar/build/pass-radar_v0.1-2-g8fcce59-signed.bin`,19:09)
   返回 `META_SIG_OK`;修复前签的旧镜像(`pass-radar_signed_v2.bin`,13:53)
   正确返回 `META_SIG_VERIFY_FAIL`(其摘要算法有误,必须重新签名)。
   `build/` 下两个 launcher 构建(12:07 与 19:06)均已内嵌当前公钥,签名端
   与验签端一致。
2. **真正的 bug 在安装路径:即 BUG-03。** README 首选的本地安装流程是
   `node tools/install-slot/server.mjs` → 开发副本安装页。该副本(a)用
   第二次 `writeFlash` 向已含 MSIG 签名的同一 4KB 尾扇区写显示名 blob,
   把签名擦掉;(b)按分区大小计算 blob 地址,烧写到槽位分区之外。经它
   安装必然破坏签名扇区 → launcher 读到全 0xFF 的尾部 → 槽位被判定为未签名
   —— 尽管 .bin 文件本身的签名完全正确。

**结论:** 用当前 `sign-firmware.sh` 重新签名后,经(已修复的)开发页或
Cloudflare Pages 正式页安装;不要再用旧开发页此前烧写过的产物直接重装
而不重写尾扇区。BUG-03 修复后,该复现路径已被关闭。

**本地线刷已端到端调通(后续 2):** 本地线刷服务(`node
tools/install-slot/server.mjs`,服务规范 `install-slot/` 页面)冒烟通过
(页面、ES 模块、vendor 资源全部 200;SSRF 防护与 404 行为正确),并对真实
签名镜像完整回放了安装字节路径:`extractAppImage()` 解析出 image_len 962416 /
尾扇区偏移 0xEB000 / 完整 4KB tail sector;套用安装页的 MNAM 显示名补丁
(4056 处右对齐)后,组装出的槽位字节通过**固件验签代码**(`main/meta_sign.c`
+ 真实 mbedtls):`META_SIG_OK`、`meta_sign_detect_sector() == true`、彩蛋文本
完好。安装页 `tailSectorOffset`(image_len 后 4K 对齐)与设备端
`meta_sign_sector_offset()` 及设计文档 §7 布局完全一致
(MSIG [0..127] / MAEG [128..4055] / MNAM [4056..4095],尾扇区单次写入)。

验证命令:

```bash
tools/signing/run-verify-tests.sh ../pass-radar/build/pass-radar_v0.1-2-g8fcce59-signed.bin
node tools/install-slot/test-extract.mjs   # 安装页单测(规范模块)
bash tools/validate.sh --static            # 静态检查 + 主机测试,全部 PASS
PORT=4191 node tools/install-slot/server.mjs  # 本地线刷;浏览器打开 http://localhost:4191/
```

---

## BUG-01(高)— `add_battery()` 渲染出垃圾电量标签

**文件:** `main/main.c:64-71`
**状态:已修复** — 已按下述方案修复(`main.c:69` 加入 `soc < 0` 守卫);
代码块保留作缺陷记录。

```c
static void add_battery(lv_obj_t *parent)
{
    const int soc = bsp_battery_soc();
    char text[12];
    lv_obj_t *lbl = ui_pixel_label(parent, text, &lv_font_montserrat_14, UI_PAPER);
    lv_obj_set_pos(lbl, 204, 30);
}
```

**现象。** 标签由 `text` 构造,而 `text` 是**完全未初始化的栈缓冲**。
`ui_pixel_label()` 内部立即调用 `lv_label_set_text(label, text)`
(`ui_pixel.c:18-26`),会对其 `strlen()`:画出的是栈上的随机字节,若 12 字节
内没有 NUL 还会越界读取。注释承诺"读数 −1(不可用)时不画,避免显示假
数字"——`bsp_battery_soc()` 失败时确实返回 −1(`bsp_battery.h:13`)——但这个
返回值从未被使用。`soc` 成了死变量;主机测试链(`tools/validate.sh`)从不
编译 `main.c`,`-Werror` 因此从未发现。

**根因。** 半成品功能:SOC → 文本格式化和"不可用则不画"的守卫从未编写。
两处调用点都受影响——列表页(`main.c:143`)和详情页(`main.c:308`,该页已于
2026-09-23 移除),即最常用的两个页面右上角都是垃圾内容。

**修复方案。**

```c
static void add_battery(lv_obj_t *parent)
{
    const int soc = bsp_battery_soc();
    if (soc < 0) return;                       // 无电量计 → 不画
    char text[12];
    snprintf(text, sizeof(text), "%d%%", soc);
    lv_obj_t *lbl = ui_pixel_label(parent, text, &lv_font_montserrat_14, UI_PAPER);
    lv_obj_set_pos(lbl, 204, 30);
}
```

(确认已包含 `<stdio.h>` 以使用 `snprintf`。)

---

## BUG-02(高)— `HOST_TEST` 解析器的彩蛋 magic 判断反了

**文件:** `main/meta_sign.c:47`(`meta_egg_parse` 的 `#ifdef HOST_TEST` 变体)
**状态:已修复** — 上游提交已去掉多余的 `!`;该行现为
`if (memcmp(egg, egg_magic, 4) != 0) return META_EGG_ABSENT;`,与设备变体
及主机 stub 一致。代码块保留作缺陷记录。
**已加回归守卫:** `tests/test_meta_net_upload.c` 现已覆盖彩蛋路径 ——
`m1_egg_parse_valid`(有效 MAEG → `META_EGG_OK`)、`m2_egg_parse_absent`
(擦除态扇区 → `META_EGG_ABSENT`)、`m3_upload_signed_tail_preserved`
(含 MAEG 的已签名尾扇区在上传后完整保留,dispname 被拒)、
`m4_upload_unsigned_tail_rebuilt`(未签名上传重建 MNAM,无 MAEG 残留)。
旧的反转变体会在 m1/m3 立即失败。

```c
static const unsigned char egg_magic[4] = META_EGG_MAGIC_BYTES;
if (!memcmp(egg, egg_magic, 4)) return META_EGG_ABSENT;
```

**现象。** 与同一函数的其它所有实现语义相反:

- 设备变体 `main/meta_sign.c:97`:magic **缺失** 才返回 `META_EGG_ABSENT`
- 主机 stub `tests/esp_stubs/meta_sign_stub.c:58`:`memcmp(...) != 0` 返回 ABSENT
- 格式测试 `tests/test_meta_sign.c:158`:"擦除态(无 MAEG)→ `META_EGG_ABSENT`"

HOST_TEST 变体在 magic **匹配** 时返回 `META_EGG_ABSENT`,即有效彩蛋被报
为"无彩蛋",空白(0xFF)扇区反被当作"有彩蛋"进入解析。

**根因。** 多写了一个 `!`。注意这不是死代码:`tools/validate.sh` 用真实的
`main/meta_sign.c` 加 `-DHOST_TEST` 编译 `tests/test_meta_net_upload.c`
(validate.sh:73-78),每次 CI 都会构建该变体——只是上传测试尚未覆盖彩蛋
路径,所以一直没被发现。

**修复方案。** 与另外两处实现对齐:

```c
if (memcmp(egg, egg_magic, 4) != 0) return META_EGG_ABSENT;
```

更彻底的做法:删除 `main/meta_sign.c` 中的 HOST_TEST 块,让
`test_meta_net_upload` 链接 `tests/esp_stubs/meta_sign_stub.c` 的
`meta_egg_parse`/`meta_sign_verify`(与 `test_meta_sign` 的做法一致),
只保留一份实现,杜绝再次漂移。

---

## BUG-03(高)— 开发版安装页已漂移出正确性 bug

**文件:** `tools/install-slot/install-slot.html`、
`tools/install-slot/extract-app-image.js`(对照线上 `install-slot/` 副本)。
**状态:已修复** — 已把线上页的单次写入尾扇区流程与 `& 1` 位测试移植进
开发副本;`diff` 曾确认两份副本一致(仅 `name-blob.js` 头部一行路径注释
不同)。
**结构性修复(去重)也已完成:** 重复的开发页文件已删除,
`tools/install-slot/` 现仅保留 `server.mjs` 与 `test-extract.mjs`,
`server.mjs` 直接服务规范目录 `install-slot/`(`PAGE_DIR =
../../install-slot`),即 Cloudflare Pages 部署的同源字节。
`test-extract.mjs` 改为从规范目录导入模块、读取规范 HTML 做 i18n 测试。
`README.md`、`README.zh_CN.md` 与 `install-slot/README(.zh_CN).md` 的目录
结构说明已同步更新。验证:`node tools/install-slot/test-extract.mjs` →
8/8 PASS;`server.mjs` 冒烟(`/`、`/name-blob.js`、
`/extract-app-image.js`、`/vendor/esptool-js.js` 均 200)。

仓库维护两份安装页(README:152-153):`install-slot/` 是部署在 Cloudflare
Pages 的正式页;`tools/install-slot/` 是由 `server.mjs:91-94` 服务的本地
开发副本。没有任何机制校验两者一致,开发副本已落后出三处影响行为的差异。

### (a) 显示名 blob 写到槽位分区之外(破坏下一个分区)

开发副本 `tools/install-slot/install-slot.html:133,561-563`:

```js
import { packNameBlob, sanitizeDisplayName, blobOffset, maxAppImageSize } from "./name-blob.js";
...
const blob = packNameBlob(dispName);
const blobAddr = address + blobOffset(s.size);
```

`blobOffset(len)` = `ceil(len/4096)*4096 + 4056`(`name-blob.js:23-29`),
期望的入参是**镜像长度**,开发页却传入了**分区大小** `s.size`。`s.size`
是 4K 对齐的,于是 blob 落在

```
槽位起始 + part_size + 4056
```

即槽位末尾之外 4056 字节。以槽 0 为例(`ota_0`,0x180000 + 0x1D6000 =
0x356000 结束):blob 被烧写到 **0x356FD8 —— `cardid` NVS 分区内**
(0x356000,0x4000);对另两个槽,则落在下一个 app 分区的起始扇区。
即便按旧的尾扇区布局理解,这也是差了整整一个 sector("分区末尾"的 blob
本应在 `槽位起始 + part_size − 40`)。结果:显示名功能静默失效,**且安装
器破坏了无关分区**(设备的 card ID,或相邻槽位的前几个扇区)。

线上页已是正确的"按镜像长度 + 单次写入"流程
(`install-slot/install-slot.html:564-584`,`packNameBlobTail` 合入 4KB
`image.tailSector`,写到 `image.tailSectorOffset`)。

### (b) 双写擦除签名/彩蛋扇区

开发页先写 app 镜像,再发**第二次** `writeFlash` 写 blob(开发页:552-569)。
esptool 写 flash 前会擦除目标扇区,第二次调用落在第一次已写过的同一 4KB
尾扇区上,把先写入的内容抹掉。签名镜像的 MNAM 窗口与 MSIG 同扇区,因此
**经开发页安装的签名固件会丢失签名**,启动器把该槽位判定为未签名。线上页的注释正是为此而写("分次 writeFlash 会重复擦除同一
sector,把先写入的签名/彩蛋擦掉",`install-slot/install-slot.html:562-563`)
——修复从未同步回开发副本。

### (c) `hashAppended` 标志位判断漂移

`tools/install-slot/extract-app-image.js:29`:

```js
const hashAppended = buf[start + 23] === 1;              // 开发副本
const hashAppended = (buf[start + 23] & 1) === 1;        // 线上副本(第 29 行)
```

ESP 镜像头第 23 字节是标志字节(bit0 = hash_appended)。当前 ESP-IDF 只写
0 或 1,两者行为一致;但开发副本的相等判断在标志字节出现其它置位时会把
镜像长度算短 32 字节,导致烧写的 app 镜像被截断。开发副本应采用线上副本
的位测试。

**根因。** 代码复制且无同步机制:`tools/check_repo.py` 不比较两份副本,
`tools/install-slot/test-extract.mjs` 只测开发版 `extract-app-image.js`,
不与线上版对照;开发页 HTML 在引入尾扇区单次写入时没有同步更新。

**修复方案。**
1. 短期:把线上页的写入流程移植进 `tools/install-slot/install-slot.html`,
   位测试移植进其 `extract-app-image.js`。
2. 结构性:让 `tools/install-slot/server.mjs` 直接服务规范的
   `install-slot/` 目录(单一事实来源),或在 `tools/check_repo.py` / CI 中
   增加漂移检查:除头部注释行外两份副本必须逐字节一致。

---

## BUG-04(低)— 上传日志用 `%d` 打印 `size_t`

**文件:** `main/meta_net.c:407`
**状态:已修复** — 已改为 `%zu`,直接传 `size_t`。

```c
ESP_LOGI(TAG, "槽位 %d 写入成功: %s %s (%d B)", slot, name, ver, req->content_len);
```

`req->content_len` 是 `size_t`(ESP-IDF `httpd_req_t`;项目自己的 stub 也是
如此定义,`tests/esp_stubs/esp_http_server.h:32`)。在 32 位 ESP 目标上无害,
但在其它主机上是潜在的 `-Wformat` 告警,也与项目其它地方的 `%u` 风格不一
致。建议改为 `(unsigned)req->content_len` 配 `%u`。仅影响规范性。

---

## 已排查并排除的疑点(附证据)

- **LVGL 9.5 定时器自删除** — `lv_timer.c` 用 `act_timer_deleted` 守护回调,
  回调内删除自身定时器是受支持的行为,UI 拆除路径无问题。
- **彩蛋页程序化滚动** — `block()` 会剥掉自身子对象的
  `LV_OBJ_FLAG_SCROLLABLE`,但彩蛋面板(`main.c:189-190`)通过
  `lv_obj_set_scroll_dir()` 重新启用了滚动;`lv_obj_scroll_by_raw()` 根本不
  检查该标志。方向语义符合 LVGL 9.5 头文件契约(`dy > 0` 向开头滚动),
  `btn == BSP_BTN_UP ? +step : −step`(`main.c:440`)映射正确。
- **大小上限算术** — 对 `partitions.csv` 中 4K 对齐的分区大小(0x1D6000 /
  0x200000 / 0x29E000),`meta_sign_app_limit()`(`meta_sign.h:56`)、
  `meta_name_max_app_size()`(`meta_name.h:42`)与 JS `maxAppImageSize()`
  (`name-blob.js:33`)共用的 `part_size − 4096` 上限是**恰好紧**的:对任何
  被接受的 `image_len`,必有 `ceil(image_len/4096)*4096 ≤ part_size − 4096`,
  尾部 metadata sector 总放得下。无 off-by-one。
- **按键长按接线** — `bsp_button.c:72-76` 以栈上 `button_event_args_t`
  注册 `BUTTON_LONG_PRESS_START`(`press_time = BSP_BTN_LONG_MS (1500)`);
  `iot_button_register_cb` 返回前已把 `press_time` 拷入内部 cb_info 数组
  (`iot_button.c:378,402-411`),栈生命周期无问题。零初始化的
  `button_config_t` 使 `TIME_TO_TICKS(0, LONG_TICKS)` 回退到
  `CONFIG_BUTTON_LONG_PRESS_TIME_MS`(Kconfig 默认 1500ms),与显式参数一致。
- **`meta_seq` 匹配器** — 全路径推演(索引 > 0 时的间隔超时、按键失配重启、
  重复首键、无符号减法容忍回绕):行为与设计文档一致。
- **`meta_name_pack_tail`/`unpack_tail` ↔ JS `packNameBlobTail`** — 右对齐
  40B 窗口、xor 在末字节;C 与 JS 逐字节一致。
- **`meta_egg_parse` 窗口边界** — 编译期检查
  (`META_EGG_WINDOW_OFF + META_EGG_TOTAL_LEN ≤ META_NAME_BLOB_OFF`,
  即 128 + 3928 ≤ 4056),与 `sign-firmware.sh` 的布局(MSIG@0、MAEG@128、
  MNAM@4056)一致。
- **`image_len` 来源** — 上传路径用 `esp_image_verify` 元数据,正确避开了
  "镜像头 +20 处不是长度字段"的坑(`meta_net.c:314-318`)。
- **线上安装器的尾扇区提取** — 将 MSIG/MAEG/MNAM 提取进同一个
  `image.tailSector` 并单次 `writeFlash` 写入,正确。

---

*报告基于静态审查 + 主机测试源码验证。行号对应当前分支。姊妹文档:
[BUGS.md](BUGS.md)。*
