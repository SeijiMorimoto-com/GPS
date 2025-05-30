# GPS時刻同期手順

---
## Prompt:
RaspberryPi に GPSレシーバー と RTC をつけて GPS 1PPS 対応 NTPサーバ にする.md

ファイル

- この記事を元に、GPSから時刻を受信するraspberry piを作成したいです。
- 記事ではraspberry pi4 を使用していますが、私はraspberry pi 3bを使用します。
- 記事内の略語の解説をまず行い、raspberry piをGPS時刻に合わせる手順と、それ以降に話を分けて初学者が理解しやすいように詳細に解説してください。
- この記事の話を元に、最後に3bでのGPS時刻同期の実装方法についての手順を詳細に解説してください。

Thought for 18 seconds

ドキュメントを読み込んでいます

To adapt the article for Raspberry Pi 3B, I’ll break it into hardware setup, OS configuration, and NTP integration. I’ll explain abbreviations like GPS, PPS, and NTP for clarity. I need to focus on Pi 3B-specific differences—like using the miniUART or disabling Bluetooth for communication—while making sure to provide up-to-date steps using chrony, as it's still the standard. The article from 2022 is mostly applicable, no need for location-specific info or extra tools. I'll focus on the clear steps and final implementation for syncing GPS time.

完了

---
## Response:
### 1 | まずは略語（ざっくり辞書）

| 略語 | 意味／役割 | ひとことメモ |
| --- | --- | --- |
| **GPS / GNSS** | Global Positioning System / Global Navigation Satellite System | 全衛星測位システムの総称。今回は「時刻を貰う」ために使う。 |
| **QZSS** | Quasi-Zenith Satellite System（みちびき） | 日本上空に長く居る測位衛星。受信できると補足が速い。 |
| **1 PPS** | 1 Pulse-Per-Second | GPSモジュールが毎秒出す基準パルス（±10 ns～）。NTPにこれを食わせると秒単位の位相ずれがほぼゼロになる。 |
| **UART** | Universal Async Receiver Transmitter | TX/RX 2 本でシリアル通信。GPSモジュールの NMEA 文を Pi へ流す。 |
| **PL011 / miniUART** | Pi 本体にある 2 種類の UART | 3B は BT 用に PL011 が取られているので要設定。 |
| **NMEA** | GPS から流れてくる ASCII データフォーマット | 緯度経度や時刻などを “$GPGGA,…” の形で出す。 |
| **PPS-GPIO** | Linux カーネルの PPS フロントエンド | GPIO ピンを PPS として扱うドライバ。 |
| **gpsd** | GPS データを扱うデーモン | UART + PPS をまとめて chrony へ渡す。 |
| **chrony** | 高性能 NTP サーバ／クライアント実装 | gpsd の時刻・PPS を元にシステムクロックを校正。 |
| **RTC / DS3231** | Real-Time Clock | 衛星が捕まる前のバックアップ時計（今回はおまけ）。 |

出典 : 記事全体で使われている用語から要約RaspberryPi に GPSレシーバー…

* * *

2 | 流れを大きく分けると
--------------

1.  **ハードをつなぐ**  
    GPS モジュール（電源・TX/RX・1 PPS）を 40 pin ヘッダに配線。Pi 3B なら基本は同じピン番号（PIN 4 = 5 V, 6 = GND, 8 = TXD, 10 = RXD, 12 = GPIO 18 → PPS）RaspberryPi に GPSレシーバー…
    
2.  **OS 側で UART & PPS を有効化**
    
    *   `/boot/config.txt` に `enable_uart=1` と `init_uart_baud=9600` を追加。
        
    *   **Pi 3B 固有**:
        
        *   _Bluetooth を殺す_ → `dtoverlay=disable-bt` **または**
            
        *   _BT を miniUART へ追い出す_ → `dtoverlay=miniuart-bt` ＋ `core_freq=250` RaspberryPi に GPSレシーバー…
            
    *   シリアルコンソールを外す（`console=serial0,…` を消す）。
        
    *   PPS 用 overlay → `dtoverlay=pps-gpio,gpiopin=18,assert_falling_edge=true`。
        
3.  **gpsd を入れて動作チェック**
    
    *   `sudo apt install gpsd gpsd-tools pps-tools`
        
    *   `gpsmon -n /dev/ttyAMA0` で NMEA が流れるか確認。
        
    *   `ppstest /dev/pps0` で 1 PPS が刻むか確認。
        
4.  **gpsd をサービス化**  
    `/etc/default/gpsd`
    
    ```text
    GPSD_OPTIONS="-n"
    DEVICES="/dev/ttyAMA0 /dev/pps0"
    ```
    
    → `systemctl restart gpsd`RaspberryPi に GPSレシーバー…
    
5.  **chrony でシステムクロックを合わせる**  
    `/etc/chrony/chrony.conf` に
    
    ```text
    pool ntp.nict.jp iburst minpoll 6 maxpoll 8   # 外部参照（好みで）
    refclock PPS /dev/pps0 lock NMEA refid GPS    # PPS を基準に
    leapsecmode slew
    maxslewrate 1000
    smoothtime 400 0.001 leaponly
    ```
    
    → `systemctl restart chrony` → `chronyc sources -v` で `GPS` 行に `*` が付けば勝ち。RaspberryPi に GPSレシーバー…
    
6.  **（任意）RTC 追加**  
    DS3231 を I²C へ配線し `dtoverlay=i2c-rtc,ds3231`。Chrony が 11 分ごとに hwclock を書くのでバックアップ完了。RaspberryPi に GPSレシーバー…
    

* * *

3 | Pi 3B ならではの要点
-----------------

| 項目 | Pi 4B（記事） | **Pi 3B** |
| --- | --- | --- |
| Bluetooth | USB-接続 | SoC 内蔵で PL011 UART を占有 |
| デフォルト UART ピン | `serial0` = PL011 | `serial0` = **miniUART** |
| 推奨設定 | そのまま `enable_uart=1` | **どちらか選ぶ**  
① `dtoverlay=disable-bt` → BT 無効・GPIO は PL011 へ  
② `dtoverlay=miniuart-bt` → BT を miniUART へ移し、`core_freq=250` で安定化 |
| config ファイル場所 | `/boot/firmware/config.txt`（Bookworm 64-bit） | Bookworm 32-bit Lite でも **同じパス**。古い環境なら `/boot/config.txt` |

* * *

4 | 手順を“実装書”としてまとめる（Pi 3B 用）
----------------------------

> **ゴール**: Pi 3B を単体 GPS-NTP サーバにする（RTC なし構成）。  
> **想定 OS**: Raspberry Pi OS Bookworm Lite 32-bit（最新 Imager で焼く）。

### 4-1 ハード配線

| GPS モジュール | Pi 3B 40 pin | 備考 |
| --- | --- | --- |
| VDD 5 V | PIN 4 (5 V) | 5 V 電源 |
| GND | PIN 6 (GND) |  |
| TXD | PIN 10 (GPIO 15 / RXD) |  |
| RXD | PIN 8 (GPIO 14 / TXD) |  |
| 1 PPS | PIN 12 (GPIO 18) | 1 PPS = Low 100 ms |

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

* * *

### 5 | トラブルシューティング早見表

| 症状 | チェックポイント |
| --- | --- |
| `/dev/ttyAMA0` が無い | `enable_uart=1` が効いているか／BT overlay 設定ミス |
| NMEA は来るが PPS デバイスが出ない | `dtoverlay=pps-gpio,…`／配線（GPIO 18） |
| `chronyc sources` で `?` のまま | gpsd が起動していない／DEVICES 設定漏れ |
| Pi が数分で落ちる | 5 V 電源不足（GPS モジュールは 150 mA くらい食う） |

* * *

これで **Raspberry Pi 3B 版 GPS 1 PPS NTP サーバ** の完成！　屋根裏など空が見える場所に置けば、研究室全体の時計基準として十分な精度が得られます。お疲れさまでした 🎉

---
## Prompt:
これを複数台のraspberry pi3bで行えば、すべてのraspberry pi3bはGPS時刻に統一され、デバイス間の時刻誤差は小さくなりますか？

---
## Response:
Thought for 18 seconds

To address this, when multiple Raspberry Pi 3Bs use GPS 1PPS, each will align independently to GPS time (UTC). While synchronization will be tight, internal oscillator errors and PPS-to-kernel jitter can introduce minor inaccuracies, often in the microsecond range. Network discrepancies could lead to offsets if devices aren't on the same network, still keeping time within microseconds or better. Using chrony or PPS discipline will help, though attention to antenna setup and environmental factors is important for minimizing error.

I’m thinking about whether browsing is necessary for this task. I could rely on my internal knowledge, but it might be helpful to look up GPS accuracy information specifically for chrony and Raspberry Pi. However, I don't think it's strictly needed. I could just reference general accuracy standards and the chrony docs. Still, if I'm unsure, it might be worth briefly browsing for "chrony GPS PPS accuracy Raspberry Pi" to confirm the details.

ウェブ検索済み

chrony GPS PPS accuracy raspberry pi microseconds

