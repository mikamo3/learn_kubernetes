# 2.2プロダクションレディなアプリケーション運用の実現

nodeメンテのため、kubectl drainでノード上のPodを退避する

その際、どのnodeにイルカを意識しないとサービスが止まってしまう(同一nodeにすべてのPodがいる,他のPodで起動中のときdrainしてしまうなど)

PodDisruptionBudgetを指定すると最低限必要なPodの数を指定でき、可用性を維持できる

# 2.2.2 PriorityClassによる起動優先順位の定義

ノードのリソースが不足しているとき、通常は新たにPodを配置できないが、PriorityClassにより優先度を高くしたPodは他のPodを退避して起動できる

```
$ kubectl get priorityclass
```

で確認できる

PriorityClassNameが設定されていない、かつglobalDefaultがtrueで設定されたものが存在しない場合、0

Podの優先度は

```
kubectl describe pod
```

で確認可能

# 2.2.3 各種Probeによるヘルスチェックの実現

* ReadinessProbe: Podがリクエストを受け付けるか確認する。失敗したらService配下からはずれる
* LivenessProbe: Podが実行中か確認する。失敗したらkubeletにより停止されるrestartPolicyがAlways,OnFailureの場合、再作成される
* StartupProve: 初回起動できたか判定、失敗したときの挙動はLivenessProbeと同じ

## チェック方法

* コマンド実行
* HTTPリクエスト
* TCP
* gRPC

* failureThreshold: チェックに失敗可能な回数
* periodSeconds: チェック間隔
* initialDelaySeconds: チェック開始するまでの秒数

# 2.2.4 preStopによる安全な停止タイミング

停止時はPod内のアプリケーションにSIGTERMが飛ぶ

それで終了が確認できなければ30秒後にSIGKILLが飛ぶ

非同期で行われるのでService配下から外れていないにもかかわらずPodが停止してしまう可能性がある

preStopを設定するとSIGTERMが飛ぶ前にコマンドを実行できる

それ以外にhttpGetの設定も可能

# 2.2.5 GracePeriodによる停止期間の設定

`.spec.terminationGracePeriodSeconds`で、SIGKILLまでの時間の設定が可能

上記設定値を超えたらpreStopの処理が終わりSIGKILLされる

# 2.2.6 分散スケジューリングの実現

特定のノードで実行したいということを実現できる

nodeSelector,nodeAffinityで設定する

podAntiAffinityでノードを分散してPodを配置できる.PodAffinityはその逆ができる

## nodeAffinity

requireDuringSchedulingIgnoredDuringExecutionで設定された条件を満たすことが必須

preferredDuringSchedulingIgnorerdDuringExecutionは優先される

## podAffinity/podAntiAffinity

nodeAffinity同様の必須、優先設定が可能

# 2.3 Kubernetesにおけるバッチ処理



