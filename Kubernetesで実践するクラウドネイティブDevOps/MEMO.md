# 2 Kubernetes最初の一歩

スタンドアロンで動かすときはminikube

立てたpodにアクセスするには

```
kubectl port-forward hogehoge 9999:8888
```

# 3 Kubernetes環境の選択

Kubernetesがコンテナのスケジューリングやサービス管理、APIリクエスト処理を実行するためのコントロールプレーンがある

ユーザのワークロードを実行するクラスタメンバはワーカーノードという

コントロールプレーンは高可用性を備えている

耐障害性を確保するため、複数のワーカーノード、アベイラビリティゾーンを分けるなど対処しておく

本番運用ではマネージドのほうが良さげ。むしろ特別な理由がないのであればセルフホスティングしないほうがいい

* GKE (google)
* EKS (amazon)
* AKE (azure)

原則

* 実行するソフトウェアを減らす
* 標準のテクノロジを選ぶ
* 差別化につながらない面倒な作業はアウトソーシングする
* 長続きする競争優位性を生み出す

# 4 Kubernetesオブジェクトの基本操作

Deployment: Kubernetesのスーパバイザ

コントローラ: 状態を望ましい状態と一致することを保証する

ReplicaSet: 同一Pod、レプリカのグループを管理する

スケジューラ: スケジューリングされていないPodのキューを監視して、そこから次のPodを取り出して実行場所となるノードを見つける

Service: PodのIPのかわりに単一の変化しないIPアドレス、DNS名が与えられ、任意の対応するPodへ自動的にルーティングされる

* port: Serviceの提供するポート
* targetPort: 対応するPodのポート
* selectorで対応するPodを指定

Helm:
  チャート: アプリケーションを実行するための必要なリリース
  リポジトリ: チャートを収集、共有できる場所
  リソース: 実行されるチャートの特定のインスタンス

# 5 リソースの管理

リソース制限: 割り当てられた制限を超えてCPUを使用しようとするPodはスロットルの対象となる
割り当てられた制限を超えてメモリを使用しようとするPodは強制終了される

Kubernetesではリソースのオーバーコミットは許容される

Podの強制終了が必要な場合、リソース要求を最も超過しているPodから強制終了する

コンテナは小さく保つ

Readiness probe: get podsに表示されるREADYの状態はこれが完了したかどうか

namespaceを分割してもservice同士は通信可能

DNS名は SERVICE.NAMESPACE.svc.cluster.local

同一名称のサービスでnamespaceをまたがるサービスにアクセスしたい場合 SERVICE.NAMESPACEでアクセス可能

namespaceでquotaを設定できる。ただしCPU,memoryを制限するのはおすすめできない

Namespace内のリソース制限、要求はデフォルトLimitRangeから取得するが明示的に記載すべき

コンテナのリソース制限は一般的に通常運用の最大量より少し多い値に設定する

node内のPodは10-100くらい

ガベージコレクションでディスクスペースが不足し始めると未使用のイメージは自動的にクリーンアップされる

# 6 クラスタの運用

クラスタのキャパシティ計画は重要

従来型のサーバでアプリケーションを動かす場合どのくらいの台数が必要か？から算出する

Kubernetesクラスタで耐障害性を実現する場合、マスタノードの最少数は3(リーダ選出の関係で2代は不適切)

ワークロードがフォールトレラントであるためには少なくとも2つのワーカーノードが必要

クラスタが大きくなるほど、マスタノードにかかる負荷は大きくなる

そのため大規模な運用が必要な場合、フェデレーションさせることによりワークロードをクラスタ間で複製できる。

これは2つ以上のクラスタを同期させて同じワークロードを実行できる

とはいえそこまで使うシチュエーションはない

ノードの追加は容易に行える。スケールダウンはドレインが行われ、ワークロードが他のノードに確実に移行されるようにする。

Kubernetesとワークロードに関してK8Guardというツールで問題をチェックできる

sonobuoyでkubernetes適合性テストを実行できる

Copperでマニフェストをデプロイする前のチェックをしてくれる

kube-benchでクラスタがセキュリティに関するベストプラクティスに準拠しているかチェックできる

監査ロギングを有効にすると、クラスタAPIに対するすべてのリクエストがログに記録される

chaosnuke,chaosmonkeyでクラスタの耐障害性をテストできる

kube-monkeyはプリセットされた時刻に実行され、対象のPodを対象にテストする

