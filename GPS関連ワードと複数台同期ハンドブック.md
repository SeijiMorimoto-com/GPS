
---

## 1. 馴染みのない GPS 用語を厳密に定義する

### 1.1 オフセット (Offset)

|項目|内容|||
|---|---|---|---|
|**厳密定義**|ある瞬間 tt におけるシステム時計 Tsys(t)T_{sys}(t) と基準時計 Tref(t)T_{ref}(t) の位相差 Δϕ(t)=Tsys(t)−Tref(t)\Delta\phi(t)=T_{sys}(t)-T_{ref}(t)。単位は秒。|||
|**数式メモ**|Δϕ(t)\Delta\phi(t) が正 → システムが進んでいる，負 → 遅れている。|||
|**イメージ**|時計 2 つを並べたときの「時報がどちらが先か」。|||
|**chrony での表示**|`chronyc tracking` → _Last offset_。|||
|**目標値 (Pi+GPS+PPS)**|(|\Delta\phi|\le 5,µs) 程度なら良好。|

---

### 1.2 ジッター (Jitter)

| 項目              | 内容                                                                                                                                   |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| **厳密定義**        | 連続観測したオフセット {Δϕi}\{\Delta\phi_i\} の標準偏差 σϕ=1N∑i=1N(Δϕi−Δϕˉ)2\sigma_\phi=\sqrt{\frac1N\sum_{i=1}^N(\Delta\phi_i-\bar{\Delta\phi})^2}。 |
| **意味**          | "ブレ" の量。値が小さいほど時刻測定が安定。                                                                                                              |
| **単位**          | 秒（µs, ns スケール）。                                                                                                                      |
| **chrony での表示** | `chronyc sourcestats` → _Jitter_ 列、`tracking` → _Root jitter_。                                                                       |
| **目標値**         | GNSS+PPS なら < 1–3 µs。                                                                                                                |

---

### 1.3 ロック (Lock)

|項目|内容|||||
|---|---|---|---|---|---|
|**厳密定義**|同期ループ (PI/PLL) が収束し、位相誤差 (|\Delta\phi|<\varepsilon_\phi) かつ周波数誤差 (|d\Delta\phi/dt|<\varepsilon_f) の条件を継続的に満たす状態。|
|**chrony 指標**|`chronyc sources` の行頭 `*` 印、`tracking` → _Leap status : Normal_。|||||
|**イメージ**|イヤホンの Bluetooth が "Connected" 表示になった瞬間。|||||
|**実用判断**|ロック後に offset と jitter が目標内なら時刻基準として使用可。|||||

---

## 2. Q&A

### Q2-1. オフセットを求める際の GPS 時刻と UTC の違いは？

- **GPS 時刻 (GPST)** は 1980‑01‑06 00:00:00 を起点とし、_うるう秒を挿入しない_ 連続秒数。
    
- **UTC** は 1 日 = 86,400 秒を維持するために数年おきに ±1 秒のうるう秒を入れる。
    
- 衛星は _Δt**UTC‑GPS_（現在 18 s）をナビメッセージで配信。受信機は  
    UTC=GPST−ΔtUTC‑GPSUTC = GPST - Δt_{UTC‑GPS}  
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

|テスト|コマンド|合格基準||
|---|---|---|---|
|PPS 入力|`sudo ppstest /dev/pps0`|1 Hz でタイムスタンプが出続ける||
|NMEA 受信|`gpsmon /dev/ttyAMA0`|GPGGA/GPRMC に有効な時刻が表示||
|offset 確認|`chronyc tracking|grep 'Last offset'`|±5 µs 以内|
|相対誤差|`ntpdate -q pi-A pi-B`|各 Pi 間 offset 差 <5 µs||

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

> _この検証は「必須」ではありませんが、********万一 GPS 信号が途切れた場合にどれだけズレが広がるか********を知るうえで有益です。_

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

|ドリフト傾斜 (ppm)|解釈|運用上のメモ|
|---|---|---|
|≈±1 ppm|標準 Pi 水晶そのまま|10 分以内に再ロック推奨|
|≈±0.1 ppm|温度安定化・TCXO|数時間以内なら μs 台維持|

---