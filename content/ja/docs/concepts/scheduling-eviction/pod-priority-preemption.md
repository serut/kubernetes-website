---
title: Podの優先度とプリエンプション
content_type: concept
weight: 90
---

<!-- overview -->

{{< feature-state for_k8s_version="v1.14" state="stable" >}}

[Pod](/ja/docs/concepts/workloads/pods/)は _priority_（優先度）を持つことができます。 
優先度は他のPodに対する相対的なPodの重要度を示します。
もしPodをスケジューリングできないときには、スケジューラーはそのPodをスケジューリングできるようにするため、優先度の低いPodをプリエンプトする（追い出す）ことを試みます。



<!-- body -->


{{< warning >}}
クラスターの全てのユーザーが信用されていない場合、悪意のあるユーザーが可能な範囲で最も高い優先度のPodを作成することが可能です。これは他のPodが追い出されたりスケジューリングできない状態を招きます。
管理者はResourceQuotaを使用して、ユーザーがPodを高い優先度で作成することを防ぐことができます。

詳細は[デフォルトで優先度クラスの消費を制限する](/ja/docs/concepts/policy/resource-quotas/#limit-priority-class-consumption-by-default)
を参照してください。
{{< /warning >}}

## 優先度とプリエンプションを使う方法 {#how-to-use-priority-and-preemption}

優先度とプリエンプションを使うには、

1.  1つまたは複数の[PriorityClass](#priorityclass)を追加します

1.  追加したPriorityClassを[`priorityClassName`](#pod-priority)に設定したPodを作成します。
    もちろんPodを直接作る必要はありません。
    一般的には`priorityClassName`をDeploymentのようなコレクションオブジェクトのPodテンプレートに追加します。

これらの手順のより詳しい情報については、この先を読み進めてください。

{{< note >}}
Kubernetesには最初から既に2つのPriorityClassが設定された状態になっています。
`system-cluster-critical`と`system-node-critical`です。
これらは汎用のクラスであり、[重要なコンポーネントが常に最初にスケジュールされることを保証する](/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/)ために使われます。
{{< /note >}}

## PriorityClass

PriorityClassはnamespaceによらないオブジェクトで、優先度クラスの名称から優先度を表す整数値への対応を定義します。
PriorityClassオブジェクトのメタデータの`name`フィールドにて名称を指定します。
値は`value`フィールドで指定し、必須です。
値が大きいほど、高い優先度を示します。
PriorityClassオブジェクトの名称は[DNSサブドメイン名](/ja/docs/concepts/overview/working-with-objects/names#dns-subdomain-names)として適切であり、かつ`system-`から始まってはいけません。

PriorityClassオブジェクトは10億以下の任意の32ビットの整数値を持つことができます。これは、PriorityClassオブジェクトの値の範囲が-2147483648から1000000000までであることを意味します。
それよりも大きな値は通常はプリエンプトや追い出すべきではない重要なシステム用のPodのために予約されています。
クラスターの管理者は割り当てたい優先度に対して、PriorityClassオブジェクトを1つずつ作成すべきです。

PriorityClassは任意でフィールド`globalDefault`と`description`を設定可能です。
`globalDefault`フィールドは`priorityClassName`が指定されないPodはこのPriorityClassを使うべきであることを示します。`globalDefault`がtrueに設定されたPriorityClassはシステムで一つのみ存在可能です。`globalDefault`が設定されたPriorityClassが存在しない場合は、`priorityClassName`が設定されていないPodの優先度は0に設定されます。

`description`フィールドは任意の文字列です。クラスターの利用者に対して、PriorityClassをどのような時に使うべきか示すことを意図しています。

### PodPriorityと既存のクラスターに関する注意

-   もし既存のクラスターをこの機能がない状態でアップグレードすると、既存のPodの優先度は実質的に0になります。

-   `globalDefault`が`true`に設定されたPriorityClassを追加しても、既存のPodの優先度は変わりません。PriorityClassのそのような値は、PriorityClassが追加された以後に作成されたPodのみに適用されます。

-   PriorityClassを削除した場合、削除されたPriorityClassの名前を使用する既存のPodは変更されませんが、削除されたPriorityClassの名前を使うPodをそれ以上作成することはできなくなります。

### PriorityClassの例

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "この優先度クラスはXYZサービスのPodに対してのみ使用すべきです。"
```

## 非プリエンプトのPriorityClass {#non-preempting-priority-class}

{{< feature-state for_k8s_version="v1.24" state="stable" >}}

`preemptionPolicy: Never`と設定されたPodは、スケジューリングのキューにおいて他の優先度の低いPodよりも優先されますが、他のPodをプリエンプトすることはありません。
スケジューリングされるのを待つ非プリエンプトのPodは、リソースが十分に利用可能になるまでスケジューリングキューに残ります。
非プリエンプトのPodは、他のPodと同様に、スケジューラーのバックオフの対象になります。これは、スケジューラーがPodをスケジューリングしようと試みたものの失敗した場合、低い頻度で再試行するようにして、より優先度の低いPodが先にスケジューリングされることを許します。

非プリエンプトのPodは、他の優先度の高いPodにプリエンプトされる可能性はあります。

`preemptionPolicy`はデフォルトでは`PreemptLowerPriority`に設定されており、これが設定されているPodは優先度の低いPodをプリエンプトすることを許容します。これは既存のデフォルトの挙動です。
`preemptionPolicy`を`Never`に設定すると、これが設定されたPodはプリエンプトを行わないようになります。

ユースケースの例として、データサイエンスの処理を挙げます。
ユーザーは他の処理よりも優先度を高くしたいジョブを追加できますが、そのとき既存の実行中のPodの処理結果をプリエンプトによって破棄させたくはありません。
`preemptionPolicy: Never`が設定された優先度の高いジョブは、他の既にキューイングされたPodよりも先に、クラスターのリソースが「自然に」開放されたときにスケジューリングされます。

### 非プリエンプトのPriorityClassの例

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-nonpreempting
value: 1000000
preemptionPolicy: Never
globalDefault: false
description: "この優先度クラスは他のPodをプリエンプトさせません。"
```

## Podの優先度 {#pod-priority}

一つ以上のPriorityClassがあれば、仕様にPriorityClassを指定したPodを作成することができるようになります。優先度のアドミッションコントローラーは`priorityClassName`フィールドを使用し、優先度の整数値を設定します。PriorityClassが見つからない場合、そのPodの作成は拒否されます。

下記のYAMLは上記の例で作成したPriorityClassを使用するPodの設定の例を示します。優先度のアドミッションコントローラーは仕様を確認し、このPodの優先度は1000000であると設定します。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority
```

### スケジューリング順序におけるPodの優先度の効果

Podの優先度が有効な場合、スケジューラーは待機状態のPodをそれらの優先度順に並べ、スケジューリングキューにおいてより優先度の低いPodよりも前に来るようにします。その結果、その条件を満たしたときには優先度の高いPodは優先度の低いPodより早くスケジューリングされます。優先度の高いPodがスケジューリングできない場合は、スケジューラーは他の優先度の低いPodのスケジューリングも試みます。

## プリエンプション

Podが作成されると、スケジューリング待ちのキューに入り待機状態になります。スケジューラーはキューからPodを取り出し、ノードへのスケジューリングを試みます。Podに指定された条件を全て満たすノードが見つからない場合は、待機状態のPodのためにプリエンプションロジックが発動します。待機状態のPodをPと呼ぶことにしましょう。プリエンプションロジックはPよりも優先度の低いPodを一つ以上追い出せばPをスケジューリングできるようになるノードを探します。そのようなノードがあれば、優先度の低いPodはノードから追い出されます。Podが追い出された後に、Pはノードへスケジューリング可能になります。

### ユーザーへ開示される情報

Pod PがノードNのPodをプリエンプトした場合、ノードNの名称がPのステータスの`nominatedNodeName`フィールドに設定されます。このフィールドはスケジューラーがPod Pのために予約しているリソースの追跡を助け、ユーザーにクラスターにおけるプリエンプトに関する情報を与えます。

Pod Pは必ずしも「指名したノード」へスケジューリングされないことに注意してください。スケジューラーは、他のノードに対して処理を繰り返す前に、常に「指定したノード」に対して試行します。Podがプリエンプトされると、そのPodは終了までの猶予期間を得ます。スケジューラーがPodの終了を待つ間に他のノードが利用可能になると、スケジューラーは他のノードをPod Pのスケジューリング先にすることがあります。この結果、Podの`nominatedNodeName`と`nodeName`は必ずしも一致しません。また、スケジューラーがノードNのPodをプリエンプトさせた後に、Pod Pよりも優先度の高いPodが来た場合、スケジューラーはノードNをその新しい優先度の高いPodへ与えることもあります。このような場合は、スケジューラーはPod Pの`nominatedNodeName`を消去します。これによって、スケジューラーはPod Pが他のノードのPodをプリエンプトさせられるようにします。

### プリエンプトの制限

#### プリエンプトされるPodの正常終了

Podがプリエンプトされると、[猶予期間](/ja/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)が与えられます。
Podは作業を完了し、終了するために十分な時間が与えられます。仮にそうでない場合、強制終了されます。この猶予期間によって、スケジューラーがPodをプリエンプトした時刻と、待機状態のPod Pがノード Nにスケジュール可能になるまでの時刻の間に間が開きます。この間、スケジューラーは他の待機状態のPodをスケジュールしようと試みます。プリエンプトされたPodが終了したら、スケジューラーは待ち行列にあるPodをスケジューリングしようと試みます。そのため、Podがプリエンプトされる時刻と、Pがスケジュールされた時刻には間が開くことが一般的です。この間を最小にするには、優先度の低いPodの猶予期間を0または小さい値にする方法があります。

#### PodDisruptionBudgetは対応するが、保証されない

[PodDisruptionBudget](/docs/concepts/workloads/pods/disruptions/) (PDB)は、アプリケーションのオーナーが冗長化されたアプリケーションのPodが意図的に中断される数の上限を設定できるようにするものです。KubernetesはPodをプリエンプトする際にPDBに対応しますが、PDBはベストエフォートで考慮します。スケジューラーはプリエンプトさせたとしてもPDBに違反しないPodを探します。そのようなPodが見つからない場合でもプリエンプションは実行され、PDBに反しますが優先度の低いPodが追い出されます。

#### 優先度の低いPodにおけるPod間のアフィニティ

次の条件が真の場合のみ、ノードはプリエンプションの候補に入ります。
「待機状態のPodよりも優先度の低いPodをノードから全て追い出したら、待機状態のPodをノードへスケジュールできるか」

{{< note >}}
プリエンプションは必ずしも優先度の低いPodを全て追い出しません。
優先度の低いPodを全て追い出さなくても待機状態のPodがスケジューリングできる場合、一部のPodのみ追い出されます。
このような場合であったとしても、上記の条件は真である必要があります。偽であれば、そのノードはプリエンプションの対象とはされません。
{{< /note >}}

待機状態のPodが、優先度の低いPodとの間でPod間の{{< glossary_tooltip text="アフィニティ" term_id="affinity" >}}を持つ場合、Pod間のアフィニティはそれらの優先度の低いPodがなければ満たされません。この場合、スケジューラーはノードのどのPodもプリエンプトしようとはせず、代わりに他のノードを探します。スケジューラーは適切なノードを探せる場合と探せない場合があります。この場合、待機状態のPodがスケジューリングされる保証はありません。

この問題に対して推奨される解決策は、優先度が同一または高いPodに対してのみPod間のアフィニティを作成することです。

#### 複数ノードに対するプリエンプション

Pod PがノードNにスケジューリングできるよう、ノードNがプリエンプションの対象となったとします。
他のノードのPodがプリエンプトされた場合のみPが実行可能になることもあります。下記に例を示します。

*   Pod PをノードNに配置することを検討します。
*   Pod QはノードNと同じゾーンにある別のノードで実行中です。
*   Pod Pはゾーンに対するQへのアンチアフィニティを持ちます (`topologyKey: topology.kubernetes.io/zone`)。
*   Pod Pと、ゾーン内の他のPodに対しては他のアンチアフィニティはない状態です。
*   Pod PをノードNへスケジューリングするには、Pod Qをプリエンプトすることが考えられますが、スケジューラーは複数ノードにわたるプリエンプションは行いません。そのため、Pod PはノードNへはスケジューリングできないとみなされます。

Pod Qがそのノードから追い出されると、Podアンチアフィニティに違反しなくなるので、Pod PはノードNへスケジューリング可能になります。

複数ノードに対するプリエンプションに関しては、十分な需要があり、合理的な性能を持つアルゴリズムを見つけられた場合に、将来的に機能追加を検討する可能性があります。

## トラブルシューティング

Podの優先度とプリエンプションは望まない副作用をもたらす可能性があります。
いくつかの起こりうる問題と、その対策について示します。

### Podが不必要にプリエンプトされる

プリエンプションは、リソースが不足している場合に優先度の高い待機状態のPodのためにクラスターの既存のPodを追い出します。
誤って高い優先度をPodに割り当てると、意図しない高い優先度のPodはクラスター内でプリエンプションを引き起こす可能性があります。Podの優先度はPodの仕様の`priorityClassName`フィールドにて指定されます。優先度を示す整数値へと変換された後、`podSpec`の`priority`へ設定されます。

この問題に対処するには、Podの`priorityClassName`をより低い優先度に変更するか、このフィールドを未設定にすることができます。`priorityClassName`が未設定の場合、デフォルトでは優先度は0とされます。

Podがプリエンプトされたとき、プリエンプトされたPodのイベントが記録されます。
プリエンプションはPodに必要なリソースがクラスターにない場合のみ起こるべきです。
このような場合、プリエンプションはプリエンプトされるPodよりも待機状態のPodの優先度が高い場合のみ発生します。
プリエンプションは待機状態のPodがない場合や待機状態のPodがプリエンプト対象のPod以下の優先度を持つ場合には決して発生しません。そのような状況でプリエンプションが発生した場合、問題を報告してください。

### Podはプリエンプトされたが、プリエンプトさせたPodがスケジューリングされない

Podがプリエンプトされると、それらのPodが要求した猶予期間が与えられます。そのデフォルトは30秒です。
Podがその期間内に終了しない場合、強制終了されます。プリエンプトされたPodがなくなれば、プリエンプトさせたPodはスケジューリング可能です。

プリエンプトさせたPodがプリエンプトされたPodの終了を待っている間に、より優先度の高いPodが同じノードに対して作成されることもあります。この場合、スケジューラーはプリエンプトさせたPodの代わりに優先度の高いPodをスケジューリングします。

これは予期された挙動です。優先度の高いPodは優先度の低いPodに取って代わります。

### 優先度の高いPodが優先度の低いPodより先にプリエンプトされる

スケジューラーは待機状態のPodが実行可能なノードを探します。ノードが見つからない場合、スケジューラーは任意のノードから優先度の低いPodを追い出し、待機状態のPodのためのリソースを確保しようとします。
仮に優先度の低いPodが動いているノードが待機状態のPodを動かすために適切ではない場合、スケジューラーは他のノードで動いているPodと比べると、優先度の高いPodが動いているノードをプリエンプションの対象に選ぶことがあります。この場合もプリエンプトされるPodはプリエンプトを起こしたPodよりも優先度が低い必要があります。

複数のノードがプリエンプションの対象にできる場合、スケジューラーは優先度が最も低いPodのあるノードを選ぼうとします。しかし、そのようなPodがPodDisruptionBudgetを持っており、プリエンプトするとPDBに反する場合はスケジューラーは優先度の高いPodのあるノードを選ぶこともあります。

複数のノードがプリエンプションの対象として利用可能で、上記の状況に当てはまらない場合、スケジューラーは優先度の最も低いノードを選択します。

## Podの優先度とQoSの相互作用 {#interactions-of-pod-priority-and-qos}

Podの優先度と{{< glossary_tooltip text="QoSクラス" term_id="qos-class" >}}は直交する機能で、わずかに相互作用がありますが、デフォルトではQoSクラスによる優先度の設定の制約はありません。スケジューラーのプリエンプションのロジックはプリエンプションの対象を決めるときにQoSクラスは考慮しません。
プリエンプションはPodの優先度を考慮し、優先度が最も低いものを候補とします。より優先度の高いPodは優先度の低いPodを追い出すだけではプリエンプトを起こしたPodのスケジューリングに不十分な場合と、`PodDisruptionBudget`により優先度の低いPodが保護されている場合のみ対象になります。

kubeletは[node-pressureによる退避](/docs/concepts/scheduling-eviction/node-pressure-eviction/)を行うPodの順番を決めるために、優先度を利用します。QoSクラスを使用して、最も退避される可能性の高いPodの順番を推定することができます。
kubeletは追い出すPodの順位付けを次の順で行います。

  1. 枯渇したリソースを要求以上に使用しているか
  1. Podの優先度
  1. 要求に対するリソースの使用量

詳細は[kubeletによるPodの退避](/docs/concepts/scheduling-eviction/node-pressure-eviction/#pod-selection-for-kubelet-eviction)を参照してください。

kubeletによるリソース不足時のPodの追い出しでは、リソースの消費が要求を超えないPodは追い出されません。優先度の低いPodのリソースの利用量がその要求を超えていなければ、追い出されることはありません。より優先度が高く、要求を超えてリソースを使用しているPodが追い出されます。


## {{% heading "whatsnext" %}}

* PriorityClassと関連付けてResourceQuotaを使用することに関して [デフォルトで優先度クラスの消費を制限する](/ja/docs/concepts/policy/resource-quotas/#limit-priority-class-consumption-by-default)
* [Podの破壊](/docs/concepts/workloads/pods/disruptions/)を読む
* [APIを起点とした退避](/ja/docs/concepts/scheduling-eviction/api-eviction/)を読む
* [Node-pressureによる退避](/docs/concepts/scheduling-eviction/node-pressure-eviction/)を読む