# 7 Kubernetesの強力なツール

kubectlについて

--selectorフラグでLabelに一致するリソースを対象として操作できる

リソースタイプは省略形がある

* po pod
* deploy deployment
* svc service
* ns namespace
など

--watchフラグでリソースの変更を監視できる

インフラにおいて命令的なコマンドを使う問題点は、唯一の信頼できる情報源が存在しないこと

コマンドを実行した後、何らかのマニフェストが実行されると宣言的コマンドが上書きされる

マニフェストの作成で`--dry-run -o yaml`を活用する

kubectl attachでPodにアタッチできる
kubectl execで任意のコマンドを実行できる

`alias bb=$kubectl run busybox --image=busybox:1.28 --rm -it --restart=Never$`

こんなかんじのaliasを設定しておくと便利

`bb nslookup demo` など

# 8 コンテナの実行

PodとはKubernetesにおけるスケジューリングの単位

KubernetesではPodが最小のデプロイ単位

理想はコンテナが1つである状態。

portsフィールドはアプリケーションが待ち受けるポート番号を記載するが、実際はなくても動く、ただし書くのがベストプラクティス

envで環境変数を引き渡せるがConfigMap,Secretを使うほうが良い

ユーザ指定はDockerfileでもできるがrunAsUserを使うほうが柔軟にできる

securityContext:
  runasnonroot: rootで起動しようとするコンテナを拒否する
  readOnlyRootFilesystem: コンテナ自身のファイルシステムに書き込むことを拒否する
  allowPrivilegeEscalation: 特権モードでの実行を拒否する
  capabilities: ケイパビリティの追加、削除
すべてのPodとコンテナでセキュリティコンテキストを設定するのがベストプラクティス

PodSecurityPolicyリソースを使用すればクラスタレベルで設定できるのでPodごとに設定が不要で楽

serviceAccountNameでPodにサービスアカウントを割当できる。他のNamespaceでPodを表示したりなど

ボリューム:
  empryDir: Podのライフサイクルに紐づく一時的なディスク
  PersistentVolumeClaimオブジェクトを作成することにより、永続的なディスクを追加できる
  
# 9 Podの管理

Label: Podなどのオブジェクトにふかされるキーと値ペア. ユーザにとって意味や関連性があるもの、コアシステムにとってのセマンティックスを直接的に構成するわけではないオブジェクト識別属性

つまりユーザにとって意味があるがシステム的には意味を持たない情報でリソースをタグ付けする

セレクタ: Labelとの一致条件を記述する式、リソースのグループをLabelによって指定する手段,ServiceなどでPodとリンクさせるなどの使い方

annotation: labelとの違いはリソースを選別すること。annotationは識別には用いられない情報であり、Kubernetes外部のツールやサービスが使用する

Node Affinity: Podを特定のノードにスケジューリングするための機能

Pod Affinity: 特定のPod同士を同居させたり、別のNodeにスケジューリングすることを強制させたりできる。
  大抵はスケジュラーがよしなにやってくれるが、問題を把握しており、解決方法がこれに限られる場合、利用するのが良い

Taint: ノードが一連のPodを寄せ付けないようにできる

Toleration: Taintを許容するようにPodに指示できる

DaemonSet: Elasticsearchなどにログを送信したいときに。クラスタ内のここのノードにPodを配置する

StatefulSet: Podを特定の順序で起動、終了する機能。redisクラスタなど起動順序が厳密に必要な場合など

Job: Podを指定した回数だけ実行する。その後、Jobは完了したとみなす.起動トリガはJobマニフェストの作成時、

CronJob: Jobを定期的に実行する

Horizontal Pod Autoscaler: サービスの需要に応じてPodの数を自動的にスケールする

水平とはサービスのインスタンス数を増減させること

スケールで対象とするメトリクスはKubernetesのメトリクスなら何でも可能、一般的にはCPU

スケール対象はDeployment

OperatorとCustom Resource Definition(CRD): Kubernetesのリソースを拡張するための仕組み

新しいタイプのオブジェクトを独自に作成できる

Ingress: Serviceの前に配置されるロードバランサ

TLS終端にもできる。その場合証明書はSecretに格納する

cert-managerで証明書は自動で更新,発行できる

Ingressコントローラ: Ingressリソースの管理、NGINX Ingressなどが代表

Istio: サービスメッシュを提供する。複数のアプリケーションが相互に通信し、依存する場合に

