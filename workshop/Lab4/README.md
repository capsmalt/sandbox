# Lab4: JPetStore on IBM Cloud Kubernetes Service

既存のJavaアプリをコンテナ化してKubernetesにデプロイします。
このLabでは、Kubernetesのパッケージング技術の1つである Helm をしたデプロイの方法を学びます。

1. レガシーなJava（J2EE）のWebアプリケーションであるJPetStoreをDockerコンテナ化（ハンズオンでは実施しません）
2. [IBM Cloud Kubernetes Service](https://www.ibm.com/cloud/container-service) にデプロイ

![](readme_images/architecture.png)

## 事前準備

本ハンズオンの実施には以下が必要になります。
Lab1~3をすでに完了している場合は追加の操作は不要です。

1. CLIのインストール
If you do not have Docker or Kubernetes tooling installed, see [Setting up the IBM Cloud Developer Tools CLI](https://cloud.ibm.com/docs/cli/idt/setting_up_idt.html).

2. Kubernetesクラスターの作成[IBM Cloud Kubernetes Service](http://www.ibm.com/cloud/container-service) のセットアップと and [provision a Standard **Paid** cluster](https://cloud.ibm.com/docs/containers/container_index.html#clusters) (it might take up to 15 minutes to provision, so be patient). A Free cluster will *not* work because this demo uses Ingress resources.

3. Follow the instructions in the **Access** tab of your cluster to gain access to your cluster using [**kubectl**](https://kubernetes.io/docs/reference/kubectl/overview/).

### ソースコードの入手

このリポジトリをクローンします:

```bash
git clone https://github.com/kissyyy/jpetstore-kubernetes
cd jpetstore-kubernetes
```

#### ソースコードの構成

| フォルダー | 説明 |
| ---- | ----------- |
|[**jpetstore**](/jpetstore)| Traditional Java JPetStore application |
|[**mmssearch**](/mmssearch)| New Golang microservice (which uses Watson to identify the content of an image) |
|[**helm**](/helm)| Helm charts for templated Kubernetes deployments |
|[**pet-images**](/pet-images)| Pet images (which can be used for the demo) |

## アプリケーションのデプロイ

Kubernetesにデプロイする方法として、2つの方法があります。
There are two different ways to deploy the three micro-services to a Kubernetes cluster:

- Using [Helm](https://helm.sh/) to provide values for templated charts (recommended)
- Or, updating yaml files with the right values and then running  `kubectl create`

### オプション 1: Helmを利用してデプロイ

1. Install [Helm](https://docs.helm.sh/using_helm/#installing-helm). (`brew install kubernetes-helm` on MacOS)

2. Find your **Ingress Subdomain** by running `ibmcloud cs cluster-get YOUR_CLUSTER_NAME` , it will look similar to "mycluster.us-south.containers.mybluemix.net".

3. Open `../helm/modernpets/values.yaml` and make the following changes.

    - Update `repository` and replace `<NAMESPACE>` with your Container Registry namespace.
    - Update `hosts` and replace `<Ingress Subdomain>` with your Ingress Subdomain.

4. Repeat the previous step and update `../helm/mmssearch/values.yaml` with the same changes.

5. Next, install JPetStore and Visual Search using the helm yaml files you just created:

    ```bash
    # Change into the helm directory
    cd ../helm

    # Initialize helm
    helm init

    # Create the JPetstore app
    helm install --name jpetstore ./modernpets

    # Ceate the MMSSearch microservice
    helm install --name mmssearch ./mmssearch
    ```

### オプション 2: YAMLファイルを使用してデプロイ

yamlファイルを使用して宣言的にデプロイを実行します。

1. Edit **jpetstore/jpetstore.yaml** and **jpetstore/jpetstore-watson.yaml** and replace all instances of:

  - `<CLUSTER DOMAIN>` with your Ingress Subdomain (`ibmcloud cs cluster-get CLUSTER_NAME`)
  - `<REGISTRY NAMESPACE>` with your Image registry URL. For example:`registry.ng.bluemix.net/mynamespace`

2. `kubectl create -f jpetstore.yaml`  - This creates the JPetstore app and database microservices
3. `kubectl create -f jpetstore-watson.yaml`  - This creates the MMSSearch microservice

## 動作確認

You are now ready to use the UI to shop for a pet or query the store by sending it a picture of what you're looking at:

1. Access the java jpetstore application web UI for JPetstore at `http://jpetstore.<Ingress Subdomain>/shop/index.do`

   ![](readme_images/petstore.png)

## アプリの変更

mmssearch.go or index.htmlを変更し再度docker buildする。
ビルドはibmclodu cr buildで実行

## Troubleshooting

### The toolchain complains about "incompatible versions" for helm

The DEPLOY log shows:
```
Error: UPGRADE FAILED: incompatible versions client[v2.8.1] server[v2.4.2]
```

It means you have already `helm` installed in your cluster and this version is not compatible with the one from the toolchain.

Two options:
1. Before running the toolchain again, update `helm` to the version used by the toolchain on your local machine then issue a `helm init --upgrade` against the cluster.
2. Edit `bluemix/pipeline-DEPLOY.sh` at line 60 in your repository and replace `helm init` with `helm init --upgrade`.

Then re-run the DEPLOY job.

