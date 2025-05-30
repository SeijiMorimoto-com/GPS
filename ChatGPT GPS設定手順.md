https://qiita.com/yamakenjp/items/e69eeabdefd9cc960610
## 4 | 手順を“実装書”としてまとめる（Pi 3B 用）

> **ゴール**: Pi 3B を単体 GPS-NTP サーバにする（RTC なし構成）。  
> **想定 OS**: Raspberry Pi OS Bookworm Lite 32-bit（最新 Imager で焼く）。

### 4-1 ハード配線

| GPS モジュール | Pi 3B 40 pin           | 備考                 |     |
| --------- | ---------------------- | ------------------ | --- |
| VDD 5 V   | PIN 4 (5 V)            | 5 V 電源             |     |
| GND       | PIN 6 (GND)            |                    |     |
| TXD       | PIN 10 (GPIO 15 / RXD) |                    |     |
| RXD       | PIN 8 (GPIO 14 / TXD)  |                    |     |
| 1 PPS     | PIN 12 (GPIO 18)       | 1 PPS = Low 100 ms |     |

### 4-2 OS 初期設定

bash

コピーする編集する

`# SSH でログイン後 sudo raspi-config nonint do_change_timezone Asia/Tokyo sudo raspi-config nonint do_boot_behaviour B2  # CLI オンリー`

### 4-3 /boot/config.txt 追記

bash

コピーする編集する

`sudo tee -a /boot/config.txt <<'EOF' ################################ # GPS 時刻同期 (RPI3B) enable_uart=1 init_uart_baud=9600  # === ここはどちらか一方 === dtoverlay=disable-bt               # ① BT 要らない派 #dtoverlay=miniuart-bt            # ② BT 残したい派 #core_freq=250                     # ↑②を選んだ場合に必須 # =============================  dtoverlay=pps-gpio,gpiopin=18,assert_falling_edge=true EOF`

### 4-4 シリアルコンソール無効化

bash

コピーする編集する

`sudo sed -i 's/console=serial0,[0-9]* //g' /boot/cmdline.txt sudo systemctl disable --now serial-getty@ttyS0.service`

### 4-5 モジュールロード

bash

コピーする編集する

`echo pps-gpio | sudo tee -a /etc/modules`

### 4-6 再起動してデバイス確認

bash

コピーする編集する

`sudo reboot # 戻ったら ls -l /dev/ttyAMA0   # 存在すれば OK ls -l /dev/pps0      # 同上`

### 4-7 gpsd + ツール導入

bash

コピーする編集する

`sudo apt update sudo apt -y install gpsd gpsd-tools pps-tools`

`/etc/default/gpsd` を以下のように：

bash

コピーする編集する

`sudo sed -i 's|DEVICES=""|DEVICES="/dev/ttyAMA0 /dev/pps0"|' /etc/default/gpsd sudo sed -i 's|GPSD_OPTIONS=""|GPSD_OPTIONS="-n"|' /etc/default/gpsd sudo systemctl restart gpsd.socket gpsd.service`

動作確認：

bash

コピーする編集する

`gpsmon -n      # 衛星が捕まると経度緯度が動く sudo ppstest /dev/pps0   # 1 秒ごとに assert が出れば OK`

### 4-8 chrony 導入

bash

コピーする編集する

`sudo apt -y install chrony`

`/etc/chrony/chrony.conf` 末尾に追記：

text

コピーする編集する

`refclock PPS /dev/pps0 lock NMEA refid GPS leapsecmode slew maxslewrate 1000 smoothtime 400 0.001 leaponly`

**外部 NTP も置き換えたい場合**（NICT 推奨）:

bash

コピーする編集する

`sudo sed -i 's|pool .*debian.*|pool ntp.nict.jp iburst minpoll 6 maxpoll 8|' /etc/chrony/chrony.conf`

サービス再起動：

bash

コピーする編集する

`sudo systemctl restart chrony`

状態確認（3-5 分後）：

bash

コピーする編集する

`chronyc sources -v # "GPS" 行に '*' が付いて offset ±数 µs → 成功`

### 4-9 動作チェックまとめ

bash

コピーする編集する

`timedatectl | grep -E 'System clock|NTP service' # → yes / active`

これで **Pi 3B が µs レベルで GPS 時刻にロックされた NTP サーバ** になりました。ローカル LAN 内の PC や別の Raspberry Pi から

bash

コピーする編集する

`sudo timedatectl set-ntp false   # 既存 NTP を切る場合 sudo ntpdate -q <Pi3B_IP>`

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