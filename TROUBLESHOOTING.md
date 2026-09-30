# トラブルシューティング記録

書籍「AWS IoT実践講座」のハンズオンを実施する過程で遭遇した、書籍執筆時点からの環境差異と、その特定・対応の記録です。クラウド技術書籍のハンズオンでは、執筆時点から時間が経つほどライブラリやサービス仕様が変化するため、都度一次情報（公式ドキュメント・Pricing API・実際のエラーメッセージ）を確認しながら対応する必要がありました。

## 1. Amazon Timestream for LiveAnalyticsの新規作成終了

書籍では時系列データベースとしてTimestreamの利用を想定した記載がありましたが、確認したところ新規作成ができなくなっていました。CloudWatch Metrics / Timestream for InfluxDB / Amazon Managed Prometheus / DynamoDB転用の4案を比較し、Grafanaとの親和性・構成のシンプルさからCloudWatch Metricsを採用しました。

## 2. `smart_open`ライブラリのAPI変更

推論結果を可視化するノートブックのセルで、以下のエラーが2段階で発生しました。

```python
for line in smart_open.smart_open("s3://{}/{}".format(bucket_name,prefixKey)):
    _list_line = line.decode().replace('\n','').split(',')
```

1. `AttributeError: module 'smart_open' has no attribute 'smart_open'`
   → `smart_open`はv2.0以降、トップレベル関数名が`smart_open.smart_open()`から`smart_open.open()`に変更されている。
2. `smart_open.open()`に修正すると `AttributeError: 'str' object has no attribute 'decode'`
   → 新しい`smart_open.open()`はデフォルトでテキストモード(`'r'`)を返すため、`.decode()`が不要になっている（旧関数はデフォルトがバイナリモード`'rb'`だった）。

**対応**:
```python
for line in smart_open.open("s3://{}/{}".format(bucket_name,prefixKey)):
    _list_line = line.replace('\n','').split(',')
```

## 3. SageMaker Notebook Instanceのデフォルトカーネルに不足しているパッケージ

データ加工用ノートブックの実行時、`matplotlib`・`seaborn`（`ModuleNotFoundError`）、`ipywidgets`（`tqdm.notebook`が要求する`IProgress`が見つからず`ImportError`）が未インストールだった。`!pip install`セルへの追加とカーネル再起動で解決。

## 4. AWS Step Functionsの新規ステートマシンがデフォルトでJSONata形式になっている

書籍サンプルのASL定義ファイル（JSONPath形式: `Parameters` + `.$`サフィックス + `States.Format`組み込み関数）を元に、AWSコンソールのWorkflow Studioで新規ステートマシンを作成したところ、以下のバリデーションエラーが発生した。

```
/States/CreateTransformJob/Arguments: The field 'TransformJobName' is required but was missing
/States/CreateTransformJob/Arguments: The field 'TransformOutput.S3OutputPath' is required but was missing
```

**原因の切り分け**: エラーメッセージのパスが`/Arguments`となっている点に着目し、「コード」タブでステートマシン定義を確認したところ、末尾に`"QueryLanguage": "JSONata"`が自動付与されていた。これはAWSコンソールが新規作成するステートマシンのデフォルトが、近年追加された**JSONata形式**（`Parameters`ではなく`Arguments`、動的値は`{% ... %}`という別記法）になっているためで、書籍のASLファイルが前提とするclassic JSONPath形式と非互換だった。Workflow Studioのフォーム入力経由では、`States.Format`を使う動的フィールドがJSONata形式にうまく変換されず欠落していた。

**対応**: 「コード」タブのエディタ内容を、書籍のASLファイル（プレースホルダーを実値に置換したもの）で全置換。これにより`QueryLanguage`の指定自体が入らず、classic JSONPath形式で動作するようになった。

## 5. SageMaker Python SDK v2 非推奨警告

`RandomCutForest`を使った学習ジョブ実行時、v3の`ModelTrainer`への移行を促す`SageMakerV2DeprecationWarning`が出力される（動作自体には影響なし）。

## 6. S3の再帰コピーによる意図しないネスト

`aws s3 cp --recursive`で、コピー元(`csv/3rd_test/`)とコピー先(`csv/3rd_test/2026/9/29/`)が入れ子関係にあったところ、大量オブジェクトのページネーション処理中にコピー先の新規オブジェクトを再度ソースとして拾ってしまい、二重にネストしたファイル（`csv/3rd_test/2026/9/29/2026/9/29/...`）が生成された。

**対応**: `aws s3api list-objects-v2`で対象キーを静的に列挙してから、1件ずつ`aws s3 cp`で個別コピーする方式に変更し解消。以降、送信元・送信先が親子関係になるS3コピーでは、この個別コピー方式を採用する運用ルールとした。

## 7. AWS IoT Rule / Firehose用IAMロールの伝播遅延

Firehose配信ストリーム作成直後に「Firehose is unable to assume role ... Please check the role provided.」というエラーが発生。IAMロールの信頼ポリシーをCLIで確認したところ設定自体は正しく、`RoleLastUsed`が空（未使用）だったことから、IAMロール作成直後の伝播遅延と判断。1〜2分待って再試行し解決した。