Envoy: 高度なロードバランシングを実現

# 10 設定と機密情報

configMap: YAMLのリテラル値として記載する方法、kubectlから直接作成する方法、ファイルから作成する方法がある

valueFromでリテラル値をとるのではなく別の場所を検索して値を見つける必要があることを指示する

secretな情報はconfigmapではなくsecretで扱う

secretはbase64でエンコードされているだけなので、機密情報を扱う場合は注意

Secretのアクセス権の付与はRBACが行う、ない場合は任意のコンテナからアクセスできてしまう

manifest内での機密情報の扱い方

* バージョン管理リポジトリに格納する際に暗号化、複合する
* 機密情報のリモート保存

軽量なツールにSOPSなど

# 11 セキュリティとバックアップ

クラスタのセキュリティを確保するために実施できる一番カンタンで効率的な方法はクラスタにアクセスできるユーザを制限すること

Kubernetesクラスタにアクセスする必要があるユーザのグループは一般的にクラスタ運用者とアプリケーション開発者

ロールベースのアクセス制御(RBAC)

クラスタ内で特定の操作を誰が実行できるか制御できる

有効かどうかは`kubectl describe pod -n kube-system -l component=kube-apiserver`で確認できる

RBACにはユーザ、ロール、ロールバインディングの概念がある

Kubernetesではパーミッションは加法的

cluster-admin=root

デフォルトではすべてのPodはそれらが作成されるNamespaceのdefaultサービスアカウントとして実行され、いかなるロールとも関連付けされていない

クラスタで実行しているソフトのセキュリティチェックをする必要がある

Clair: コンテナスキャナ
など

バックアップ

必要性、データが永続ボリュームに保存されている場合など

etcdに障害が発生した場合、クラスタの状態は破壊される

Veleroなどのツールがある

監視

`kubectl get componentstatuses`で確認できる

`kubectl get nodes`でノードのステータスが確認できる

`kubectl top`でCPU,メモリのキャパシティと現在の使用率を確認できる

Kubernetes DashboardというWebベースの管理ツールがある

# 12 Kubernetesアプリケーションのデプロイ

YAMLファイルでの管理は保守が難しくなる

HelmテンプレートはGoのテンプレートエンジンを利用している

Helmでは他のHelmテンプレートとの依存関係を管理できる

`helm upgrade`コマンドでアプリケーションの一部の値を変更できる

HelmFileでクラスタにインストールする必要のある一式すべてを単一のコマンドでデプロイ可能

kustomizeはオーバレイを用いることで異なる環境向けのマニフェストをパッチとして適用する

# 13 開発ワークフロー

Skaffold: 開発ワークフローを迅速化する。コンテナレジストリとの間のデプロイ操作の手間を削減

Draft: コードの変更時に更新版をクラスタに自動的にデプロイするためにHelmを利用できる

Knative: サーバレススタイルの機能も提供

Kubernetesにおけるアプリケーションのデプロイ戦略はDeploymentマニフェストで定義する。

maxSurgeは超過して作成されるPodの最大数を設定する。10のPodがあり、maxSurgeを30%に設定すると一度に13個までのPodが実行されることを許可する。

maxUnavailableは利用できないPodの最大値を設定する。10のPodがあり20%が設定されていれば利用可能なPodが8を下まわらないようにする

ブルーグリーンデプロイ: 新旧のバージョンが混在して同時にリクエストを処理する状況を作らない。

Helmではデプロイの過程で処理が発生する順序をコントロールできる。

# 14 Kubernetesにおける継続的デプロイ

継続的デプロイとは、成功したビルドを本番へ自動的にデプロイする仕組み

デプロイも一元的かつ自動的に管理する必要がある

CDの大きなメリットは本番で驚かないこと。ステージング段階でテストに成功押している完璧なバイナリイメージでなければ本番にデプロイされることはない。

# 15 オブザーバビリティと監視

オブザーバビリティとはシステムの監視を超えた幅広い世界を表現する概念

ブラックボックス監視: システムの外から挙動を確認する

欠点として、
* 予測可能な障害しか検出できない
* システムのうち、外部に露出する部分の挙動しかチェックできない
* 基本的に問題が発生した後でしか通知しない
* 何が壊れているかはわかるが、なぜか？という点はわからない

ロギング

何を記録し、何をしないかはアプリケーション開発者に委ねられる

なのでブラックボックス型のチェックと同じく答えられる疑問は事前に予測できるものになる

