# Lab5: JPetStore on IBM Cloud Kubernetes Service

Lab4で作成したJpetStoreアプリケーションを拡張して、 IBM Cloudの画像認識サービス([Watson Visual Recognition](https://www.ibm.com/watson/services/visual-recognition/))と連携させます。
このLabでは外部APIとの連携の方法を学びます。

![](readme_images/architecture.png)

## つくるもの

Lab4ではWebとDBから構成されるレガシーなJavaアプリケーションをコンテナ化し、Kubernetesにデプロイしました。
これにより、既存のアプリケーションがモダナイズ（Modernize)できたことになります。

Lab5ではモダナイズされたアプリを拡張し、新しい機能をマイクロサービスとして実装します。
具体的にはGo言語で書かれたチャットアプリに、画像認識の機能を追加したアプリケーション（mmsserach）が該当します。

## Visual Recognitionサービスの作成

ブラウザで [IBM Cloudのカタログページ](https://cloud.ibm.com/catalog/) にアクセスし、Visual Recognitionサービスを作成します。
サービスを作成する地域は「米国南部(Dallas)」、サービスプランは「ライト（Lite）」を選択し、「作成」をクリックします。

IBM Cloudのライト・プランは無料で利用可能です。

サービスが作成されると画面が遷移し、サービスの詳細画面が表示されます。このページで、API呼び出しをするために必要な**APIキー**が取得できます。このAPIキーはこの後使うのですぐに参照できるようにしておいてください。

## Kubernetes secret の作成

Kubernetes上のアプリケーションから外部APIを呼び出す際

`Configmap` と `Secret` の2種類の方法がありますが、APIキーのようなより機密性のな情報についてはSecretを使用することが推奨されています。

テンプレートファイル**mms-secrets.json.template**を使用して、**mms-secrets.json** ファイルを作成します:

   ```bash
   # from jpetstore-kubernetes directory
   cd mmssearch
   cp mms-secrets.json.template mms-secrets.json
   ```

エディタで**mms-secrets.json** を開き、先ほど作成したVisual RecognitionのAPIキーを貼り付けます。

   ![](readme_images/watson_credentials.png)


`kubectl`コマンドを使用してKuberenetesクラスターのSecretを生成します。

```bash
# from the jpetstore-kubernetes directory
cd mmssearch
kubectl create secret generic mms-secret --from-file=mms-secrets=./mms-secrets.json
```

Secretが正しく生成できているか確認します。

```bash
kubectl get secret
```

```bash
kubectl get secret mms-secret -o yaml
```

mms-search欄にはランダムの文字列が並んでいますが、これはAPIキーがBase64 エンコードされているものです。
なお、Configmapとして生成した場合は暗号化されず平文のまま値が格納されます。

これで、Visual RecognitionのAPIキーがSecretとしてKubernetesクラスターに渡されました。
実際にアプリケーションから読み出す方法はSecretをVolumeとしてマウントする方法と、環境変数としてSecretを参照する方法があります。
今回のアプリでは以下のようにVolumeとしてマウントする方法で実装されています。

```yaml
    spec:
        volumeMounts:
         - name: service-secrets
           mountPath: "/etc/secrets"
           readOnly: true
      volumes:
      - name: service-secrets
        secret:
          secretName: mms-secret
          items:
          - key: mms-secrets
            path: mms-secrets.json
`
```

これにより`/etc/secret/mms-secrets.json`がマウントされます。

### 補足

IBM Cloudでは、KubernetesクラスターとIBM Cloudのサービスの接続を容易にするためのコマンドも用意されています。

```bash
ibmcloud ks cluster-service-bind mycluster default visual-recognition-xx
```

これを実行すると、`binding-<サービスインスタンス名>`という名前のsecretが生成されます。
ただし実行前にCFの組織とスペースを`ibmcloud target`で指定する必要があります。
詳しくはリンクを参照してください。


この場合`jpetstore-watson-nodeport.yaml`を変更する必要あり。
```yaml
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: mmssearch
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: mmssearch
      annotations:
        sidecar.istio.io/inject: "true"
    spec:
      containers:
      - name: mmssearch
        image: kissyyy/mmssearch
        ports:
        - containerPort: 8080
        env:
        - name: DB_LOCATION
          value: "tcp(db)/jpetstore"
        volumeMounts:
         - name: binding #変更
           mountPath: "/etc/secrets"
           readOnly: true
      volumes:
      - name: binding #変更
        secret:
          secretName: binding-vr-jpet #変更
          items:
          - key: binding #変更
            path: mms-secrets.json #変更するならmmssearch/main.goの記述を変更する
```


## アプリケーションのデプロイ

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

2. Access the mmssearch app and start uploading images from `pet-images` directory.

   ![](readme_images/webchat.png)

## アプリの変更

mmssearch.go or index.htmlを変更し再度docker buildする。
ビルドはibmclodu cr buildで実行


## お片付け

```bash
# Use "helm delete" to delete the two apps
helm delete jpetstore --purge
helm delete mmssearch --purge

# Delete the secrets stored in our cluster
kubectl delete secret mms-secret

# Remove the container images from the registry
ibmcloud cr image-rm ${MYREGISTRY}/${MYNAMESPACE}/mmssearch
ibmcloud cr image-rm ${MYREGISTRY}/${MYNAMESPACE}/jpetstoreweb
ibmcloud cr image-rm ${MYREGISTRY}/${MYNAMESPACE}/jpetstoredb

# Delete your entire cluster!
ibmcloud cs cluster-rm yourclustername
```

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

