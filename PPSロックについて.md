
### 1. 屋外なら PPS は **ほぼ常時ロック**できる

|条件|現実的な可用率|
|---|---|
|**空が 45° 以上開けた場所**（ビル影・樹木なし）|**99 % 以上**（昼夜問わず）|
|軽く遮られる（車内・ベランダ手すり程度）|95 % 前後：DOP が悪化した瞬間に数秒外れることがある|
|大型車両が横付け、激しい豪雨|数 分単位で外れる可能性|

“ロック”＝**chrony が PPS ソースを選択 (`*`) している状態**。衛星側の UTC ⇄ GPST 変換は整数秒なので失われません。可用率を落とす主因は **視界遮蔽と給電ノイズ** です。

---

## 2. PPS ロックで動かすまでの完全手順（Pi 3B 例）

> **想定 OS**: Raspberry Pi OS Bookworm Lite  
> **GPS モジュール**: u-blox M8N（1 PPS 出力可）  
> **GPIO**: PPS → GPIO 18 (pin 12)

### 2-1 ハード配線チェック

```text
    Pi 3B Pin            GPS モジュール
    ──────────────────────────────────
    5 V (pin 4)   ──▶ VCC
    GND (pin 6)   ──▶ GND
    GPIO15 RXD    ◀─ TXD
    GPIO14 TXD    ──▶ RXD
    GPIO18        ◀─ 1 PPS
```

- PPS 出力が **3.3 V かつパルス幅 ≥10 µs** を確認。
    

### 2-2 /boot/config.txt 設定

```ini
enable_uart=1
# Bluetooth を miniUART へ追い出し、PL011 を GPS に
dtoverlay=miniuart-bt
core_freq=250

# PPS GPIO
dtoverlay=pps-gpio,gpiopin=18,assert_falling_edge=true
```

### 2-3 シリアルコンソール無効化

```bash
sudo sed -i 's/console=serial0,[0-9]* //g' /boot/cmdline.txt
sudo systemctl disable --now serial-getty@ttyS0.service
```

### 2-4 パッケージ

```bash
sudo apt update
sudo apt -y install gpsd gpsd-tools pps-tools chrony
```

`/etc/default/gpsd` を編集:

```text
DEVICES="/dev/ttyAMA0 /dev/pps0"
GPSD_OPTIONS="-n"
```

### 2-5 chrony の PPS ロック設定

`/etc/chrony/chrony.conf` 末尾に追記:

```text
# 秒ラベル用に NMEA を入れる
refclock SHM 0 offset 0.0 delay 0.2 refid NMEA

# 位相制御は PPS、NMEA をロック対象に
refclock PPS /dev/pps0 lock NMEA refid GPS

maxslewrate 1000
smoothtime 400 0.001 leaponly
```

> `lock NMEA` が **「NMEA が来てる秒ラベルを PPS にひも付けろ」** という “PPS ロック” の肝。

### 2-6 再起動 & 動作確認

```bash
sudo reboot
# ─────────────── after login ───────────────
gpsmon -n          # NMEA 流入を確認
sudo ppstest /dev/pps0   # 1 秒ごとに assert → OK

chronyc sources -v
```

```
MS Name/IP   Stratum Poll Reach LastRx Last sample
============================================================================
#* PPS       0       4    377     3   -0.2ns[+14ns] / 0.4µs
#? NMEA      0       4    377     2   +68ms[+68ms] / 1.3ms
```

- `*` が PPS 行に付いたら **PPS ロック成立**。
    
- 数分で `Last sample` の角括弧 […] 部が ±数 µs → ±数 100 ns に収束します。
    

---

## 3. ロック維持のコツ（屋外設置）

|施策|効果|
|---|---|
|**広い空視界**（最低 120 ° 開ける）|DOP 安定 → PPS 途切れを防ぐ|
|**アンテナを接地メタル板に載せる**|C/N₀ +3 dB → 仰角低い衛星も捕捉|
|**5 V/USB 電源を 2 A 以上のノイズ低い物に**|PPS 立ち上がりジッタ低減|
|**`minpoll 4 / maxpoll 4`** に固定|chrony が毎 16 s で PPS 位相差を統計処理 → 安定収束|
|**RTC 併設 (DS3231)**|再起動直後でも offset < ±100 ms に抑え、ロック時間短縮|

---

## 4. ロック状態を自動監視するワンライナー

```bash
watch -n 5 'chronyc sources | awk "/^#\*/ {print strftime(), \"LOCK OK\", $0} END {if(NR==0) print strftime(), \"NO LOCK\"}"'
```

- `LOCK OK` が出続ければ PPS ロック継続。
    
- `NO LOCK` を検知したらメール通知や LED 点灯などに回せば現場でも安心です。
    

---

### TL;DR

1. **屋外で空が開けていれば PPS は 99 % 以上ロックしっぱなし**。
    
2. **核心は `/etc/chrony/chrony.conf` の**  
    `refclock PPS /dev/pps0 lock NMEA`  
    これが “PPS ロックで動かす” スイッチ。
    
3. `chronyc sources` で PPS 行に `*` が付いていれば、もう μs～ns オーダで同期しています。