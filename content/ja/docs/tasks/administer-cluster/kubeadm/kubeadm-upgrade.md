---
title: kubeadmクラスターのアップグレード
content_type: task
weight: 30
---

<!-- overview -->

このページでは、kubeadmを用いて構築されたKubernetesクラスターのアップグレード方法を説明します。
対象のバージョンは、バージョン{{< skew currentVersionAddMinor -1 >}}.xからバージョン{{< skew currentVersion >}}.x、およびバージョン
{{< skew currentVersion >}}.xからバージョン{{< skew currentVersion >}}.y(`y > x`)です。 
アップグレードの際にマイナーバージョンをスキップすることはサポートされていません。
詳細は、[バージョンスキューポリシー](/releases/version-skew-policy/)を確認してください。

古いkubeadmのバージョンを使用して構築したクラスターのアップグレードにおける情報を確認するには、代わりに以下のページを参照してください。

- [{{< skew currentVersionAddMinor -2 >}}から{{< skew currentVersionAddMinor -1 >}}へのkubeadmクラスターのアップグレード](https://v{{< skew currentVersionAddMinor -1 "-" >}}.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [{{< skew currentVersionAddMinor -3 >}}から{{< skew currentVersionAddMinor -2 >}}へのkubeadmクラスターのアップグレード](https://v{{< skew currentVersionAddMinor -2 "-" >}}.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [{{< skew currentVersionAddMinor -4 >}}から{{< skew currentVersionAddMinor -3 >}}へのkubeadmクラスターのアップグレード](https://v{{< skew currentVersionAddMinor -3 "-" >}}.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [{{< skew currentVersionAddMinor -5 >}}から{{< skew currentVersionAddMinor -4 >}}へのkubeadmクラスターのアップグレード](https://v{{< skew currentVersionAddMinor -4 "-" >}}.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

Kubernetesプロジェクトでは、速やかに最新のパッチリリースへアップグレードし、サポートされているKubernetesのマイナーリリースが実行されていることを確認することを推奨しています。
この推奨事項に従うことで、セキュリティを維持できます。

アップグレードの大まかな手順を以下に示します。

1. 最初のコントロールプレーンノードをアップグレードする。
1. 2番目以降のコントロールプレーンノードをアップグレードする。
1. ワーカーノードをアップグレードする。

## {{% heading "prerequisites" %}}

- [リリースノート](https://git.k8s.io/kubernetes/CHANGELOG)を必ずよく読んでください。
- クラスターは静的なコントロールプレーンと、etcd podもしくは外部etcdを使用している必要があります。
- データベースに保存されているアプリレベルの状態など、重要なコンポーネントは必ずバックアップしてください。
  `kubeadm upgrade`はワークロードには影響せず、Kubernetesの内部コンポーネントのみを対象とします。ただし、バックアップは常にベストプラクティスです。
- [スワップは無効にする必要があります](https://serverfault.com/questions/684771/best-way-to-disable-swap-in-linux)。

### Additional information

- The instructions below outline when to drain each node during the upgrade process.
  If you are performing a **minor** version upgrade for any kubelet, you **must** first drain the node (or nodes) that you are upgrading.
  In the case of control plane nodes, they could be running CoreDNS Pods or other critical workloads.
  詳細な情報は、 [ノードのDrain](/docs/tasks/administer-cluster/safely-drain-node/)を参照してください。
- The Kubernetes project recommends that you match your kubelet and kubeadm versions.
  You can instead use a version of kubelet that is older than kubeadm, provided it is within the
  range of supported versions.
  詳細は、[kubeletに対するkubeadmのバージョンの差異](/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#kubeadm-s-skew-against-the-kubelet)を参照してください。
- アップグレード後はコンテナイメージのハッシュ値が変化するため、全てのコンテナが再起動されます。
- kubeletのアップグレード後にkubeletサービスが再起動が成功したか確認するには、`systemctl status kubelet`を実行するか、`journalctl -xeu kubelet`を実行してサービスのログを表示させてください。
- `kubeadm upgrade`は、アップグレードプロセスに使用する[`UpgradeConfiguration` APIタイプ](/docs/reference/config-api/kubeadm-config.v1beta4)として、`--config`オプションをサポートします。
- `kubeadm upgrade`は、既存のクラスターの再設定はサポートしていません。
   代わりに、[kubeadmクラスターの再設定](/docs/tasks/administer-cluster/kubeadm/kubeadm-reconfigure)の手順を参照してください。

### etcdのアップグレードにおける考慮事項

Because the `kube-apiserver` static pod is running at all times (even if you
have drained the node), when you perform a kubeadm upgrade which includes an
etcd upgrade, in-flight requests to the server will stall while the new etcd
static pod is restarting.
As a workaround, it is possible to actively stop the `kube-apiserver` process a few seconds before starting the `kubeadm upgrade apply` command.
This permits to complete in-flight requests and close existing connections, and minimizes the consequence of the etcd downtime.
This can be done as follows on control plane nodes:

```shell
killall -s SIGTERM kube-apiserver # kube-apiserverのグレースフルなシャットダウンをトリガーします
sleep 20 # 処理中のリクエストが完了するまで少し待ちます
kubeadm upgrade ... # kubeadm upgradeコマンドを実行します
```

<!-- steps -->

## パッケージリポジトリの変更

コミュニティ所有のパッケージリポジトリ(`pkgs.k8s.io`)を使用している場合、必要なKubernetesのマイナーリリースのパッケージリポジトリを有効化する必要があります。
これは、[Kubernetesパッケージリポジトリの変更](/docs/tasks/administer-cluster/kubeadm/change-package-repository/)
ドキュメントにて説明されています。

{{% legacy-repos-deprecation %}}

## アップグレード先のバージョンの決定

OSのパッケージマネージャーを使用し、Kubernetes {{< skew currentVersion >}}の最新のパッチリリースを確認してください。

{{< tabs name="k8s_install_versions" >}}
{{% tab name="Ubuntu, Debian or HypriotOS" %}}

```shell
# Find the latest {{< skew currentVersion >}} version in the list.
# It should look like {{< skew currentVersion >}}.x-*, where x is the latest patch.
sudo apt update
sudo apt-cache madison kubeadm
```

{{% /tab %}}
{{% tab name="CentOS, RHEL or Fedora" %}}

```shell
# Find the latest {{< skew currentVersion >}} version in the list.
# It should look like {{< skew currentVersion >}}.x-*, where x is the latest patch.
sudo yum list --showduplicates kubeadm --disableexcludes=kubernetes
```

{{% /tab %}}
{{< /tabs >}}

If you don't see the version you expect to upgrade to, [verify if the Kubernetes package repositories are used.](/docs/tasks/administer-cluster/kubeadm/change-package-repository/#verifying-if-the-kubernetes-package-repositories-are-used)

## コントロールプレーンノードのアップグレード

コントロールプレーンノードのアップグレード手順は、一度に1ノードずつ実行する必要があります。
最初にアップグレードするコントロールプレーンノードを選択してください。そのノードには`/etc/kubernetes/admin.conf`ファイルが存在している必要があります。

### "kubeadm upgrade"の実行

**最初のコントロールプレーンノード**

1. kubeadmのアップグレード:

   {{< tabs name="k8s_install_kubeadm_first_cp" >}}
   {{% tab name="Ubuntu, Debian or HypriotOS" %}}

   ```shell
   #「{{< skew currentVersion >}}.x-*」の「x」は最新のパッチバージョンに置き換えてください。
   sudo apt-mark unhold kubeadm && \
   sudo apt-get update && sudo apt-get install -y kubeadm='{{< skew currentVersion >}}.x-*' && \
   sudo apt-mark hold kubeadm
   ```

   {{% /tab %}}
   {{% tab name="CentOS, RHEL or Fedora" %}}

   ```shell
   #「{{< skew currentVersion >}}.x-*」の「x」は最新のパッチバージョンに置き換えてください。
   sudo yum install -y kubeadm-'{{< skew currentVersion >}}.x-*' --disableexcludes=kubernetes
   ```

   {{% /tab %}}
   {{< /tabs >}}

1. kubeadmがダウンロードされており、期待どおりのバージョンであることを確認します。

   ```shell
   kubeadm version
   ```

1. アップグレードの計画を確認します。

   ```shell
   sudo kubeadm upgrade plan
   ```

   This command checks that your cluster can be upgraded, and fetches the versions you can upgrade to.
   It also shows a table with the component config version states.

   {{< note >}}
   `kubeadm upgrade` also automatically renews the certificates that it manages on this node.
   To opt-out of certificate renewal the flag `--certificate-renewal=false` can be used.
   For more information see the [certificate management guide](/docs/tasks/administer-cluster/kubeadm/kubeadm-certs).
   {{</ note >}}

1. Choose a version to upgrade to, and run the appropriate command. For example:

   ```shell
   # replace x with the patch version you picked for this upgrade
   sudo kubeadm upgrade apply v{{< skew currentVersion >}}.x
   ```

   Once the command finishes you should see:

   ```
   [upgrade/successful] SUCCESS! Your cluster was upgraded to "v{{< skew currentVersion >}}.x". Enjoy!

   [upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
   ```

   {{< note >}}
   v1.28以前のバージョンでは、kubeadmはデフォルトで、他のコントロールプレーンノードがアップグレードされていない場合でも、`kubeadm upgrade apply`の実行時にアドオン(CoreDNSやkube-proxyを含む)を直ちにアップグレードするモードになっていました。
   これにより互換性の問題が発生する可能性があります。
   v1.28以降では、kubeadmはデフォルトで、アドオンのアップグレードを開始する前に、全てのコントロールプレーンノードがアップグレード済みか確認するモードになりました。
   コントロールプレーンノードのアップグレードは順次実行するか、少なくとも他のすべてのコントロールプレーンノードが完全にアップグレードされるまで最後のノードのアップグレードを開始しないようにしてください。
   アドオンのアップグレードは、最後のコントロールプレーンノードのアップグレード完了後に実行されます。
   {{</ note >}}

1. Manually upgrade your CNI provider plugin.

   Your Container Network Interface (CNI) provider may have its own upgrade instructions to follow.
   Check the [addons](/docs/concepts/cluster-administration/addons/) page to
   find your CNI provider and see whether additional upgrade steps are required.

   This step is not required on additional control plane nodes if the CNI provider runs as a DaemonSet.

**2番目以降のコントロールプレーンノード**

最初のコントロールプレーンノードと同様ですが、下記のコマンドの代わりに、

```shell
sudo kubeadm upgrade apply
```

こちらのコマンドを実行してください。

```shell
sudo kubeadm upgrade node
```

また、`kubeadm upgrade plan`の実行や、CNIプロバイダープラグインのアップグレードは不要です。

### ノードのDrain実行

ノードをスケジュール不可としてマークし、ワークロードを削除することで、ノードのメンテナンスの準備をします。

```shell
# <node-to-drain>は、drainを実施するノード名に置き換えてください。
kubectl drain <node-to-drain> --ignore-daemonsets
```

### kubeletとkubectlのアップグレード

1. kubeletとkubectlをアップグレードします。

   {{< tabs name="k8s_install_kubelet" >}}
   {{% tab name="Ubuntu, Debian or HypriotOS" %}}

   ```shell
   #「{{< skew currentVersion >}}.x-*」の「x」は最新のパッチバージョンに置き換えてください。
   sudo apt-mark unhold kubelet kubectl && \
   sudo apt-get update && sudo apt-get install -y kubelet='{{< skew currentVersion >}}.x-*' kubectl='{{< skew currentVersion >}}.x-*' && \
   sudo apt-mark hold kubelet kubectl
   ```

   {{% /tab %}}
   {{% tab name="CentOS, RHEL or Fedora" %}}

   ```shell
   #「{{< skew currentVersion >}}.x-*」の「x」は最新のパッチバージョンに置き換えてください。
   sudo yum install -y kubelet-'{{< skew currentVersion >}}.x-*' kubectl-'{{< skew currentVersion >}}.x-*' --disableexcludes=kubernetes
   ```

   {{% /tab %}}
   {{< /tabs >}}

1. kubeletを再起動します。

   ```shell
   sudo systemctl daemon-reload
   sudo systemctl restart kubelet
   ```

### ノードのオンライン化

ノードをスケジュール可能としてマークすることで、ノードをオンラインに戻します。

```shell
# <node-to-uncordon>は対象のノード名に置き換えてください
kubectl uncordon <node-to-uncordon>
```

## ワーカーノードのアップグレード

ワーカーノードのアップグレード手順は、ワークロードの実行に必要な最小限のキャパシティを損なうことなく、一度に1ノードずつ、または数ノードずつ実行する必要があります。

以下のページでは、LinuxおよびWindowsワーカーノードのアップグレード方法を説明します。

* [Linuxノードのアップグレード](/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/)
* [Windowsノードのアップグレード](/docs/tasks/administer-cluster/kubeadm/upgrading-windows-nodes/)

## クラスターの状態の確認

After the kubelet is upgraded on all nodes verify that all nodes are available again by running
the following command from anywhere kubectl can access the cluster:

```shell
kubectl get nodes
```

The `STATUS` column should show `Ready` for all your nodes, and the version number should be updated.

## Recovering from a failure state

If `kubeadm upgrade` fails and does not roll back, for example because of an unexpected shutdown during execution, you can run `kubeadm upgrade` again.
This command is idempotent and eventually makes sure that the actual state is the desired state you declare.

To recover from a bad state, you can also run `sudo kubeadm upgrade apply --force` without changing the version that your cluster is running.

During upgrade kubeadm writes the following backup folders under `/etc/kubernetes/tmp`:

- `kubeadm-backup-etcd-<date>-<time>`
- `kubeadm-backup-manifests-<date>-<time>`

`kubeadm-backup-etcd` contains a backup of the local etcd member data for this control plane Node.
In case of an etcd upgrade failure and if the automatic rollback does not work, the contents of this folder
can be manually restored in `/var/lib/etcd`. In case external etcd is used this backup folder will be empty.

`kubeadm-backup-manifests` contains a backup of the static Pod manifest files for this control plane Node.
In case of a upgrade failure and if the automatic rollback does not work, the contents of this folder can be
manually restored in `/etc/kubernetes/manifests`. If for some reason there is no difference between a pre-upgrade
and post-upgrade manifest file for a certain component, a backup file for it will not be written.

{{< note >}}
After the cluster upgrade using kubeadm, the backup directory `/etc/kubernetes/tmp` will remain and
these backup files will need to be cleared manually.
{{</ note >}}

## コマンドの動作内容

`kubeadm upgrade apply`では以下の処理を行っています。

- クラスターがアップグレード可能な状態かチェックする:
  - APIサーバーが疎通可能であること
  - 全てのノードが`Ready`状態であること
  - コントロールプレーンに障害がないこと
- バージョンスキューポリシーを適用する
- コントロールプレーンのイメージが利用可能、もしくはコマンド実行中のマシンから取得可能であることを確認する
- コンポーネントの設定にアップグレードが必要な場合、新しい設定に置換する、および/またはユーザー提供の上書き設定を使用する
- コントロールプレーンのコンポーネントをアップグレードする、もしくはいずれかが失敗した場合はロールバックする
- 新しい`CoreDNS`と`kube-proxy`のマニフェストを適用し、必要な全てのRBACルールが作成されているか確認する
- APIサーバー用の新しい証明書と秘密鍵のファイルを作成し、180日間以内に有効期限が切れるファイルがあればバックアップする

2番目以降のコントロールプレーンノードにおいて、`kubeadm upgrade node`は以下の処理を行っています。

- クラスターから、kubeadmの`ClusterConfiguration`を同期する
- 任意として、kube-apiserverの証明書のバックアップする
- コントロールプレーンノードのコンポーネント用のstatic Podのマニフェストをアップグレードする
- コマンドを実行しているノードのkubeletの設定をアップグレードする

ワーカーノードにおいて、`kubeadm upgrade node`は以下の処理を行っています。

- クラスターから、kubeadmの`ClusterConfiguration`を同期する
- コマンドを実行しているノードのkubeletの設定をアップグレードする
