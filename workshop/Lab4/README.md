# Lab4) Helmチャートを使用したアプリケーションのデプロイ

Lab4では、Kubernetesのパッケージング技術の1つである [Helm](https://helm.sh/) を利用したデプロイの方法を学びます。

1. レガシーなJava（J2EE）のWebアプリケーションであるJPetStoreをDockerコンテナ化（ハンズオンでは実施しません）
2. Helmを利用して [IBM Cloud Kubernetes Service](https://www.ibm.com/cloud/container-service) にデプロイ

![](images/jpet-architecture.png)

## ソースコードの入手
Lab4，5に使用するリポジトリをクローンします:

```bash
$ git clone https://github.com/kissyyy/jpetstore-kubernetes
$ cd jpetstore-kubernetes
```

#### フォルダーの構成
クローンしたリポジトリは以下のファイルから構成されています。

| フォルダー | 説明 |
| ---- | ----------- |
|[**jpetstore**](/jpetstore)| Javaでかかれたペットショップのアプリケーション |
|[**mmssearch**](/mmssearch)| 新規にGOで開発したマイクロサービス。Watsonを使った画像認識の機能が実装されている |
|[**helm**](/helm)| KubernetesにデプロイするためのHelm チャート |
|[**pet-images**](/pet-images)| 動作確認用の動物画像ファイル |

## 既存アプリケーションのコンテナ化
Lab4では既に公開済みのPublic Imageを利用します。
そのため以下の操作は実施する必要はありませんが興味のある方はぜひ試してみてください。

（時間があったら書きます）

## アプリケーションのデプロイ

### Helmを利用したデプロイ
Helmは、Kubernetesのパッケージ・マネージャーです。 Helm チャートと呼ばれる定義ファイルを使用して、Kubernetes アプリケーションの定義やインストール、アップグレードを行うことができます。

#### Helmのセットアップ（これ必要か？要検証）
Helm チャートを IBM Cloud Kubernetes Service で使用するために、クラスターに Helm インスタンスをインストールして初期化する必要があります。

[Helm](https://docs.helm.sh/using_helm/#installing-helm)をインストールします。

クラスターのセキュリティーを維持するため、IBM Cloud kube-samples リポジトリーから以下の .yaml ファイルを適用することによって、Tiller のサービス・アカウントを kube-system 名前空間に作成し、tiller-deploy ポッドに対する Kubernetes RBAC クラスター役割バインディングを作成します。 注: kube-system 名前空間のサービス・アカウントとクラスター役割バインディングを使用して Tiller をインストールするには、cluster-admin 役割が必要です。

```bash
$ kubectl apply -f https://raw.githubusercontent.com/IBM-Cloud/kube-samples/master/rbac/serviceaccount-tiller.yaml
```

作成したサービス・アカウントを使用して、Helm を初期化し、tiller をインストールします。

```bash
$ helm init --service-account tiller
```

クラスター内の tiller-deploy ポッドの「状況」が「実行中」になっていることを確認します。

```bash
$ kubectl get pods -n kube-system -l app=helm
```

#### Helmを利用したアプリのデプロイ

Helm チャートを使用してJPetStore アプリをデプロイします。

```bash
$ cd ../helm

# Helm 初期化
$ helm init
```

```bash
# JPetstoreアプリをインストール
$ helm install --name jpetstore ./modernpets
NAME:   jpetstore-helm
LAST DEPLOYED: Wed Feb 13 15:14:08 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Service
NAME  CLUSTER-IP     EXTERNAL-IP  PORT(S)       AGE
web   172.21.60.105  <nodes>      80:30327/TCP  1s
db    172.21.53.155  <none>       3306/TCP      1s

==> v1beta2/Deployment
NAME                                    KIND
jpetstore-helm-modernpets-jpetstoreweb  Deployment.v1beta2.apps

==> v1beta1/Deployment
NAME                                   DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
jpetstore-helm-modernpets-jpetstoredb  1        1        1           0          1s
```

### 補足: YAMLファイルを使用したデプロイ

Lab3までと同様、yamlファイルを使用してデプロイすることも可能です。
yamlファイルでデプロイをする場合は以下のようになります。

```bash
$ kubectl apply -f jpetstore.yaml
```

JpetStoreのWebコンテナとDBコンテナがデプロイされます。

```bash
$ kubectl get all
```

## 動作確認

ブラウザ上で以下のURLからjpetアプリの動作をテストします:
`<クラスターのPublic IP>:<ポート>`にアクセスしてください。

   ![](images/petstore.png)

以上でLab4は終了です。
最後のハンズオンは[Lab5](../Lab5/README.md)です。