異なる形式でログを書き出すので情報の抽出が難しい

何でも記録するとノイズが増える

トラフィック量に応じたスケールが困難

メトリクス

なにかの状態を測定した数値

* 現時点で処理中のリクエスト数
* 毎分のリクエスト処理数
* リクエスト処理中に発生したエラーの数
* リクエスト処理に要した平均時間

メトリクスはなぜか？という疑問の解決、問題の予測に役立つ

トレーシング

メトリクスはシステムの個別のコンポーネントで何が起こっているかを示すが、トレーシングは単一のリクエストがたどるライフサイクル全体を明らかにする

オブザーバビリティ

オブザーバビリティとは測定の歯やすさがどれほど担保されているか？内部で何が起こっているかをどれほど容易に判断できるか？を示す尺度

オブザーバビリティは理解を促進する。特定の機能の性能を改善する方に設計されたコードをロールアウトした場合、それが意図したとおりに機能するか？など

オブザーバビリティパイプライン

データソースを送信先から切り離してバッファを提供する。すべてのデータをパイプラインに送信すればパイプラインがよしなに処理してくれる

# 16 Kubernetesにおけるメトリクス

メトリクスとは？特定の対象を測定した数値、特定の瞬間における状況のスナップショット

どのような種類の数値か？カウンタとゲージ

カウンタ：増加のみ。

ゲージ: 増減する。booleanなどもこちら

メトリクスは異常や障害を教えてくれる

物事が適切に機能しているかも教えてくれる(ユーザ数など)

関心対象となるメトリクスにフォーカスする必要がある

いくつかパターンがある

* REDパターン
  * リクエスト
  * 失敗するリクエスト数（エラー）
  * 持続時間(dulation)

* USEパターン
  * 関心事はサービスではなくリソース
  * CPU,Memory,ディスクなどの
    * 使用率
    * 飽和度
    * エラー
  * 処理が拘束で可視性が高いことが利点(問題を見過ごす可能性が低い)

* ビジネスメトリクス
  * SaaSサービスのようなサブスクリプションビジネスにおいては加入者のデータを把握する必要がある
  * ファネル分析(ランディングページ、サインアップなど)
  * 解約率
  * 顧客あたりの収益
  * サポートの有用性
  * システムステータスページのトラフィック量(機能停止やサービス劣化の場合、スパイクが発生する)

KubernetesにおいてはcAdvisorというツールが書くクラスタノードで実行されている

kube-state-metricsというツールでKubernetes自体を監視することができる

クラスタの健全性に関わるメトリクス

* ノード数
* ノードの健全性ステータス
* ノードあたり、全体のPod数
* ノードあたり、全体のリソース使用率

Deploymentに関するメトリクス

オートスケールのオプションを有効にしているなら時間経過で追跡できるようにしておく

利用できないレプリカに関するデータはクラスタのキャパシティに関しての問題の把握に使える

コンテナに関するメトリクス

再起動回数はリソース制限の超過の情報になるので頻度を取得する

その他バグの可能性も示唆できる

アプリケーションに関するメトリクス

アプリケーションの機能が実際にどのようなものかによって異なる。パターン参照

開発の初期においては将来的にわからない可能性がある、その場合はとりあえず記憶しておく

## メトリクスの分析

一つの数値に集約する方法

中央値: 値の集合を2つの同数の部分に分割する値

単純平均: ハズレ値の影響を受けやすい

パーセンタイルの把握: オブザーバビリティに関して測定される頻度が高いパーセンタイルはP90,P95,P99

## メトリクスのダッシュボードによるグラフ化

場合によってはアラートなど

ダッシュボードのレイアウトは合わせておく

複数サービスにまたがる情報は情報ラジエータを利用する(大画面で表示して複数チーム全員から参照できる)

表示すべきデータは不可欠の情報のみにしておく


## メトリクスに基づくアラート

アラートとは安定している状態から想定外の逸脱が発生していることを示す

分散システムにおいては基本的にそうした状態はない。(ほとんど常に部分的に劣化した状態)

アラートを上げる際は信号帯雑音の比率を大きくしないといけない

アラートを上げるということは直ちに人間が対処する必要があるという単純な事実のみでなければならない

オンコール対応をなるべく減らす。対応にはハードリミットを設ける

呼び出し対象とならないアラートは非同期通知にする

## メトリクス管理とツールサービス

prometheusがデファクト




