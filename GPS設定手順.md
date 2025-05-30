### 1 | まずは略語（ざっくり辞書）

| 略語                   | 意味／役割                                                          | ひとことメモ                                                    |
| -------------------- | -------------------------------------------------------------- | --------------------------------------------------------- |
| **GPS / GNSS**       | Global Positioning System / Global Navigation Satellite System | 全衛星測位システムの総称。今回は「時刻を貰う」ために使う。                             |
| **QZSS**             | Quasi-Zenith Satellite System（みちびき）                            | 日本上空に長く居る測位衛星。受信できると補足が速い。                                |
| **1 PPS**            | 1 Pulse-Per-Second                                             | GPSモジュールが毎秒出す基準パルス（±10 ns～）。NTPにこれを食わせると秒単位の位相ずれがほぼゼロになる。 |
| **UART**             | Universal Async Receiver Transmitter                           | TX/RX 2 本でシリアル通信。GPSモジュールの NMEA 文を Pi へ流す。                |
| **PL011 / miniUART** | Pi 本体にある 2 種類の UART                                            | 3B は BT 用に PL011 が取られているので要設定。                            |
| **NMEA**             | GPS から流れてくる ASCII データフォーマット                                    | 緯度経度や時刻などを “$GPGGA,…” の形で出す。                              |
| **PPS-GPIO**         | Linux カーネルの PPS フロントエンド                                        | GPIO ピンを PPS として扱うドライバ。                                   |
| **gpsd**             | GPS データを扱うデーモン                                                 | UART + PPS をまとめて chrony へ渡す。                              |
| **chrony**           | 高性能 NTP サーバ／クライアント実装                                           | gpsd の時刻・PPS を元にシステムクロックを校正。                              |
| **RTC / DS3231**     | Real-Time Clock                                                | 衛星が捕まる前のバックアップ時計（今回はおまけ）。                                 |

出典 : 記事全体で使われている用語から要約

---

## 2 | 流れを大きく分けると

1. **ハードをつなぐ**  
    GPS モジュール（電源・TX/RX・1 PPS）を 40 pin ヘッダに配線。Pi 3B なら基本は同じピン番号（PIN 4 = 5 V, 6 = GND, 8 = TXD, 10 = RXD, 12 = GPIO 18 → PPS）
    
2. **OS 側で UART & PPS を有効化**
    
    - `/boot/config.txt` に `enable_uart=1` と `init_uart_baud=9600` を追加。
        
    - **Pi 3B 固有**:
        
        - _Bluetooth を殺す_ → `dtoverlay=disable-bt` **または**
            
        - _BT を miniUART へ追い出す_ → `dtoverlay=miniuart-bt` ＋ `core_freq=250`
            
    - シリアルコンソールを外す（`console=serial0,…` を消す）。
        
    - PPS 用 overlay → `dtoverlay=pps-gpio,gpiopin=18,assert_falling_edge=true`。
        
3. **gpsd を入れて動作チェック**
    
    - `sudo apt install gpsd gpsd-tools pps-tools`
        
    - `gpsmon -n /dev/ttyAMA0` で NMEA が流れるか確認。
        
    - `ppstest /dev/pps0` で 1 PPS が刻むか確認。
        
4. **gpsd をサービス化**  
    `/etc/default/gpsd`
    
    ```text
    GPSD_OPTIONS="-n"
    DEVICES="/dev/ttyAMA0 /dev/pps0"
    ```
    
    → `systemctl restart gpsd`
    
5. **chrony でシステムクロックを合わせる**  
    `/etc/chrony/chrony.conf` に
    
    ```text
    pool ntp.nict.jp iburst minpoll 6 maxpoll 8   # 外部参照（好みで）
    refclock PPS /dev/pps0 lock NMEA refid GPS    # PPS を基準に
    leapsecmode slew
    maxslewrate 1000
    smoothtime 400 0.001 leaponly
    ```
    
    → `systemctl restart chrony` → `chronyc sources -v` で `GPS` 行に `*` が付けば勝ち。
    
6. **（任意）RTC 追加**  
    DS3231 を I²C へ配線し `dtoverlay=i2c-rtc,ds3231`。Chrony が 11 分ごとに hwclock を書くのでバックアップ完了。
    

---

## 3 | Pi 3B ならではの要点

| 項目            | Pi 4B（記事）                                    | **Pi 3B**                                                                                                                   |
| ------------- | -------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Bluetooth     | USB-接続                                       | SoC 内蔵で PL011 UART を占有                                                                                                      |
| デフォルト UART ピン | `serial0` = PL011                            | `serial0` = **miniUART**                                                                                                    |
| 推奨設定          | そのまま `enable_uart=1`                         | **どちらか選ぶ**① `dtoverlay=disable-bt` → BT 無効・GPIO は PL011 へ② `dtoverlay=miniuart-bt` → BT を miniUART へ移し、`core_freq=250` で安定化 |
| config ファイル場所 | `/boot/firmware/config.txt`（Bookworm 64-bit） | Bookworm 32-bit Lite でも **同じパス**。古い環境なら `/boot/config.txt`                                                                  |
|               |                                              |                                                                                                                             |

