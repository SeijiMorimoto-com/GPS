# Raspberry Pi 3B での GPS 時刻同期・複数台運用ガイド
## 1. 目的
- 各 Pi を **GPS 1 PPS + chrony** でストラタム 1 化し、研究装置間のタイムスタンプを ±数 µs 以内に揃える。

## 2. ハード構成
| パーツ | 推奨例 | 数量 / Pi |
|--------|-------|-----------|
| Raspberry Pi 3B | – | 1 |
| u‑blox M8N 受信モジュール | PPS 出力必須 | 1 |
| アクティブ GPS アンテナ | 3 m 延長可 | 1 |
| 杜若線 (GPIO 18) | PPS |

### 配線
```
 Pi 3B Pin   GPS Module
 -------------------------
 5 V   (PIN 4)  -> VCC
 GND   (PIN 6)  -> GND
 GPIO15 RXD (PIN10) <- TXD
 GPIO14 TXD (PIN 8) -> RXD
 GPIO18        (PIN12) <- 1 PPS
```

## 3. OS 設定
`/boot/config.txt`
```ini
enable_uart=1
init_uart_baud=9600
dtoverlay=disable-bt          # BT 無効化で PL011 を確保
dtoverlay=pps-gpio,gpiopin=18,assert_falling_edge=true
```
`cmdline.txt` から `console=serial0...` を削除。

## 4. パッケージ
```bash
sudo apt update
sudo apt -y install gpsd gpsd-tools pps-tools chrony
```
`/etc/default/gpsd`
```
DEVICES="/dev/ttyAMA0 /dev/pps0"
GPSD_OPTIONS="-n"
```
`/etc/chrony/chrony.conf`
```
refclock PPS /dev/pps0 lock NMEA refid GPS
maxslewrate 1000
smoothtime 400 0.001 leaponly
```

## 5. 動作確認
```bash
gpsmon -n          # NMEA 流入確認
sudo ppstest /dev/pps0
chronyc sources -v # *GPS となればロック
```

## 6. 精度評価スクリプト
各 Pi で:
```bash
#!/bin/bash
chronyc tracking | awk '/Last offset/ {print systime(), $4*1e6}'
```
同時実行し CSV を比較 → 差分 <5 µs で合格。

LAN 経由チェック例:
```bash
ntpdate -q 192.168.0.101 192.168.0.102 | grep offset
```

## 7. 複数台ロールアウト手順
1. ゴールデン SD カードを `rpi-imager` の “clone” で複製  
2. 個別に `hostnamectl set-hostname pi-gps-XX`  
3. 監視サーバへ `log2ram` + `prometheus-node-exporter` 導入

## 8. 運用 Tips
- アンテナは金属板（地盤面）に置くと利得 +3 dB。  
- GPS ロックまで 30 s～2 min。UPS 起動時のウォームアップを考慮。  
- `chronyc sourcestats` で jitter を常時監視し、>10 µs なら警報。

## 9. トラブルシューティング
| 症状                | 原因候補           | 対策                                            |
| ----------------- | -------------- | --------------------------------------------- |
| `/dev/pps0` が無い   | DTO Overlay ミス | `dtoverlay` 行を再確認                             |
| jitter が 50 µs 以上 | マルチパス or 電源ノイズ | アンテナ位置変更、5 V 供給強化                             |
| Pi 間差 >50 µs      | 片方だけ GPS ロック外れ | `chronyc tracking` で “Not synchronised” をチェック |

## 10. 参考リンク
- chrony FAQ “How accurate can I get with PPS?”
- u‑blox M8 Receiver Description »
