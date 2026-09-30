# AWS IoT実践講座 ハンズオン実装記録

書籍「[AWS IoT実践講座](https://gihyo.jp/book/2024/978-4-297-14518-7)」（技術評論社）のハンズオンを通じて実装した、IoTデバイスのデータ収集・可視化基盤、および機械学習を用いた設備予兆保全システムの記録です。書籍発行後の環境変化（ライブラリのAPI変更、AWSサービスのデフォルト仕様変更、サービス提供状況の変化）に対応しながら実装したため、その過程で得たトラブルシューティングの記録もあわせて公開しています。

元となる書籍リポジトリ: [tsugunao/IoTBook](https://github.com/tsugunao/IoTBook)

## 実装した内容

### 1. IoTデータ収集・可視化基盤

ESP32デバイス（DHT11温湿度センサー）から取得したデータを、AWS IoT Core経由でクラウドに送信し、ダッシュボードで可視化するパイプラインです。

```
ESP32(DHT11) --MQTT(相互TLS認証)--> AWS IoT Core --Rules Engine--> CloudWatch Metrics --> Amazon Managed Grafana
```

- ESP-IDF(C言語) + AWS IoT Device SDKによる、証明書ベースの相互TLS認証でのMQTT通信
- IoT Coreルールエンジン(SQL)によるメトリクス振り分け
- Amazon Managed Grafanaでの温度・湿度ダッシュボード構築

### 2. 機械学習を用いた設備予兆保全システム

ベアリング(軸受)の振動データから異常の予兆を検知し、担当者にメール通知する、実運用相当のサーバーレスパイプラインです。

```
[データ収集]
ESP32(加速度センサー) --MQTT--> AWS IoT Core --Rules Engine--> Amazon Data Firehose --Lambda(JSON→CSV変換)--> S3

[モデル構築]
NASA Bearing Dataset --データ加工(Notebook)--> Amazon SageMaker Random Cut Forest(教師なし異常検知モデル)

[予兆検知パイプライン(日次バッチ)]
EventBridge Scheduler --起動--> AWS Step Functions
  → Lambda(実行日時取得) → SageMaker バッチ変換(異常度スコア推論) → Lambda(異常判定) → Amazon SNS(メール通知)
```

- NASA Bearing Datasetを用いた教師なし異常検知モデル(Random Cut Forest)の学習・評価
- Step Functions + SageMakerバッチ変換 + Lambda + SNSによる、日次実行の予兆検知パイプライン
- **正常系・異常系の両方でエンドツーエンドの動作確認済み**（異常系データでは異常度スコア2.06〜3.03を検出し、閾値0.75を大きく超過してアラートメールが送信されることを確認）

## 使用技術

| 分野 | 技術 |
|---|---|
| 組み込み開発 | ESP32 / ESP-IDF(C) / AWS IoT Device SDK / MQTT相互TLS認証 |
| IoT基盤 | AWS IoT Core(Rules Engine) / Amazon Data Firehose |
| 可視化 | Amazon Managed Grafana / Amazon CloudWatch |
| 機械学習 | Amazon SageMaker(Random Cut Forest、バッチ変換) / 教師なし異常検知 |
| サーバーレス | AWS Lambda / AWS Step Functions / Amazon EventBridge Scheduler / Amazon SNS |
| データ | Amazon S3 / NASA Bearing Dataset(IMS) |

## ディレクトリ構成

```
esp32/examples/
├── dht11/            # 温湿度センサーのデバイス側プログラム(書籍サンプルからの修正版)
└── dummy_adxl345/    # 加速度センサー(ダミーデータ)のデバイス側プログラム(修正版)

predictive_maintenance/
├── notebook/
│   ├── section6-data-processing.ipynb   # NASA Bearing Datasetの推論用データ加工
│   └── section6_nasa-bearing.ipynb      # 異常検知モデルの構築・評価
└── sfn/
    └── anomaly-detection.asl.json       # 予兆検知パイプラインのStep Functionsステートマシン定義

TROUBLESHOOTING.md   # 書籍執筆時点からの環境差異と対応内容の記録
```

## 主なトラブルシューティング

書籍発行から時間が経過したことで生じた環境差異への対応を [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) にまとめています。一次情報（AWS公式ドキュメント・Pricing API等）を確認しながら、以下のような問題を特定・解決しました。

- Amazon Timestream for LiveAnalyticsの新規作成終了への対応（代替サービスの比較検討・設計変更）
- Pythonライブラリの破壊的API変更への対応（`smart_open` v2.0でのAPI変更等）
- AWS Step Functionsの新規ステートマシンのデフォルトクエリ言語がJSONataに変更されており、書籍のJSONPath形式定義と非互換だった問題の切り分けと回避

## 注意事項

本リポジトリには、AWSアカウントID・S3バケット名・デバイス証明書・Wi-Fi認証情報などの機密情報は含まれていません（プレースホルダーに置き換え、または除外しています）。実際に動かす場合は、各自の環境情報に置き換えてご利用ください。

## 参考

- [書籍公式ページ（技術評論社）](https://gihyo.jp/book/2024/978-4-297-14518-7)
- [書籍サンプルコードリポジトリ](https://github.com/tsugunao/IoTBook)