---

## 4 | 手順を“実装書”としてまとめる（Pi 3B 用）

> **ゴール**: Pi 3B を単体 GPS-NTP サーバにする（RTC なし構成）。  
> **想定 OS**: Raspberry Pi OS Bookworm Lite 32-bit（最新 Imager で焼く）。

### 4-1 ハード配線

|GPS モジュール|Pi 3B 40 pin|備考|
|---|---|---|
|VDD 5 V|PIN 4 (5 V)|5 V 電源|
|GND|PIN 6 (GND)||
|TXD|PIN 10 (GPIO 15 / RXD)||
|RXD|PIN 8 (GPIO 14 / TXD)||
|1 PPS|PIN 12 (GPIO 18)|1 PPS = Low 100 ms|

### 4-2 OS 初期設定

```bash
# SSH でログイン後
sudo raspi-config nonint do_change_timezone Asia/Tokyo
sudo raspi-config nonint do_boot_behaviour B2  # CLI オンリー
```

### 4-3 /boot/config.txt 追記

```bash
sudo tee -a /boot/config.txt <<'EOF'
################################
# GPS 時刻同期 (RPI3B)
enable_uart=1
init_uart_baud=9600

# === ここはどちらか一方 ===
dtoverlay=disable-bt               # ① BT 要らない派
#dtoverlay=miniuart-bt            # ② BT 残したい派
#core_freq=250                     # ↑②を選んだ場合に必須
# =============================

dtoverlay=pps-gpio,gpiopin=18,assert_falling_edge=true
EOF
```

### 4-4 シリアルコンソール無効化

```bash
sudo sed -i 's/console=serial0,[0-9]* //g' /boot/cmdline.txt
sudo systemctl disable --now serial-getty@ttyS0.service
```

### 4-5 モジュールロード

```bash
echo pps-gpio | sudo tee -a /etc/modules
```

### 4-6 再起動してデバイス確認

```bash
sudo reboot
# 戻ったら
ls -l /dev/ttyAMA0   # 存在すれば OK
ls -l /dev/pps0      # 同上
```

### 4-7 gpsd + ツール導入

```bash
sudo apt update
sudo apt -y install gpsd gpsd-tools pps-tools
```

`/etc/default/gpsd` を以下のように：

```bash
sudo sed -i 's|DEVICES=""|DEVICES="/dev/ttyAMA0 /dev/pps0"|' /etc/default/gpsd
sudo sed -i 's|GPSD_OPTIONS=""|GPSD_OPTIONS="-n"|' /etc/default/gpsd
sudo systemctl restart gpsd.socket gpsd.service
```

動作確認：

```bash
gpsmon -n      # 衛星が捕まると経度緯度が動く
sudo ppstest /dev/pps0   # 1 秒ごとに assert が出れば OK
```

### 4-8 chrony 導入

```bash
sudo apt -y install chrony
```

`/etc/chrony/chrony.conf` 末尾に追記：

```text
refclock PPS /dev/pps0 lock NMEA refid GPS
leapsecmode slew
maxslewrate 1000
smoothtime 400 0.001 leaponly
```

**外部 NTP も置き換えたい場合**（NICT 推奨）:

```bash
sudo sed -i 's|pool .*debian.*|pool ntp.nict.jp iburst minpoll 6 maxpoll 8|' /etc/chrony/chrony.conf
```

サービス再起動：

```bash
sudo systemctl restart chrony
```

状態確認（3-5 分後）：

```bash
chronyc sources -v
# "GPS" 行に '*' が付いて offset ±数 µs → 成功
```

### 4-9 動作チェックまとめ

```bash
timedatectl | grep -E 'System clock|NTP service'
# → yes / active
```

これで **Pi 3B が µs レベルで GPS 時刻にロックされた NTP サーバ** になりました。ローカル LAN 内の PC や別の Raspberry Pi から

```bash
sudo timedatectl set-ntp false   # 既存 NTP を切る場合
sudo ntpdate -q <Pi3B_IP>
```

でオフセットを確認してみると、十数 µs 程度まで追い込まれているはずです。

---

### 5 | トラブルシューティング早見表

|症状|チェックポイント|
|---|---|
|`/dev/ttyAMA0` が無い|`enable_uart=1` が効いているか／BT overlay 設定ミス|
|NMEA は来るが PPS デバイスが出ない|`dtoverlay=pps-gpio,…`／配線（GPIO 18）|
|`chronyc sources` で `?` のまま|gpsd が起動していない／DEVICES 設定漏れ|
|Pi が数分で落ちる|5 V 電源不足（GPS モジュールは 150 mA くらい食う）|

---

これで **Raspberry Pi 3B 版 GPS 1 PPS NTP サーバ** の完成！　屋根裏など空が見える場所に置けば、研究室全体の時計基準として十分な精度が得られます。お疲れさまでした 🎉