[

![Favicon](https://www.google.com/s2/favicons?domain=https://austinsnerdythings.com&sz=32)

Austin's Nerdy Things

](https://austinsnerdythings.com/2021/04/19/microsecond-accurate-ntp-with-a-raspberry-pi-and-pps-gps/)

[

![Favicon](https://www.google.com/s2/favicons?domain=https://blog.networkprofile.org&sz=32)

NetworkProfile.org

](https://blog.networkprofile.org/gps-backed-local-ntp-server/)

[

![Favicon](https://www.google.com/s2/favicons?domain=https://www.reddit.com&sz=32)

Reddit

](https://www.reddit.com/r/RASPBERRY_PI_PROJECTS/comments/n3wiby/built_a_gps_pps_disciplined_microsecond_accurate/)

![Favicon](https://www.google.com/s2/favicons?domain=https://www.youtube.com&sz=32)

![Favicon](https://www.google.com/s2/favicons?domain=https://forums.raspberrypi.com&sz=32)

![Favicon](https://www.google.com/s2/favicons?domain=https://conorrobinson.ie&sz=32)

あと 6 個

Each Raspberry Pi will have its own GPS module, providing a local signal to chrony, which aligns the system clock to UTC with microsecond accuracy—often even sub-microsecond if jitter is minimal. When using PPS, typical offsets between devices stay under ±2 microseconds. However, without PPS (using just NMEA), offsets increase to tens or hundreds of milliseconds. Small errors, like GPIO latency and internal drift, can cause slight deviations, around 10-20 microseconds. This setup achieves accurate synchronization across devices.

完了

はい。  
各 Raspberry Pi 3B に **GPS-1 PPS + chrony** を入れて “ストラタム 1” 化すると、全台が同じ UTC 基準に同期します。実際のズレ量は以下のようなイメージです。

| 構成 | 1 台あたりの UTC との典型オフセット\* | Pi 間の相対誤差 |
| --- | --- | --- |
| NMEA だけ（PPS 無し） | ±50–300 ms | 同じ程度 |
| GPS + PPS（今回の構成） | ±1–5 µs | 同じオーダ（数 µs） |
| PPS をカーネルモードで使い、更に温度管理 | <1 µs | 1 µs 未満も可 |

\* 長時間(>30 min)ロック後／衛星可視が十分という前提。実測例では chrony が ±2 µs 台に収束した報告が多数あります。[Austin's Nerdy Things](https://austinsnerdythings.com/2021/04/19/microsecond-accurate-ntp-with-a-raspberry-pi-and-pps-gps/?utm_source=chatgpt.com)[Austin's Nerdy Things](https://austinsnerdythings.com/2025/02/14/revisiting-microsecond-accurate-ntp-for-raspberry-pi-with-gps-pps-in-2025/?utm_source=chatgpt.com)

* * *

### なぜ数 µs まで揃うのか？

1.  **PPS パルス精度**  
    多くの GPS 受信機は ±10 ns～100 ns 以内で 1 PPS を出します。
    
2.  **Linux PPS フレームワーク**  
    GPIO 割り込み時刻をカーネルがナノ秒解像度で取り込み、 chrony/NTPD に渡します。
    
3.  **chrony のディシプリン**  
    システムクロック（CPU の PLL）を PPS に “位相ロック” するので、発振器のドリフトが秒ごとに修正され続ける。
    
4.  **各 Pi が“独立に”UTC を参照**  
    相互にパケット往復させなくても、それぞれが同じ原器（GPS 衛星）に結びつくため、物理的にほぼ同じ時刻に収束します。
    

* * *

### 残るわずかなズレ要因

| 要因 | 目安 | 対策 |
| --- | --- | --- |
| GPIO‐IRQ 遅延の個体差 | 1–5 µs | 同じ SoC ならほぼ一定。重要度小 |
| 温度依存での局所水晶遅延 | 数 µs | ケース内温度を安定化／ヒートシンク |
| アンテナケーブル長差 | 5 ns/ m | 無視できる or 長さを揃える |
| chrony のスムージング設定 | 0.1–1 µs | `smoothtime` をそろえる |
| 受信衛星切替で一時的に jitter ↑ | 数 µs | 空の見通しを良くする／みちびき(QZSS)対応モジュール |

これらを抑えれば、研究用途で「デバイス間のタイムスタンプを ±10 µs 以内に整合させたい」という要求は問題なく満たせます。オーディオのサンプル同期（44.1 kHz で ±0.5 サンプル ≒ ±11 µs）や分散センサの時刻合わせでも十分な精度です。

* * *

### もっと精度を詰めたい場合

*   **カーネル PPS 線形化モード**（`refclock PHC /dev/pps0` + `lock`) を使い、ユーザー空間 jitter をさらに減らす
    
*   **共通外部 10 MHz 基準** で GNSS-DO（GPSDO）を作り、各 Pi の GPIO PPS をそこから分配する
    
*   **PTP（Precision Time Protocol）ハードウェアタイムスタンプ** 対応 NIC を追加し、LAN 上でサブ µs を狙う
    

ただし追加ハード・設定が必要になるため、多くのケースでは今回の“Pi + GPS PPS”だけで十分です。

* * *

TL;DR
-----

*   **複数台で同じ構成を組めば、自動的に µs オーダで揃う。**
    
*   実測では ±1–5 µs 程度が一般的。LAN 内アプリやセンサ統合には十分小さい誤差。
    
*   温度管理やカーネル PPS モードでさらに詰められるが、そこまでせずともオーディオ用途や実験ログの同期には十分実用レベル。

---
## Prompt:
"なぜ数 µs まで揃うのか？"について詳しく説明してください

---
## Response:
Thought for 1m 31s

The user is looking for a detailed explanation of how GPS PPS and chrony can synchronize devices to within microseconds. To elaborate, I’ll cover the entire synchronization process from the GPS satellite to the Pi system clock. I'll explain the role of signals, latencies, algorithms, and control loops. Key factors include GPS PPS accuracy, NMEA vs. PPS, jitter, kernel timestamping, the chrony algorithm, and temperature effects. Further, I'll discuss concepts like phase-locked loops, servo design, and offset smoothing.

The explanation of GPS PPS accuracy is conceptual, but it may be helpful to provide references, especially from Chrony documentation or other sources. I’ll consider browsing briefly for citations related to GPS PPS accuracy and using Chrony on a Raspberry Pi. The core explanation remains unchanged, but referencing current docs could lend credibility. I'll structure the explanation with several key sections, focusing on satellite timing accuracy, kernel time-stamping, Chrony’s compensation process, and local oscillators. Browsing could add official documentation as citation pointers.

After Chrony warms up, it can maintain an offset of less than 5 µs across devices. This involves statistical filtering using Chrony's fudge factors like pi, minpoll, and maxpoll for error bounding. When synchronizing multiple Raspberry Pis, each has independent correction loops, resulting in a typical difference considering their systemic residual offsets. The final difference is dominated by factors like jitter, oscillator quality, and temperature, which Chrony compensates for using techniques like temperature compensation and slew.

ウェブ検索済み

chrony gps pps accuracy microseconds raspberry pi

GPS 1pps sawtooth correction nanoseconds

[

![Favicon](https://www.google.com/s2/favicons?domain=https://chrony.tuxfamily.org&sz=32)

chrony.tuxfamily.org

](https://chrony.tuxfamily.org/doc/2.4/chrony.conf.html)

Chrony’s fudge factors (like pi, minpoll, and maxpoll) play a key role in error bounding, resulting in offsets under 5 µs after warm-up. When synchronizing multiple Raspberry Pis, each has independent correction loops, but their relative error is governed by their system’s residual offsets. The final difference depends on factors like jitter, oscillator quality, and temperature, which Chrony handles using techniques like temperature compensation and slew. We can cite sources on microsecond-level and nanosecond accuracy to support these claims.

I keep thinking about convergence times. It's essential to look at the performance in real-world scenarios, like showing an example from a blog to illustrate how the process works. I'll need to focus on discussing how convergence times can vary in practice and maybe offer a concrete example that demonstrates how quickly the system stabilizes. This will paint a clearer picture for the user, and it will make the technical detail easier to understand.

The Raspberry Pi handles PPS signals with low jitter, corrected via sawtooth error adjustments. Chrony uses the Linux PPS API, relying on the system’s monotonic clock for nanosecond precision and minimal latency, often with ten-nanosecond jitter. It locks to PPS as a reference clock and adjusts the system clock with a Proportional-Integral (PI) controller. Filtering out errors and residual jitter optimizes accuracy. Additional details cover oscillator drift, microsecond-level offsets, and chrony’s fine-grained calibration. I’ll document the steps with relevant formulas to show detailed timing adjustments and factors affecting synchronization.

To better understand the convergence process, I'll look at observed performance from blog sources and chrony outputs. I'll cite details from lines 31-34 that show a residual offset of -175 ns and jitter of 126 ns, supporting the microsecond accuracy. I’ll also refer to PPS electronics precision and chrony’s use of a servo algorithm for compensation. My goal is to craft a comprehensive explanation along with supporting citations, focusing on the path to achieving precise microsecond-level synchronization.

I’m thinking about incorporating uncertainties at each step. First, GPS PPS accuracy is influenced by the internal oscillator and sawtooth error, which can reach nanoseconds. Then, there's the latency from both the Linux system and the hardware itself. The Chrony algorithm adjusts the system clock with 1 Hz updates and an integration time constant. After the system warms up, the residual offset remains, which is important to account for in accuracy measurements.

ウェブ検索済み

chrony pps lock nmea phase error microsecond

ウェブ検索済み

chrony refclock PPS accuracy

We’ll account for uncertainties at each step: First, the GPS PPS accuracy is influenced by the internal oscillator and sawtooth error. Then, there’s the latency from both Linux and hardware. Chrony’s algorithm controls the system clock with 1 Hz updates and an integration time constant, and once the system warms up, the residual offset remains. We’ll gather additional info from Chrony docs, but using performance info from blogs helps clarify things. The response should be technical but casual as requested. I’ll also remember to cite as I write.

I’m focusing on referencing reliable sources for precision. From one source, we see an offset of -175ns, and from another, the document mentions nanosecond-level accuracy for 1PPS. There’s also information on chrony’s refclock precision. I’ll keep these citations in mind: they provide solid evidence, and I’ll include these references at appropriate points in my answer. The goal is to ensure they back up the explanation clearly and accurately while following the guidelines for proper citation formatting.

完了

“数 µsで揃う”仕組みを深掘りしよう
-------------------

Raspberry Pi + GPS 1 PPS + chrony が数マイクロ秒（10-6 秒）に収束できるのは、**①元ネタの精度がナノ秒級であること、②Linux がそのパルスをナノ秒解像度で時刻印字できること、③chrony が 1 Hz のフィードバック制御でホスト水晶を従えること**──この三段ロケットが効いているからです。以下、パスの各区間でどの程度の誤差が乗るかを順に見てみます。

* * *

### 1\. 衛星 → GPS 受信機

*   **原器は原子時計**  
    GPS 衛星は 10-14 級の周波数安定度を持つ原子時計で UTC に対して ±<40 ns で校正されています。
    
*   **受信機が“丸め誤差”を補償**  
    受信機内部の水晶（例えば 26 MHz）では 38.4 ns ごとに位相がズレます。大半のモジュールは「**Saw-tooth compensation**」と呼ばれる仕組みで 1 PPS のエッジを ±10–30 ns に詰め、補正量を NMEA 拡張文で吐きます[SimonsDialogs](https://www.simonsdialogs.com/2016/10/digital-delay-line-sawtooth-corrections-of-an-ultra-precise-gps-reference-1-pps-signal-and-thermal-effects/?utm_source=chatgpt.com)[Conwin](https://www.conwin.com/pdfs/AN108-saw-tooth-error-in-gps.pdf?utm_source=chatgpt.com)。
    

👉 **ここで既に 10 ns オーダ**──マイクロ秒の 100 分の 1。

* * *

### 2\. 受信機 → Pi GPIO

*   **電気配線誤差**  
    同軸でもジャンパでも伝搬速度は光速の 60–70 % 程度。1 m で **~5 ns**。通常は無視でき、合わせたければケーブル長を揃えるだけ。
    
*   **エッジ形状**  
    PPS は 3.3 V TTL で数 µs 幅。立ち上がり遅延があっても <10 ns 程度。
    

* * *

### 3\. Pi GPIO 割り込み → Linux PPS タイムスタンプ

*   **カーネルは“IRQ 到着時刻”を即刻保存**  
    `pps-gpio` ドライバはエッジ受付直後に `ktime_get_real_ts64()` を呼び、ナノ秒単位の `timespec` を確定。
    
*   **ジッタ要因**
    
    *   GPIO→CPU の同期 + 割り込みライン：~100 ns
        
    *   IRQ→時刻読出しの命令経路：~100 ns
        
    *   CPU 周波数変動（Pi3 の DVFS）：既に時刻読み出しが TSC 換算なので影響ほぼゼロ
        

合算しても **サブ µs のばらつき**に収まります。

* * *

### 4\. chrony のサーボが“μs 誤差”を潰すロジック

| 機能 | 仕組み | 誤差低減イメージ |
| --- | --- | --- |
| **粗い秒合わせ** | NMEA 文を SHM で受信。秒境界を確定し「このパルスは何時のものか」を chrony に教える。 | ±数 10 ms → 0 µs レベルへ |
| **微調整** | 同期済み PPS を `refclock PPS /dev/pps0 lock NMEA` で取り込み、パルスとシステムクロックの**位相差 φ**を 1 Hz で測定。 | φ ≈ ±10 µs からスタート |
| **周波数制御** | chrony サーボ (PI 制御) が `adjtimex()` で内蔵 PLL の周波数 f を毎秒更新。 | f を ±500 ppm → ±<0.1 ppm に収束 |
| **ノイズ除去** | 64 サンプル中央値 + 外れ値除去。PPS refclock はデフォルトで **1 ns precision** 扱い[chrony.tuxfamily.org](https://chrony.tuxfamily.org/doc/2.4/chrony.conf.html) | ジッタを ns〜100 ns 台へ |

実測例では再起動後 2–3 分で **Offset ≈ -175 ns, Jitter ≈ 126 ns** に落ち着いていることが確認できます[Austin's Nerdy Things](https://austinsnerdythings.com/2021/04/19/microsecond-accurate-ntp-with-a-raspberry-pi-and-pps-gps/)。

* * *

### 5\. 複数台が“同じ原器”を見るのでズレは √N

*   各 Pi の残留オフセット σ が ±2 µs とすると、独立 2 台間の相対誤差は  
     $\sigma_{\text{rel}} = \sqrt{2}\,\sigma \approx 2.8\;µ\text{s}$ .
    
*   実際、家庭用モジュール＋Pi3B でも **±1–5 µs** に収束した報告が多数あります[Austin's Nerdy Things](https://austinsnerdythings.com/2021/04/19/microsecond-accurate-ntp-with-a-raspberry-pi-and-pps-gps/?utm_source=chatgpt.com)。
    

これで **オーディオ 48 kHz の 0.25 サンプル (≈5 µs)**、TDOA 計測なら 340 m/s×5 µs ≈ 1.7 mm と、研究レベルでも十分な同期精度が得られます。

* * *

### 6\. まだ詰めたければ

1.  **カーネル PHC/EXTPPS**  
    GPIO ではなく SoC タイマーや NIC のハードウェアタイムスタンプで PPS を捕まえるとさらに ns 台へ。
    
2.  **TCXO/OCXO 化**  
    Pi の 19.2 MHz 水晶を TCXO に交換すると温度依存周波数誤差が 1 桁下がり、servo の振幅が小さくなる[Austin's Nerdy Things](https://austinsnerdythings.com/2021/04/19/microsecond-accurate-ntp-with-a-raspberry-pi-and-pps-gps/)。
    
3.  **PTP (IEEE 1588) ハードスタンプ**  
    LAN 上でも ns〜100 ns 台で揃えられる。[chrony.tuxfamily.org](https://chrony.tuxfamily.org/comparison.html?utm_source=chatgpt.com)
    

* * *

まとめ
---

*   **GPS 1 PPS は元々 ±10 ns 級**。
    
*   **Linux PPS フレームワーク**がそのタイミングをほぼロスなく記録。
    
*   **chrony** の PI サーボがホスト水晶を 1 Hz で制御し、残留誤差を数 μs→数 100 ns へ。
    
*   各 Pi が独立に同じ UTC にロックするため、相互誤差もそのまま **µs オーダに縮む**。
    

この三段構えのおかげで、特別な測定器を使わずとも **“数マイクロ秒で揃う分散システム”** をラズパイだけで組めるわけです。

---
## Prompt:
ジッターであったり、ロック、オフセットなどの一般生活ではあまり聞き馴染みのない用語について詳細に解説してください。

---
## Response:
Thought for 41 seconds

The user is asking for an explanation of terms like "jitter," "lock," and "offset," which are used in timing and synchronization, especially in contexts like clocks and timekeeping technologies. I’ll explain these terms in clear, intuitive ways using analogies and defining them from a technical viewpoint. It’s interesting to note that these concepts relate to sources like GPS and chrony. For completeness, I'll quickly review the definitions to ensure accuracy and provide helpful citations.

ウェブ検索済み

clock jitter definition chrony

NTP offset definition

time synchronization lock status explanation

[

![Favicon](https://www.google.com/s2/favicons?domain=https://chrony-project.org&sz=32)

chrony-project.org

](https://chrony-project.org/doc/4.0/chronyc.html)

The user has requested an explanation of terms like "jitter," "lock," and "offset," particularly in the context of clocks and time synchronization. These terms aren't widely used in everyday language, so I plan to use both analogies and technical definitions to help clarify the concepts. I’ll explain them with reference to tools like GPS and chrony. Since these are general terms, I can use internal knowledge, though it wouldn't hurt to quickly check for definitions to ensure clarity.

I'm preparing to craft the explanation for terms like "offset," "jitter," and "lock" that the user asked about.

*   "Offset" refers to the phase difference between a local clock and a reference, typically measured in microseconds. It's static but can shift over time.
    
*   "Jitter" is the variation of this offset over time, often calculated through statistical measures, like root mean square.
    
*   "Lock" means the synchronization system has successfully aligned with a time reference.
    

I'll reference reliable sources like chrony.conf and application notes for further clarity.

ウェブ検索済み

chronyc sources offset meaning jitter

I need to define the terms:

*   "Offset" is the phase difference between a local clock and a reference, measured in microseconds. It can be static but may drift.
    
*   "Jitter" refers to the variation in offset over time, often measured statistically, like using root mean square.
    
*   "Lock" indicates that a system has successfully synchronized to a reference, with some tolerance allowed.
    

These terms are key in understanding synchronization. I'll use these definitions, adding relevant context for clarity when needed.

I’ll craft an explanation with everyday analogies for a clearer understanding:

*   "Offset" is the difference between your clock and a reference, like comparing your watch to a precise atomic clock.
    
*   "Drift" represents gradual changes in frequency, a bit like your clock slowly gaining or losing time.
    
*   "Jitter" refers to random fluctuations over time in that offset, like your clock jumping erratically.
    
*   "Lock" is when your clock syncs and stays aligned with the reference.
    

I’ll use these analogies and build on chrony’s context with examples, adding citations as needed.

Here’s a straightforward breakdown:

*   **Offset**: The difference between your clock and a reference at a certain point, like your watch being fast or slow compared to an atomic clock.
    
*   **Drift**: The gradual error change over time between the clocks.
    
*   **Jitter**: The random variations in that offset, much like your watch jumping randomly around the ideal time.
    
*   **Lock**: When your clock is successfully synchronized with a reference system, like constantly adjusting your watch for precise alignment.
    

I’ll also explain how chrony tracks these with commands like `chronyc` for practical examples.

I’ll explain these key terms in simple terms:

*   **Offset**: The difference between your clock’s time and a reference clock at a given moment.
    
*   **Drift**: The gradual change in a clock’s frequency over time.
    
*   **Jitter**: The fluctuation in offset over time, measured as the root mean square of consecutive offsets.
    
*   **Lock**: When a system successfully synchronizes with a reference clock (like aligning your watch).
    

I’ll explain how chrony tracks these timing terms, showing how they reflect our clocks and sources!

Here's a breakdown of the relevant terms:

*   **Offset (phase error)**: This is the difference between your clock and a reference at a specific moment.
    
*   **Drift**: This refers to the gradual frequency shift over time.
    
*   **Jitter**: Jitter is the variation in the offset, essentially how unstable the time difference is.
    
*   **Lock**: When your system achieves synchronization with a reference clock.
    

I’ll also incorporate analogies—like comparing your watch to an atomic clock—and explain how chrony monitors these with commands like `chronyc` for tracking.

Here are the key terms explained:

*   **Offset**: The difference between your clock and a reference at a given moment.
    
*   **Drift**: A slow frequency discrepancy in time.
    
*   **Jitter**: Variability or "noise" in the offset, like random fluctuation in reading time.
    
*   **Lock**: This happens when your system synchronizes with a time reference.
    

To make these more relatable, think of offset as your watch being too

完了

### “ジッター・ロック・オフセット”って何者？

――GPS × chrony の世界で出てくる時間用語を、時計のたとえと数式を交えて分解

| キーワード | ざっくり一言 | もう少しディープな定義 | どこで見える？ |
| --- | --- | --- | --- |
| **オフセット (offset)** | **今この瞬間、あなたの時計がどれだけズレているか** | 参照時刻（UTC など）との **位相差 Δφ**。chrony では「Last offset」「System time … slow / fast」として µs 単位で表示 | `chronyc tracking`, `chronyc sources` の 3～4 列目[Chrony](https://chrony-project.org/doc/4.0/chronyc.html) |
| **ジッター (jitter)** | **そのズレが毎回どれだけブレるか** | 連続するオフセット測定値の変動幅を RMS（√平均二乗）で表したもの。揺らぎの標準偏差なので小さいほど安定。NTP/chrony では “RMS offset” や “jitter” 列で µs～ms 表示[LPI Academy](https://learning.lpi.org/en/learning-materials/102-500/108/108.1/108.1_02/?utm_source=chatgpt.com) |  |
| **ロック (lock/locked)** | **「もう基準に追いついたよ！」という合図** | サーボ制御（PI ループ）が収束し、許容範囲内のオフセットとジッターで**追従状態**になったフラグ。chrony だと該当ソース行に `*` が付く／`tracking` に “Leap status : Normal” と出る。PPS を `lock NMEA` で組ませたときの “lock” も同じ意味[Chrony](https://chrony-project.org/doc/4.1/chrony.conf.html?utm_source=chatgpt.com) |  |
| **ドリフト (drift/frequency error)** | **時計が一日でどれだけ進み／遅れるか** | 水晶発振子の周波数誤差 f (ppm)。chrony は「Frequency」「Residual freq」で示す。オフセットの“傾き”と捉えると分かりやすい[Chrony](https://chrony-project.org/doc/4.0/chronyc.html) |  |

* * *

1 | オフセット ― 位相差のスナップショット
------------------------

*   家で合わせた腕時計が 1 秒進んでいたら **offset = +1 s**。
    
*   chrony の場合、GPS 1 PPS が基準なので ±数 µs まで小さくなれば “ほぼ合っている” 判定。
    
*   **単位**は秒（s）だが cli は慣用的に ms → µs → ns へ自動スケール。
    

> `Last offset : -0.000006747 seconds` → **\-6.7 µs 遅れ**[Chrony](https://chrony-project.org/doc/4.0/chronyc.html)

* * *

2 | ジッター ― ゆらゆら度合いを統計的に測る
-------------------------

1.  **オフセットを 1 秒ごとに取る**と
    
    $$
    \Delta\phi_1,\,\Delta\phi_2,\,\dots,\Delta\phi_N
    $$
    
2.  **平均差**を引いたうえで 2 乗平均 → √ を取ると
    
    $$
    \text{jitter}= \sqrt{\frac1N\sum_{i=1}^N (\Delta\phi_i-\bar\Delta\phi)^2}
    $$
    
    ＝“ぶれ幅の標準偏差”。
    
3.  NTP 系では **RMS offset** という名前で同じ指標を出している。
    

| ジッター値 | 体感イメージ |
| --- | --- |
| **\>1 ms** | ネット遅延が大きい公衆 Wi-Fi で NTP した感じ |
| **10–100 µs** | LAN 内サーバに同期 |
| **<5 µs** | GPS + PPS＋chrony（今回の構成） |

資料では「Timing variations of signal edges from their ideal values」と定義されることが多いです[sitime.com](https://www.sitime.com/support/resource-library/application-notes/an10007-clock-jitter-definitions-and-measurement-methods?srsltid=AfmBOormn3oqN5kdrW-O1dTot0K_bwCW_nm3pEN4Rk_Sp2JSVvge21Q4&utm_source=chatgpt.com)。

* * *

3 | ロック ― サーボが“捕まった”状態
----------------------

*   \*\*制御工学で言う“位相同期（PLL が lock した）”\*\*と同義。
    
*   chrony の 2 段サーボ
    
    1.  **粗同期** NMEA 文（時刻ラベル）で秒境界を合わせる
        
    2.  **位相比較** PPS で毎秒差分 φ を測り **PI ループ**で周波数修正
        
*   収束条件を満たすとソース行に `*` が付く → これが “locked” 完了サイン。
    

* * *

4 | ドリフト ― 周波数誤差で Offset が“傾く”理由
--------------------------------

*   温度や部品ばらつきで **1 ppm ずれると 1 秒 / 11 d**。
    
*   修正しないと offset は一直線に増減。
    
*   chrony は `adjtimex()` で発振周波数を少しずつ補正し、Residual freq を 0 ppm 付近へ寄せる。
    
*   **ジッターが小さいほどドリフト推定精度が上がり、最終 offset も小さくなる**。
    

* * *

5 | まとめイメージ
-----------

```
     参考UTC
        │      ┌──────┐ jitter
        │      │  ±x  │  ← 毎回の測定揺れ
        ▼      └─┬────┘
  ┌──────────┼────────────── 時間
  │           │●
  │      ●    │          ●   ●
offset ↑   ●         ●
  │ ●
  └──────────┴───────→
        （ドリフト傾き）
```

1.  **offset** が縦軸、時間が横軸。
    
2.  黒点が毎秒の offset 測定。ばらつき幅が **jitter**。
    
3.  全体の傾きは **ドリフト**。
    
4.  chrony が補正し切って水平になり、点が狭い帯に収まった瞬間が **lock**。
    

* * *

> **要は…**
> 
> *   _Offset_ = 今のズレ量
>     
> *   _Jitter_ = そのズレがフラフラする量
>     
> *   _Drift_ = ズレが勝手に増減する速度
>     
> *   _Lock_ = それらが許容範囲まで抑え込まれ、リファレンスを“つかまえた”状態
>     

この 4 つを押さえておけば、`chronyc tracking/sources` を覗いたとき **数字が何を意味するのか一発でピンと来る**ようになります。

---
## Prompt:
複数台で時刻同期を行う場合は、相対的に時刻が合ってれば良いので、参照時刻が同じオフセット、ジッターの値が一致しているかを見れば良いという解釈で正しいですか？

---
## Response:
Thought for 9 seconds

For multiple Pi devices, relative synchronization is about having their clocks closely aligned, not necessarily with the exact same offset from the reference clock. The key is that the offsets and jitter across devices should be similar. While jitter doesn’t need to match exactly, we care about its magnitude. Important metrics include offset differences, RMS jitter, and frequency stability between devices. Tools like chronyc can help monitor these metrics for better synchronization across devices.

完了

だいたい合っていますが、**見るべき値の優先度**を整理すると次のようになります。

| 目的 | チェックポイント | 理由 |
| --- | --- | --- |
| **① 相互ズレを小さくしたい** | **各 Pi の “Last offset” を比較し、その差を取る**  
例）\` | offsetA - offsetB |
| **② その差が時々大きくならないか** | **各 Pi の “jitter”**  
値が大きければ offset も振れる → 差も増える | jitter は「1 秒ごとの測定がどれだけ揺れるか」の統計値。  
jitterA と jitterB が “_同じ_” である必要はないが、  
**どちらも十分小さい（<5 µs など）か**は確認したい。 |
| **③ 長期的にズレが増えないか** | **“Residual freq / Skew”** を見る | これは水晶の周波数誤差。  
両 Pi の値が ±0.1 ppm 程度に収束していれば、  
GPS ロックが外れても 10 分で 60 µs しか広がらない。 |

* * *

### ✅ 正しい解釈ポイント

1.  **参照（GPS 1 PPS）が同じなら相対誤差＝offset の差**
    
    *   各 Pi の `chronyc tracking` → `Last offset` を並べて引き算するだけで相互ズレが測れます。
        
    *   2 台とも `*`（ロック済み）かつ ±数 µs に収まっていれば目的達成。
        
2.  **jitter は “揃える” 指標ではなく “小さく保つ” 指標**
    
    *   片方が 1 µs、もう片方が 3 µsでも、offset の差が安定していれば問題ありません。
        
    *   ただし **jitter が大きいと offset もふらつき → 差も揺れる**ので、両方とも小さいほうが安心。
        
3.  **offset が同じ符号・同じ絶対値である必要はない**
    
    *   例：Pi-A = +3 µs、Pi-B = +4 µs → 差は 1 µs。
        
    *   “UTC に対してゼロ” かどうかより、**互いの差が小さい**ことが目的なら十分です。
        
    *   ただし大きく外れている場合（例：+50 µs と -50 µs）は GPS ロック失敗を疑うべき。
        

* * *

### 📋 実践チェック手順（例）

```bash
# Pi-A 側
chronyc tracking | grep "Last offset"
#  -> Last offset     : -0.0000032 seconds   (-3.2 µs)

# Pi-B 側
chronyc tracking | grep "Last offset"
#  -> Last offset     : -0.0000047 seconds   (-4.7 µs)

# 差は 1.5 µs → OK
```

さらに 30 秒おきに数分観測して **最大差 <5 µs、平均差 <2 µs、jitter <5 µs** なら  
音響測定や TDOA 解析などでもまず問題ないレベルです。

* * *

### ✋ 注意点

*   **アンテナ視界・PPS 配線**が悪いと片方だけ jitter が跳ね上がり、差も瞬間的に数 10 µs へ飛ぶことがあります。
    
*   **chrony の `smoothtime`** を入れていると表示上の offset が“平滑化後”の値になります。相対誤差を厳密に測るときは `smoothtime` を一時的に無効にするか `chronyc sourcestats` の raw 値で比較してください。
    
*   より厳密に監視したい場合、片方を NTP サーバにしてもう一方から `ntpdate -q <IP>` で数百回サンプリングし、統計を取る方法もあります（LAN jitter を含めても数 µs 以内に収束するはず）。
    

* * *

**まとめ**

*   _Yes_ — 複数台の相対同期では「各 Pi の offset を比べて差が小さいか」を見るのが本質。
    
*   jitter やドリフトは **差の安定性を保つための健康診断**として小さくなっているか確認する。
    
*   同じ値に“ピッタリ揃う”必要はなく、μs オーダで差が収束していれば目的達成です。

---
## Prompt:
GPSについてのきちんとした理解のための詳細な資料を作成して3つの.mdファイルをzipで出力してください。
- GPSシステムの基礎について.mdで作成してください。
- 一般的に聞かない用語についての詳細な解説を記述するようにしてください。その際は、厳密な定義とイメージを示すようにして、.mdで作成してください。
- GPSシステム、用語の解説を踏まえ、複数のraspberry pi 3bでの時刻同期の手順、精度確認の手順など、デバイス開発のための詳細な内容を別の.mdで作成するようにしてください。

---
## Response:
Thought for 28 seconds

So, I’m following the instructions about providing a file download link in the response. First, I’ll prepare the code for generating markdown files and zipping them up. After the code runs in the commentary channel, I'll use the final channel to summarize and provide a link, like "." I need to ensure that the zip file creation and naming are done properly. And I’ll do all this in the correct sequence for clarity.

完了

3 本の Markdown 資料をまとめた ZIP を用意しました。どうぞご活用ください。

Download the ZIP

---
## Prompt:
各raspberry piごとにオフセットを求める際のUTC時刻のずれはないのでしょうか？

2/2

---
## Response:
思考時間: 20 秒

各Raspberry Piがオフセットを計算する際に、UTC時刻のずれによる誤差がないか考え中です。基本的には、全てのPiは同じUTCスケールを使用します。オフセット値はそのスケールに基づいており、リモートの差異は相対誤差に過ぎません。ただし、GPSのUTC補正のディレータやスムージングによる小さな差が影響する可能性はあります。これでも大きなずれは発生しません。

Raspberry Pi間でUTCの絶対オフセットに違いがあるかどうか、という質問ですね。各PiはGPSからUTCを取得し、GPS同期の基準に基づいてオフセットを計算します（オフセットはPPSのタイミングに基づきます）。主要な違いは、サーボやケーブルの遅延、内部クロックの違い、さらにはジャンパ線やGPIOのジッタなどです。また、リープセカンドによる差異もありますが、定期的にchronyが対応します。最終的には、Pi間のUTC時刻のズレはごくわずかですが、内部オフセットの微妙な差異は存在します。

完了

### 結論だけ先に

*   **理論的には同じ UTC を見ているので「ずれゼロ扱い」で OK**  
    └ GPS ⇄ UTC の換算は衛星が配信する _Δt<sub>UTC-GPS</sub>_（現在 18 s）を使うため、各 Pi で共通。
    
*   **実際には μs オーダの残差が出る**  
    └ 受信機内部の saw-tooth 誤差、配線遅延、GPIO 割り込みジッター、chrony の平滑化など **ローカル要因** が乗る。
    

つまり「UTC そのものが Pi ごとに違う」わけではなく、**UTC に対して Pi のシステムクロックがどれだけ外れているか**（＝`Last offset`）が微妙に異なると思ってください。

* * *

1 | GPS ⇄ UTC の扱いは全機共通
----------------------

| ステージ | 処理内容 | 影響 |
| --- | --- | --- |
| **衛星** | NAV メッセージで  
① 衛星クロック補正係数  
② `Δt_{UTC-GPS}`（うるう秒差）を送信 | 全受信機が同じ値を受信 |
| **受信機** | ・自前の水晶で 1 PPS を生成  
・NAV 内の係数で GPST→UTC へ換算 | ±(10–100) ns 精度で共通 UTC |
| **`pps-gpio`** | PPS 立ち上がりをカーネル時刻と瞬時に突き合わせ | 入力ジッター <0.1 µs |
| **chrony** | PPS を “UTC の正秒” と解釈し  
システムクロックを追従 | ここで初めて個体差が現れる |

### ⇒ **UTC–GPS 換算で台ごとの差はほぼ 0**

（そもそも 18 s は整数秒なので位相の話には関与しない）

* * *

2 | Pi ごとに誤差が出る 4 つのローカル要因
--------------------------

| 要因 | 典型値 | 説明 |
| --- | --- | --- |
| **受信機 saw-tooth** | ±20 ns RMS | 26 MHz 水晶の量子化誤差を ±1 クロックで揺らす |
| **ケーブル／基板遅延** | 5 ns / m | PPS → GPIO までの伝搬時間差 |
| **GPIO→IRQ キャプチャ遅延** | 50–200 ns | SoC 内配線＋割り込みレイテンシ |
| **chrony サーボ残差** | ±1–5 µs | 水晶ドリフト補正の収束誤差・`smoothtime` 平滑化 |

> _観測値_: 実験的には複数台の Pi3B で `Last offset` が −4 µs / −5 µs / −6 µs …のように **数 µs のばらつき**になるのがふつう。

* * *

3 | 実務的なチェック方法
--------------

1.  **全台がロック済みか確認**
    
    ```bash
    chronyc sources -v     # GPS 行に * が付くか
    ```
    
2.  **`Last offset` を並べて差分を取る**
    
    ```bash
    chronyc tracking | awk '/Last offset/ {print $4*1e6}'   # 単位 µs
    ```
    
3.  **差が 5 µs 以内なら合格**
    
    *   TDOA (48 kHz で 0.24 sample) や分散センサなら十分。
        
    *   もっと絞りたい場合は
        
        *   ケーブル長を揃える
            
        *   `dtoverlay=pps-gpio,assert_falling_edge=true` で立下り固定
            
        *   `smoothtime` をオフにし raw PPS で評価
            

* * *

4 | もし大差が出たら？
-------------

| 症状 | チェックポイント |
| --- | --- |
| 片方だけ offset ±数十 µs | GPS 信号品質（C/N0 < 35 dB-Hz？） |
| jitter >20 µs | アンテナ位置によるマルチパス／電源ノイズ |
| ロックに数十分かかる | NAV データ未取得 → コールドスタートを疑う |

* * *

### TL;DR

*   **UTC‐GPS の整数秒補正は衛星から一意に降ってくる**ので、Pi ごとの “UTC の基準” がズレる心配は実質ありません。
    
*   見るべきは **各 Pi が UTC からどれだけ外れているか（Last offset）** とそのブレ（jitter）。
    
*   現実には μs オーダの残差が出るが、ケーブル長や chrony 設定で ±1–2 µs まで詰められます。

---
## Prompt:
"“ジッター・ロック・オフセット”って何者？"この部分について、厳密な定義と共に説明してください。

---
## Response:
思考時間: 8 秒

ユーザーは「ジッター・ロック・オフセット」の厳密な定義を求めているので、時間同期とChronyの文脈で正確に説明します。オフセットは、システムクロックと参照クロックの瞬間的な差異で、Chronyでは「Last offset」として表されます。ジッター、ロック、オフセット、ドリフト、または位相誤差や周波数誤差の正確な定義も含めるべきですね。しかし、インターネットを検索する必要はなく、内蔵の知識で十分対応できます。

完了

“オフセット・ジッター・ロック”──時間同期で使う 3 つの基礎概念を **“厳密な定義”** で整理
---------------------------------------------------

| 用語 | 数式ベースの厳密定義 | 観測値の出どころ | 意味合い（なぜ重要か） |
| --- | --- | --- | --- |
| **Offset**  
（オフセット，位相誤差） | 系（ホスト）の現在時刻 \\(T\_\\text{sys}(t)\\) と参照時刻 \\(T\_\\text{ref}(t)\\) の差：  
\\\[\\Delta\\phi(t)=T\_\\text{sys}(t)-T\_\\text{ref}(t)\\\] | `chronyc tracking` の _Last offset_  
`chronyc sources` 第 3 列 | _瞬間的_ にどれだけズレているか。  
μs 単位で±0に近いほど「合っている」。 |
| **Jitter**  
（ジッター，ゆらぎ） | 連続 \\(N\\) サンプルの offset について  
\\\[\\sigma\_\\phi=\\sqrt{\\dfrac1N\\sum\_{i=1}^N\\bigl(\\Delta\\phi\_i-\\overline{\\Delta\\phi}\\bigr)^2}\\\] （＝RMS/標準偏差） | `chronyc sourcestats` の _Jitter_ 列  
`tracking` の _Root delay/jitter_ | offset 測定値が_どの程度ばらつくか_。  
サーボの安定度・信号品質の指標。 |
| **Lock**  
（ロック，位相同期待機） | フィードバック制御系が設定閾値を満たし，「状態推定変数（offset・freq）が収束域内」に入った状態。  
形式的には  
\\\[ | \\Delta\\phi(t) | <\\varepsilon\\\_\\phi, \\; |

* * *

### 1\. **Offset** ― 位相の絶対誤差

*   **単位** : 秒（典型的に µs, ns 表示）。
    
*   **正負** : ＋ → システム時計が進んでいる / − → 遅れている。
    
*   **実測例** : `Last offset : -0.0000047 seconds` ⇒ −4.7 µs 遅れ。
    
*   **評価基準**
    
    *   GNSS + PPS 同期なら **|offset|≲5 µs** が常用目標。
        
    *   ネット越し NTP だけなら数 ms〜数十 ms でも「同期」と見なす場合がある。
        

### 2\. **Jitter** ― offset の揺らぎを統計量で表す

*   **RMS 定義**なので正値のみ。
    
*   **短期安定度**を示し、電波状態・ケーブル品質・GPIO 割り込み遅延の診断に使う。
    
*   **thumb-rule**
    
    *   GNSS + PPS：1 µs 未満 → 良好。
        
    *   LAN NTP：数 10 µs〜数 100 µs。
        
    *   WAN NTP：>1 ms は普通。
        

### 3\. **Lock** ― 制御理論的な“収束フラグ”

*   **位相比較ループ (PLL)** が**安定極**に入り、位相誤差が許容窓**ε**内に抑え込まれている状態。
    
*   chrony は二段構成
    
    1.  NMEA などで秒境界を粗合わせ
        
    2.  PPS で 1 Hz 位相制御 → 収束判定
        
*   **操作的判定**
    
    *   `chronyc sources` で `*GPS` の `*` が付く
        
    *   `tracking` で “Not synchronized” → “Normal” へ変わる
        

> **注意**: “lock” の文字列は設定ディレクティブ（`lock NMEA`）やハード語法（“PLL lock”）にも出て来るが、概念は共通。“基準に追従状態”という意味。

* * *

4\. 補足：ドリフト (Frequency Error)
-----------------------------

*   **定義** : 系時刻の周波数誤差 \\(\\delta f = \\dfrac{d\\Delta\\phi}{dt}\\) を ppm（1 ppm=1 µs/s）で表す。
    
*   **chrony 出力** : _Frequency_ ／ _Residual freq_.
    
*   **用途** : GPS が途切れた場合、ドリフトが小さいほど offset の増加が遅い。
    

* * *

### 実際の画面例（chrono sources）を読むコツ

```
MS Name/IP         Stratum Poll Reach LastRx Last sample
============================================================================
#* PPS             0       4    377   6     -0.8ns[+25.4ns] /  0.3µs
#? NMEA            0       4    377   5     +85.3ms[+85.3ms] /  1.2ms
```

*   `*` = **Lock 済みで現在選択中**
    
*   `-0.8ns` = **Offset**
    
*   `/ 0.3µs` = **Jitter (RMS)**
    

ここが **“ns オーダ offset & sub-µs jitter → 完全ロック”** を示す瞬間です。

* * *

まとめ
---

| 覚え方 | イメージ |
| --- | --- |
| **Offset** | “今この瞬間ズレてる量” ⇒ 位相差 Δφ |
| **Jitter** | “ズレがどれだけフラつく？” ⇒ Δφ の標準偏差 |
| **Lock** | “既に許容窓内に収まり続けられてる？” ⇒ サーボ収束判定 |

この 3 点を押さえておけば、`chronyc` 出力やデータシートの数値が**精度 (offset)、安定度 (jitter)、同期状態 (lock)** のどれを示しているのか一目で判断できます。

## Prompt:
GPS.md

ファイル

思考時間: 10 秒

ドキュメントを読み込んでいます

完了

## Response:
**Gps**

# GPS関連ワードと複数台同期ハンドブック

---

## 1. 馴染みのない GPS 用語を厳密に定義する

### 1.1 オフセット (Offset)

| 項目                   | 内容                                                                                                           |            |                   |
| -------------------- | ------------------------------------------------------------------------------------------------------------ | ---------- | ----------------- |
| **厳密定義**             | ある瞬間 \(t\) におけるシステム時計 \(T_{sys}(t)\) と基準時計 \(T_{ref}(t)\) の位相差 \(\Delta\phi(t)=T_{sys}(t)-T_{ref}(t)\)。単位は秒。 |            |                   |
| **数式メモ**             | \(\Delta\phi(t)\) が正 → システムが進んでいる，負 → 遅れている。                                                                 |            |                   |
| **イメージ**             | 時計 2 つを並べたときの「時報がどちらが先か」。                                                                                    |            |                   |
| **chrony での表示**      | `chronyc tracking` → *Last offset*。                                                                          |            |                   |
| **目標値 (Pi+GPS+PPS)** | (                                                                                                            | \Delta\phi | \le 5,µs) 程度なら良好。 |

---

### 1.2 ジッター (Jitter)

| 項目              | 内容                                                                                                                  |
| --------------- | ------------------------------------------------------------------------------------------------------------------- |
| **厳密定義**        | 連続観測したオフセット \(\{\Delta\phi_i\}\) の標準偏差 \(\sigma_\phi=\sqrt{\frac1N\sum_{i=1}^N(\Delta\phi_i-\bar{\Delta\phi})^2}\)。 |
| **意味**          | "ブレ" の量。値が小さいほど時刻測定が安定。                                                                                             |
| **単位**          | 秒（µs, ns スケール）。                                                                                                     |
| **chrony での表示** | `chronyc sourcestats` → *Jitter* 列、`tracking` → *Root jitter*。                                                      |
| **目標値**         | GNSS+PPS なら < 1–3 µs。                                                                                               |

---

### 1.3 ロック (Lock)

| 項目            | 内容                                                               |            |                               |                |                                 |
| ------------- | ---------------------------------------------------------------- | ---------- | ----------------------------- | -------------- | ------------------------------- |
| **厳密定義**      | 同期ループ (PI/PLL) が収束し、位相誤差 (                                       | \Delta\phi | <\varepsilon\_\phi) かつ周波数誤差 ( | d\Delta\phi/dt | <\varepsilon\_f) の条件を継続的に満たす状態。 |
| **chrony 指標** | `chronyc sources` の行頭 `*` 印、`tracking` → *Leap status : Normal*。 |            |                               |                |                                 |
| **イメージ**      | イヤホンの Bluetooth が "Connected" 表示になった瞬間。                          |            |                               |                |                                 |
| **実用判断**      | ロック後に offset と jitter が目標内なら時刻基準として使用可。                          |            |                               |                |                                 |

---

## 2. Q&A

### Q2-1. オフセットを求める際の GPS 時刻と UTC の違いは？

- **GPS 時刻 (GPST)** は 1980‑01‑06 00:00:00 を起点とし、*うるう秒を挿入しない* 連続秒数。
- **UTC** は 1 日 = 86,400 秒を維持するために数年おきに ±1 秒のうるう秒を入れる。
- 衛星は *Δt\*\*UTC‑GPS*（現在 18 s）をナビメッセージで配信。受信機は
  \(UTC = GPST - Δt_{UTC‑GPS}\)
  をリアルタイム計算する。よって **どの Pi でも同じ式・同じ値** で変換され、UTC の整数秒は揃う。残る差はローカル要因のみ（数 µs）。

---

### Q2-2. Raspberry Pi 3B を複数台で時刻同期させる手順

1. **ハード配線** (各 Pi 共通)
   ```text
   5V  → GPS VCC
   GND → GPS GND
   GPIO15 (RXD) ← GPS TXD
   GPIO14 (TXD) → GPS RXD (省略可)
   GPIO18        ← 1 PPS
   ```
2. **/boot/config.txt** 追記
   ```ini
   enable_uart=1
   dtoverlay=disable-bt
   dtoverlay=pps-gpio,gpiopin=18,assert_falling_edge=true
   ```
3. **パッケージ**
   ```bash
   sudo apt install gpsd gpsd-tools pps-tools chrony
   ```
4. **gpsd 設定** `/etc/default/gpsd`
   ```
   DEVICES="/dev/ttyAMA0 /dev/pps0"
   GPSD_OPTIONS="-n"
   ```
5. **chrony 設定** `/etc/chrony/chrony.conf`
   ```
   refclock PPS /dev/pps0 lock NMEA refid GPS
   maxslewrate 1000
   smoothtime 400 0.001 leaponly
   ```
6. **再起動 → ロック確認**
   ```bash
   chronyc sources -v  # *GPS 行を確認
   ```

---

### Q2-3. 動作確認テスト手順

| テスト       | コマンド                     | 合格基準                  |          |
| --------- | ------------------------ | --------------------- | -------- |
| PPS 入力    | `sudo ppstest /dev/pps0` | 1 Hz でタイムスタンプが出続ける    |          |
| NMEA 受信   | `gpsmon /dev/ttyAMA0`    | GPGGA/GPRMC に有効な時刻が表示 |          |
| offset 確認 | \`chronyc tracking       | grep 'Last offset'\`  | ±5 µs 以内 |
| 相対誤差      | `ntpdate -q pi-A pi-B`   | 各 Pi 間 offset 差 <5 µs |          |

> **ヒント**: 差分が大きいときはアンテナ視界・電源ノイズ・ケーブル長を疑うこと。

---

### 3. 参考まとめ

- **Offset** … 現在のズレ（位相差）。
- **Jitter** … オフセットがどれだけフラつくか（標準偏差）。
- **Lock** … サーボが収束し続けている状態。
- GPST⇄UTC は衛星が配信する 18 秒差で即時変換 → Pi 間で整数秒ズレは生じない。
- 複数台 Pi を μs オーダで合わせるには **GPS 1 PPS + chrony** が手堅い。

---

## 2. ［追加］ホールドオーバー（GPS遮断）ドリフト試験【任意】

> *この検証は「必須」ではありませんが、********万一 GPS 信号が途切れた場合にどれだけズレが広がるか********を知るうえで有益です。*

### 2.1 テスト目的

- GPS/PPS を一時的に切断し、chrony が内部水晶だけで走ったときの **時刻保持性能 (hold‑over)** を定量化する。
- 複数台運用の場合「どのくらいの時間内なら μs レベルを維持できるか」を把握して運用ポリシーを決める。

### 2.2 手順

1. **平常状態で GPS ロックを確認**
   ```bash
   chronyc tracking | grep 'Last offset'
   ```
2. **GPS/PPS を遮断**
   - アンテナ電源を抜く、または `systemctl stop gpsd`。
3. **定期サンプリング**
   ```bash
   while true; do
       echo -n "$(date +%s) "
       chronyc tracking | awk '/Last offset/ {print $4*1e6}';
       sleep 60
   done | tee drift_log.txt
   ```
4. **グラフ化**（例）
   ```python
   import pandas as pd, matplotlib.pyplot as plt
   df = pd.read_csv('drift_log.txt', sep=' ', names=['epoch','offset_us'])
   plt.plot((df.epoch-df.epoch[0])/60, df.offset_us)
   plt.xlabel('minutes since GPS loss')
   plt.ylabel('offset (µs)')
   plt.show()
   ```
5. **評価指標**
   - 10 分で ±60 µs 以内 → 水晶ドリフト ≃ ±1 ppm
   - 1 時間で ±360 µs 以内 → ±100 ppb 級 TCXO 相当

### 2.3 結果の読み方

| ドリフト傾斜 (ppm) | 解釈           | 運用上のメモ         |
| ------------ | ------------ | -------------- |
| ≈±1 ppm      | 標準 Pi 水晶そのまま | 10 分以内に再ロック推奨  |
| ≈±0.1 ppm    | 温度安定化・TCXO   | 数時間以内なら μs 台維持 |

---

「GPS.md」に厳密な定義・図解イメージ・テスト手順まで組み込みました。続きの修正や追記があれば教えてください！

## Prompt:
以前、GPS時刻同期のテストをしようとして、システム時刻をGPS時刻に同期を行うコマンドを実行した後、GPS時刻とシステム時刻の差を比較し、何秒で解離が広がっていくかを確認していました。この検証は必要なかったですか？

## Response:
思考時間: 6 秒

ユーザーが行ったGPS 時間同期のテストについて、システムクロックとGPSの時間差を確認し、その乖離がどれくらい増加するかを測定したと話しています。このテストは、GPSが切断されたりPPSが使用されていない場合のドリフトを確認するために役立ちます。PPSおよびchronyが正しく同期していれば、乖離は最小限

完了

### 必要だった？ ――「GPS ⇆ システム時刻の乖離テスト」の位置づけ

| ケース | そのテストが **有用** か | 理由 |
| --- | --- | --- |
| **A. chrony が PPS にロックした運用が続く** | ★☆☆ 不要に近い | _chronyc tracking/sources_ ですでに offset・jitter がリアルタイムに監視できる。乖離は常に ±数 µs 以内に抑え込まれるため、わざわざ秒単位で比べても差が見えない。 |
| **B. “ホールドオーバ” (GPS 信号を一時遮断) を想定** | ★★★ 必須 | GPS が途切れた後 **何秒で何 µs ずれるか** は「実機の水晶＋chrony の周波数補正精度」で決まる。運用要件（例：屋内 10 分稼働で ±100 µs 以内）を満たすか確認できる。 |
| **C. PPS 無し／NMEA のみで運用** | ★★☆ 推奨 | 秒境界が NMEA 文字列頼みなので jitter が大きく、長時 間の drift も不安定。乖離テストで「思ったよりずれる」ことを体感する価値がある。 |

結論 : **“常時 PPS ロックで動かす前提なら必須ではない**」けれど、

*   **ホールドオーバ精度**を知りたい
    
*   **PPS 配線が本当に効いているか**クロスチェックしたい  
    場合には依然として役立つテストです。
    

* * *

テストの意味合いを整理
-----------

1.  **周波数誤差 (drift) の検証**
    
    *   _chronyc tracking_ → `Residual freq` ≈ ±0.05 ppm なら、GPS を抜いても **1 分で ±3 µs、10 分で ±30 µs**。
        
    *   実測がそれより大きいなら、水晶温度変化や PPS 品質に問題がある。
        
2.  **PPS ルートの健全性チェック**
    
    *   PPS を外してテストしてみると offset が毎秒±50 µs 以上ずれる → 配線・GPIO・ドライバ設定が生きていることを裏付け。
        
3.  **想定停波時間に対する“持ちこたえ能力”**
    
    *   屋内に持ち込む／屋外アンテナが雪で遮蔽――など現場運用で GPS を失うシナリオへの保険。
        

* * *

もし再テストするなら：簡易手順
---------------

```bash
# 1) lock 状態を確認
chronyc waitsync   # 0 を返すまで待機

# 2) GPS を物理的に抜く、または gpsd を止める
sudo systemctl stop gpsd

# 3) 1 秒ごとに offset を取る
watch -n1 'chronyc tracking | awk "/Last offset/ {printf \"%.3f µs\\n\", \$4*1e6}"'
```

*   ログを取り、線形回帰で **傾き = 実周波数誤差** を求める。
    
*   `Residual freq` の ppm 値から理論オフセットも計算し、グラフで理論線と実線が重なるか確認すると精度チェックになる。
    

* * *

実運用のおすすめ
--------

| 目的 | 方法 |
| --- | --- |
| **常時監視は軽く済ませたい** | `chronyc sourcestats` を 30 s 間隔で Prometheus / Grafana に取り込み、offset・jitter を時系列表示 |
| **停波シミュレーションを自動化** | `gpsd` を systemd timer で 1 日 1 回 10 分停止 → テストログを cron で保存 |
| **温度ドリフトの影響を把握** | `vcgencmd measure_temp` と offset を同時ログし、散布図で相関を見る |

* * *

### TL;DR

*   **PPS が常に生きているなら offset モニタだけで十分**。乖離テストは “保険” もしくは “学習用途”。
    
*   しかし **GPS 遮断時の許容誤差** を定義する運用では、乖離テストは欠かせない実証ステップ。
    
*   テスト結果が chrony の `Residual freq` から導かれる理論値と合えば、システムは期待通りに振る舞っています。
    

2/2

## Prompt:
屋外での運用を想定しているため、PPSに通常はロックされていますか？
もしロックされているのであれば、どのようにしてppsロックで動かしますか？

## Response:
思考時間: 12 秒

屋外では、PPS（Precision Time Protocol）のロックがよく行われますが、そのためにはGPSが3D固定を取得し、PPSラインが正しく設定されている必要があります。gpsdやchronyの設定が適切であれば、PPSロックは安定します。設定後、`chronyc sources`コマンドでPPSのロック状態を確認できます。PPSの行には`*`が表示され、PPSがロックされていない場合の条件も確認しておくことが重要です。これらの設定を自動で実行できるようにする手順もあります。

完了

### 1\. 屋外なら PPS は **ほぼ常時ロック**できる

| 条件 | 現実的な可用率 |
| --- | --- |
| **空が 45° 以上開けた場所**（ビル影・樹木なし） | **99 % 以上**（昼夜問わず） |
| 軽く遮られる（車内・ベランダ手すり程度） | 95 % 前後：DOP が悪化した瞬間に数秒外れることがある |
| 大型車両が横付け、激しい豪雨 | 数 分単位で外れる可能性 |

“ロック”＝**chrony が PPS ソースを選択 (`*`) している状態**。衛星側の UTC ⇄ GPST 変換は整数秒なので失われません。可用率を落とす主因は **視界遮蔽と給電ノイズ** です。

* * *

2\. PPS ロックで動かすまでの完全手順（Pi 3B 例）
-------------------------------

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

*   PPS 出力が **3.3 V かつパルス幅 ≥10 µs** を確認。
    

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

*   `*` が PPS 行に付いたら **PPS ロック成立**。
    
*   数分で `Last sample` の角括弧 \[…\] 部が ±数 µs → ±数 100 ns に収束します。
    

* * *

3\. ロック維持のコツ（屋外設置）
------------------

| 施策 | 効果 |
| --- | --- |
| **広い空視界**（最低 120 ° 開ける） | DOP 安定 → PPS 途切れを防ぐ |
| **アンテナを接地メタル板に載せる** | C/N₀ +3 dB → 仰角低い衛星も捕捉 |
| **5 V/USB 電源を 2 A 以上のノイズ低い物に** | PPS 立ち上がりジッタ低減 |
| **`minpoll 4 / maxpoll 4`** に固定 | chrony が毎 16 s で PPS 位相差を統計処理 → 安定収束 |
| **RTC 併設 (DS3231)** | 再起動直後でも offset < ±100 ms に抑え、ロック時間短縮 |

* * *

4\. ロック状態を自動監視するワンライナー
----------------------

```bash
watch -n 5 'chronyc sources | awk "/^#\*/ {print strftime(), \"LOCK OK\", $0} END {if(NR==0) print strftime(), \"NO LOCK\"}"'
```

*   `LOCK OK` が出続ければ PPS ロック継続。
    
*   `NO LOCK` を検知したらメール通知や LED 点灯などに回せば現場でも安心です。
    

* * *

### TL;DR

1.  **屋外で空が開けていれば PPS は 99 % 以上ロックしっぱなし**。
    
2.  **核心は `/etc/chrony/chrony.conf` の**  
    `refclock PPS /dev/pps0 lock NMEA`  
    これが “PPS ロックで動かす” スイッチ。
    
3.  `chronyc sources` で PPS 行に `*` が付いていれば、もう μs～ns オーダで同期しています。

## Prompt:
![画像をアップロードしました](https://files09.oaiusercontent.com/file-QAYss62XoPmjuxCNjwRd1S?se=2025-05-28T06%3A22%3A03Z&sp=r&sv=2024-08-04&sr=b&rscc=max-age%3D299%2C%20immutable%2C%20private&rscd=attachment%3B%20filename%3D9efc72b6-d57a-426f-83eb-20f9bcdf57de.png&sig=7HpQ8pEu8W6252Iosj/gS%2BKGyNx0OXL2i6zYI/wsOPw%3D)

これまではこの手順で、GPS時刻に設定しようとしていたのですが、ppsロックを用いればこんなことはしなくて良いということですか？

## Response:
### ざっくり回答

はい。\*\*PPS + chrony による「カーネル時刻ディシプリン」\*\*を構成したら、  
画像のように `gpspipe` → `date` で 1 分ごとに手動上書きする運用は **不要** になります。  
むしろ併用すると chrony の制御を乱すので外してください。

* * *

なぜ cron スクリプト方式が要らなくなるのか
------------------------

| 項目 | cron で `date` 書き換え | PPS + chrony |
| --- | --- | --- |
| **仕組み** | 受信した NMEA の文字列を毎分パースし `date -s` で秒単位更新 | 1 PPS パルスの立ち上がりをカーネルでナノ秒刻み捕捉し、chrony が水晶周波数を連続制御 |
| **短期精度 (jitter)** | 10–100 ms（USB / シリアル遅延） | 0.1–5 µs |
| **長期安定度** | 書いた直後は 0 ms、1 分後には水晶ドリフト分だけズレる（±30 µs～数 ms） | 常時 ±数 µs 内に保たれる |
| **副作用** | ・`date` 書き換えの瞬間に時刻がジャンプ  
・アプリのタイムスタンプが乱れる | ジャンプしない（slew 制御） |
| **実装コスト** | スクリプト＋cron 必要 | `gpsd` と `chrony.conf` の 3 行だけ |

* * *

PPS ロックへの切り替え手順（おさらい）
---------------------

1.  **`/boot/config.txt`**
    
    ```ini
    enable_uart=1
    dtoverlay=pps-gpio,gpiopin=18,assert_falling_edge=true
    dtoverlay=miniuart-bt   # Pi3B の例
    core_freq=250
    ```
    
2.  **`/etc/default/gpsd`**
    
    ```text
    DEVICES="/dev/ttyAMA0 /dev/pps0"
    GPSD_OPTIONS="-n"
    ```
    
3.  **`/etc/chrony/chrony.conf`**（末尾に追加）
    
    ```text
    refclock SHM 0 offset 0.0 delay 0.2 refid NMEA
    refclock PPS /dev/pps0 lock NMEA refid GPS
    ```
    
4.  **再起動 → 確認**
    
    ```bash
    chronyc sources -v   # PPS 行に * が付けばロック完了
    ```
    
5.  **昔の cron を削除**
    
    ```bash
    crontab -e   # gpsdata.sh 行をコメントアウト or 削除
    sudo rm /home/pi/gpsdata.sh
    ```
    

* * *

もし「最初の 1 回だけ時刻合わせ」が欲しい場合
------------------------

*   chrony はブート時に **オフセットが 10 min 以内**なら自動 slewing。
    
*   それより大きい場合は `systemd-timesyncd` か `chronyd -s` を使う。
    
*   どうしてもスクリプトを残すなら **`chronyd -q`** を 1 回呼んで終わりにし、以後は cron で `date` を叩かないようにします。
    

* * *

### TL;DR

*   **PPS ロック構成なら cron/`gpsdata.sh` は不要**。
    
*   chrony がナノ秒刻みでカーネルクロックを制御し、常時 ±数 µs で張り付く。
    
*   手動 `date` 上書きは精度も安定度も劣り、逆に悪影響が出るので削除しよう。



---
Powered by [ChatGPT Exporter](https://www.chatgptexporter.com)