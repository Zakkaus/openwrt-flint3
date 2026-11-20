# GL.iNet Flint 3（GL-BE9300）的 OpenWrt

[English](README.md)

本倉庫是 [perceival/openwrt-flint3](https://github.com/perceival/openwrt-flint3) 的 fork。`flint3` 分支是 OpenWrt main 加上這個移植版與我們的修正，以合併 OpenWrt main 的方式同步。隨 OpenWrt main 進來的 GitHub Actions workflow 在此已停用：它們建置的是通用目標，不是這個裝置。

為 **GL.iNet Flint 3（GL-BE9300）** 提供 OpenWrt 主線支援：Qualcomm **IPQ5332**（四核心 Cortex-A53）、三頻 Wi-Fi 7、Realtek **RTL8372N** 10G 交換器，以及 **RTL8221B** 2.5G WAN PHY。

> **這個分支（`flint3`）是完整、可建置的 OpenWrt 原始碼樹。**直接 clone 就能建置，不需要放進另一份檢出。（先前的 `main` 分支放的是目標 *overlay*；該分支已停用，保存在標籤 `archive/main-overlay`。）

目標：**`qualcommbe/ipq53xx`**，核心 **6.18**。

> [!WARNING]
> **非官方、由社群維護的移植版，與 GL.iNet 及 OpenWrt 專案無關，未經其背書或支援。**按**現狀提供，不附任何形式的保固**。刷寫第三方韌體有實際風險，包括讓裝置無法開機，也可能使硬體保固失效。**先備份 eMMC**——ART 分割區存放這台機器專屬的無線電校準資料與 MAC 位址，別處無法還原。若這台路由器對你很重要，先在一台備用機上測試再依賴它。

## 硬體

| 區塊 | 內容 |
|---|---|
| SoC | Qualcomm IPQ5332，4× Cortex-A53 |
| Wi-Fi 2.4 GHz | SoC 內建無線電，ath12k over AHB |
| Wi-Fi 5 / 6 GHz | 2× QCN9274，ath12k over PCIe |
| 交換器 | RTL8372N，樹外 DSA 驅動程式（`realtek,rtl837x`）；SoC↔交換器連結為 10GBASE-R |
| WAN | RTL8221B 2.5G，USXGMII |
| 儲存 | eMMC |

## 狀態

| 子系統 | 狀態 |
|---|---|
| 開機 / procd / SSH | 運作中 |
| LAN（RTL8372N 經 DSA + EDMA/PPE） | 運作中 |
| WAN（2.5G，USXGMII） | 運作中，以 2.5 Gbps 建立連結 |
| VLAN（DSA 上的 bridge-vlan） | 運作中 |
| PPE 硬體 flow offload | IPv4 LAN→WAN NAT（TCP/UDP，WAN 為 untagged 或 802.1Q）約 2.3 Gbit/s、CPU 約 1%；需在防火牆啟用硬體 flow offloading 才會生效。WAN→LAN 與 IPv6 留待下一個發行版 |
| Wi-Fi 7，三個頻段 | 運作中 |
| MLO（跨 2.4/5/6 GHz 的 AP MLD） | 運作中 |
| DFS | 運作中，包含同一 DFS 無線電上的多個 BSS 一起啟動 |
| 802.11k / 802.11v | 運作中 |
| eMMC sysupgrade + 回到原廠 | 運作中 |

在兩台機器之間、經 2.5G 主幹測得的吞吐量：**約 1.8–1.9 Gbit/s**。

## 已知問題

- **ath12k 韌體在持續負載下卡住。** 連線用戶端很多、執行數小時後，Q6 可能發生致命錯誤。自 2026-09-28 起，韌體 coredump 會自動釋放，無線電在數秒內恢復，不再一直停擺；當機本身的原因仍未查明。已回報上游。
- **netlink socket 釋放時發生核心 panic**，自 8 月起在兩台 AP 上共見到六次，都在開機數小時後（socket 來自 bridge 通知、hostapd 或 wsdd2）。AP 約 90 秒後自行重開機（panic 紀錄會保留，見[當機紀錄](#當機紀錄)）。wsdd2 已持續停用，作為可能的觸發來源之一；根本原因正以 KASAN 核心調查中。
- **用戶端因 `excessive missing ACKs` 被踢除**：驅動程式回報的封包遺失事件對多鏈路 station 不可靠，所以這些映像檔中 hostapd 的 `disassoc_low_ack` 預設為 0（在 wifi-iface 上設回 1 可恢復舊行為）。
- **PPE WAN RX FIFO 溢位。** 在約 1.9 Gbit/s 時約有 0.07–0.09 % 的封包。已不再是早期建置約 600 Mbit/s 的硬性上限，但仍非零。
- **802.11r 與 MLO 不相容。** hostapd 的 FT 程式碼不認得 MLD——不要在 MLO SSID 上啟用 11r。11k/11v 沒有問題。

## 測試原廠 2.4 GHz board file

映像檔使用的 2.4 GHz board data 已知能在這個韌體上運作。GL 自己的韌體為這塊板子載入另一個 board file（`bdwlan.b12`），尚未在 ath12k 韌體上試過。要測試它，需要選用套件 `ipq-wifi-glinet_gl-be9300-bdf-test`。預設建置既不包含也不建置它，所以先在 `make menuconfig` 中選取（Firmware, ath10k Board-Specific Overrides）：選 `<*>` 放進映像檔，或選 `<M>` 只建置套件，再把建置目錄 `packages` 下的 `.apk` 複製到機器上，對它執行 `apk add --allow-untrusted`。它會安裝一個 `board-2.bin` 容器，裡面是那個 board file，board 名稱與 regdb 沿用正常版本，路徑為 `/lib/firmware/ath12k-test/ath12k/IPQ5332/hw1.0/board-2.bin`，除非選取，否則沒有任何程式會載入它。要選取到下次重開機為止：

```sh
wifi down
echo -n /lib/firmware/ath12k-test > /sys/module/firmware_class/parameters/path
rmmod ath12k_wifi7 ath12k
modprobe ath12k && modprobe ath12k_wifi7
sleep 30; wifi up
```

接著確認 2.4 GHz 網路能啟動（用 `iwinfo`，並確認 `dmesg | grep ath12k` 沒有 board-data 或韌體錯誤），再與正常檔案比較 `iw dev <2.4 GHz interface> info`（txpower）與 2.4 GHz 吞吐量。flash 上不會有任何變更：Wi-Fi 若沒回來，直接重開機就會再使用正常的 board file。請在 issue tracker 回報結果。

## 建置

```sh
git clone -b flint3 https://github.com/Zakkaus/openwrt-flint3.git
cd openwrt-flint3
./scripts/feeds update -a
./scripts/feeds install -a
make menuconfig     # Target System: Qualcomm Atheros 802.11be
                    # Subtarget:     ipq53xx
                    # Target Profile: GL.iNet GL-BE9300
make -j"$(nproc)"
```

映像檔輸出到 `bin/targets/qualcommbe/ipq53xx/`。

### 不想從原始碼建置？

**[Releases 頁面](https://github.com/Zakkaus/openwrt-flint3/releases)** 上的每一個發行版都是一次路由器建置，由本 fork 的 `configs/router.config` 建出，包含 LuCI。發行版附上 factory 與 sysupgrade 映像檔，以及 `sha256sums` 檔案。

最新的是 `flint3-20261120-2`，由 fork 提交 `df5e099e03` 建出，時間在合併 OpenWrt main 之前。

已簽章的 apk 套件 feed 含核心模組，與發行映像檔相符：

- 本 fork：**https://zakkaus.github.io/flint3-packages/openwrt/**
- ImmortalWrt fork：**https://zakkaus.github.io/flint3-packages/immortalwrt/**

核心模組只能安裝到同一次建置的映像檔上，所以先把路由器升級到該映像檔。LuCI 的繁體中文翻譯（`luci-i18n-*-zh-tw`）在 feed 裡，不在映像檔中。

上游 [perceival/openwrt-flint3](https://github.com/perceival/openwrt-flint3) 發布的 `vanilla`、`ap`、`router` 三種版本，這裡不發布。

刷寫前請先讀上面的免責聲明。

## 安裝

完整、經硬體驗證的說明——包括回到原廠的來回流程——在裝置頁面：

**https://openwrt.org/toh/gl.inet/gl-be9300**

簡短版：從原廠韌體安裝時，用 **factory** 映像檔搭配 `sysupgrade -F -n`。必須加 `-F`，因為原廠不認得映像檔的 board 名稱（見下文）；`u-boot`/`tz`/`sb11` 的 `missing section` 警告是預期的。**不要**從原廠強制刷入單純的 sysupgrade 映像檔。

`-n` 是**必要**參數，不是選用。少了它，原廠自己的 `platform_copy_config` 會在 `rootfs` 分割區內格式化 ext4 檔案系統，並把 GL 的設定複製進去，OpenWrt 之後會把它當成自己的 overlay 接手。若檔案不是 QSDK FIT（例如誤刷了 sysupgrade 映像檔），它會從偏移 0 開始格式化並摧毀原廠 rootfs，此時只有 U-Boot 救援能救回機器。強制刷寫前，先拿映像檔的 `sha256sum` 對照該次建置的 `sha256sums` 檔案。

從 OpenWrt 回到原廠：`sysupgrade -n be9300-<version>.bin`（不需要 `-F`；這裡同樣必須加 `-n`）。

原廠 QSDK 韌體可能回報 board 名稱為 `qcom,ipq5332-ap-mi01.6`。那是廠商路徑使用的通用 Qualcomm MI01.6/RDP468 身分，不是 OpenWrt 的 GL-BE9300 裝置識別碼。上游 Qualcomm RDP468 device tree 也使用同一個 compatible，所以刻意不把它加進這個 profile 的 `SUPPORTED_DEVICES`：加進去等於宣稱 Flint 3 映像檔與其他使用該通用身分的硬體相容。因此原廠出現相容性警告是預期的；請改用文件記載的 `-F` factory 映像檔流程（見[issue #9](https://github.com/perceival/openwrt-flint3/issues/9)）。

**先備份 eMMC**——ART 分割區存放這台機器的無線電校準與 MAC 位址，別處無法還原。

**從較舊建置升級並保留設定。** 較舊的映像檔會把每個 AP 固定在 `/etc/config/wireless` 中由 ART 衍生的位址上。第一次開機時，`99-gl-be9300-wifi-macs-cleanup` 會移除這些 `macaddr` 選項，但只移除該腳本寫入的確切值、且順序必須與寫入時相同，還要求至少前兩個值存在。每次移除都會記錄在 `dmesg`。其他情況都視為你自己設定的而保留，包括只有一個 AP 固定在 ART 位址，或在你新增、移除、重排 AP 之後才固定的位址。這段只執行一次：`/etc/gl-be9300-wifi-macs-cleaned` 會在之後的升級中保留，阻止它再次執行，所以你之後設定的位址會留下。若要讓某個仍保留位址的介面回到預設值：

```sh
uci delete wireless.<iface>.macaddr
uci commit wireless
wifi reload
```

## 救援

由侵入性最低排到最高：

1. **設定壞掉、系統能開機：** OpenWrt 執行中按住 reset 鍵超過 5 秒後放開。這是出廠重置（`firstboot`），機器會回到 192.168.1.1。
2. **系統無法完成開機：** failsafe。接通電源並觀察白色 LED。開機初期閃爍時短按 reset，它會轉為快速閃爍。把 PC 以靜態 192.168.1.2/24 接到 **LAN1**，執行 `ssh root@192.168.1.1`。接著用 `mount_root` 存取設定，或執行 `firstboot -y && reboot`。
3. **完全沒有核心（刷寫失敗）：** U-Boot 救援伺服器。關機，按住 reset，開機並持續按住約 5 秒。接著 192.168.1.1 會有網頁伺服器回應（PC 設為 192.168.1.2/24）。從https://dl.gl-inet.com/router/be9300/ 下載 **GL.iNet 原廠韌體** `be9300-<version>.bin` 並上傳，讓它刷寫並重開進原廠，再照上文從原廠重新安裝 OpenWrt。同一個伺服器也有可寫入 U-Boot 本身與 `0:ART` 分割區的上傳表單，既無驗證也無檢查。絕對不要用那些表單：ART 存放這台機器的校準資料與 MAC。

救援伺服器處理*非原廠*映像檔的行為是 GL 自己的程式碼（它依據的 Qualcomm U-Boot 沒有網頁伺服器），而 GL 的韌體映像檔不附 U-Boot，所以只能從機器的開機載入程式讀出來：

- `…-squashfs-factory.bin` 帶有與 GL 映像檔相同的 U-Boot 腳本（`flashinit mmc; flupdate set mmc; imxtract … hlos && flash 0:HLOS; imxtract … rootfs && flash rootfs; flupdate clear`；在 Qualcomm 的 U-Boot 中 `flupdate` 只選擇 flash 類型）。若伺服器執行它，映像檔就會被刷入。這點是從 PR #114 中的開機載入程式回報得知，尚未測試。
- `…-initramfs-uImage.itb` **不會**被刷入。伺服器回報 `HTTP ugrade is done`，但 eMMC 沒有改變。映像檔留在 RAM 的 0x50000000，可從序列埠主控台執行 `bootm 0x50000000` 啟動（見裝置頁面）。這是裝置頁面實測的結果，與 PR 對 `.itb` 的分析一致。
- `…-squashfs-sysupgrade.bin` 會被拒絕。

要確保結果，就在那裡使用原廠映像檔。

### U-Boot 環境（`fw_printenv`）

映像檔為 GL-BE9300 設好 `fw_env.config`，沿用原廠的配置：`0:APPSBLENV` 位於偏移 0、大小 256 KiB，即 Qualcomm 為 IPQ5332 提供的 U-Boot 環境大小。這已在執行原廠 4.10.1 的機器上驗證：`0:APPSBLENV` 為 256 KiB，原廠自己的 `/etc/fw_env.config` 內容是 `/dev/mmcblk0p6 0x0 0x40000`，且 `fw_printenv` 沒有 `Bad CRC` 警告。要在自己的機器上重做這項檢查，**趁它仍在執行原廠韌體時**執行：

```sh
cat /etc/fw_env.config      # must read: /dev/mmcblk0p6 0x0 0x40000
fw_printenv >/dev/null      # must print no "Bad CRC" warning
```

這個項目只在全新安裝時建立（或刪除 `/etc/config/ubootenv` 之後）；從較舊建置保留設定升級時，仍會保留原本的空設定。要在執行中的機器上手動加入：

```sh
echo "$(. /lib/functions.sh; find_mmc_part 0:APPSBLENV) 0x0 0x40000" > /etc/fw_env.config
fw_printenv >/dev/null      # again: no "Bad CRC"
```

**`fw_printenv` 警告 `Bad CRC, using default environment` 之後，絕不要執行 `fw_setenv`。** `fw_setenv` 會把工具內建的通用預設環境（`bootcmd=run distro_bootcmd`）以正確的 CRC 寫入，這個 U-Boot 之後就無法啟動韌體。請改為移除 `/etc/fw_env.config`，並回報上面原廠檢查的輸出。

## 升級失敗

升級一旦開始，ssh 與系統紀錄就沒了；若失敗，機器會立刻重開機，原因只會顯示在序列埠主控台。腳本也會把原因寫進核心紀錄。若機器回到舊系統，請在下次重開機前查看 `/sys/fs/pstore/console-ramoops-0`。這尚未在硬體上試過。升級開始前就能發現的失敗，例如 RAM 不足或缺少工具，除了加 `-F` 的情況外，`sysupgrade` 本身都會回報。

## 當機紀錄

核心 panic 會在 `kernel.panic`（3 秒）後重開機。若 panic 路徑反而卡住，watchdog 最遲 5 秒後觸發。當機的 AP 為何要約 90 秒才再次回應，無法從原始碼判定。大部分時間大概是正常開機（U-Boot、核心，以及為三個無線電載入 Wi-Fi 韌體），但卡住直到 watchdog 觸發的 panic 路徑從外觀看也一樣。保留的當機紀錄時間戳或序列埠主控台可以分辨。

當機的核心紀錄會留在 RAM 中撐過那次重開機（ramoops，在 0x4da00000 保留 1 MiB）。下次開機時 `/etc/init.d/save-pstore` 會把它複製到 `/root/pstore/NNNN-<date>/`（`dmesg-ramoops-*` 是 panic，`console-ramoops-*` 是當機那次開機的主控台輸出），保留最近 10 次當機，並在核心紀錄中說明（`dmesg | grep pstore`，`logread` 也看得到）。只有在每一筆紀錄都複製到以 flash 為後端的 overlay、同步並與原檔比對之後，RAM 中的副本才會清除。若 overlay 不在 flash 上（例如因當機受損而退回 tmpfs），或複製失敗，紀錄會留在 `/sys/fs/pstore`，可以手動複製。`/root/pstore` 在保留設定的 sysupgrade 之間會保留。斷電，或不等重開機就拔掉電源，會失去 RAM 中的副本。

只有在 TCSR（0x193D100，bit 4）的 download-mode cookie 被設定時，U-Boot 才會收集自己的 crash dump（並等待 `any key within 10s`）。Linux 在開機時清除它（`qcom_scm.download_mode=off`，預設值），所以 panic 後會直接重開進正常開機。

## 其他板子

這裡只測試 GL-BE9300。這棵樹是 `qualcommbe/ipq53xx` 目標，所有非板子專屬的部分—— IPQ5332 時脈、PCIe、PPE/EDMA 乙太網路與硬體卸載、ath12k Wi-Fi 與韌體復原處理常式——都由每個 IPQ53xx 裝置共用。新板子需要的是 device tree、映像檔定義，以及（若它的交換器不是 Realtek RTL837x）相符的交換器驅動程式。

**定義在這棵樹中，我尚未測試**（不發布它們的映像檔）：

| 板子 | SoC | 交換器 | Flash | 來源 | 備註 |
|---|---|---|---|---|---|
| GL.iNet GL-BE6500 | IPQ5332 + QCN9274 | RTL837x（同一驅動程式） | NAND（UBI） | [JiaY-shi](https://github.com/JiaY-shi/openwrt) | 最接近的親戚；可從這棵樹建置 |
| Ubiquiti UniFi 7 Pro XGS | IPQ5332 | 無（單一 10G PHY） | eMMC + SPI-NOR | Til Kaiser，上游 [#25185](https://github.com/openwrt/openwrt/pull/25185) | 需要降級開機載入程式（較新版本會強制驗證簽章） |

**其他有社群成果的 IPQ53xx 裝置**（不在這棵樹中）：

| 裝置 | SoC | 交換器 | 狀態 |
|---|---|---|---|
| Xiaomi BE3600 Pro（RN01） | IPQ5312 | Motorcomm YT9215S | 在上游 ipq53xx PR 之上移植（乙太網路、儲存、開機據回報可運作）— [#23161](https://github.com/openwrt/openwrt/pull/23161) |

**上游：** OpenWrt main 目前不支援 IPQ53xx。這個 subtarget 提案在[openwrt/openwrt#23161](https://github.com/openwrt/openwrt/pull/23161)（尚未合併）。這裡使用的 RTL8372N 交換器驅動程式只存在於這棵樹。

**徵求測試者。** 若你擁有 GL-BE6500、UniFi 7 Pro XGS 或其他 IPQ53xx 裝置，並熟悉序列埠主控台與 eMMC/NAND 備份，希望能與你聯絡：開一個 issue，附上型號、原廠開機紀錄與板子照片。沒人能測試的板子支援不會發布，多一雙手是改變這點最快的方法。在這個分支上新增其他板子支援的 PR 也歡迎。

## 上游

這項工作已進入上游或正在審查的修補：

- `wifi: ath12k: advertise AP_VLAN interface mode for IPQ5332`（linux-wireless）
- hostapd WDS/AP_VLAN `bss->ctx` 修正（由 Jouni Malinen 套用）
- ath12k `hw_scan` NULL-deref 回報（與 Qualcomm 開發團隊處理中）

## 連結

- 論壇討論串：https://forum.openwrt.org/t/gl-inet-flint-3-exploration-gl-be9300-ipq5332/250267
- 裝置頁面：https://openwrt.org/toh/gl.inet/gl-be9300
- Q6/PAS 研究筆記：[`2.4GHZ-Q6-PAS-FINDINGS.md`](2.4GHZ-Q6-PAS-FINDINGS.md)

## 致謝

建構在 [JiaY-shi](https://github.com/JiaY-shi/openwrt) 的 GL-BE6500 樹上，它提供了可用的 IPQ5332 Wi-Fi 與 RTL837x DSA 基礎。感謝所有透過 issue tracker 與論壇討論串提供硬體發現與測試的人。
