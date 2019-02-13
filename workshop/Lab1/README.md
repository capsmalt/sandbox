# Lab 1. セットアップ & Kubernetesクラスターへのアプリケーションデプロイ

Kubernetesクラスター (IBM Cloud Kubernetes Service)へのアプリケーション・デプロイの方法を学びます。

# 0. 前提となるCLIのインストールと，K8sクラスターの構成

1. CLIのインストール **(※事前準備にて実施済のため不要)**

    ["IBM Cloud Developer Tools のインストール方法"](https://console.bluemix.net/docs/cli/index.html#overview) に従い，ご利用されているOSに合わせたコマンドを実行してください。

2. K8sクラスターの作成 **(※セミナー会場にて実施済のため不要)**

    ibmcloudコマンドで作成する場合は， `$ ibmcloud cs cluster-create --name <name-of-cluster>` コマンドを実行します。

3. CLI で IBM Cloudにログイン

    `$ ibmcloud login` (Windowsの方は，PowerShellではなく**コマンドプロンプト**をご利用ください)
    
    以下をインタラクティブに入力します。
    
    - APIエンドポイントのリージョン: `us-south`
    - IBM ID: `自身のIBM Cloudアカウント(Emailアドレスの場合が多いです)`
    - パスワード: `パスワード`

    実行例:
    
    ```bash.sh
    $ ibmcloud login

    Select an API endpoint:
    1. us-south - https://api.ng.bluemix.net
    2. eu-de - https://api.eu-de.bluemix.net
    3. eu-gb - https://api.eu-gb.bluemix.net
    4. au-syd - https://api.au-syd.bluemix.net
    5. us-east - https://api.us-east.bluemix.net
    6. Enter a different API endpoint
    Enter a number> 1

    Email> kzsaitoiks@gmail.com

    Password>
    Authenticating...
    OK

    Targeted account Kazufumi Saito's Account (XXXXXXXXXXXXXXXX)

    Targeted resource group Default


    API endpoint:      https://api.ng.bluemix.net
    Region:            us-south
    User:              kzsaitoiks@gmail.com
    Account:           Kazufumi Saito's Account (XXXXXXXXXXXXXXXX)
    Resource group:    Default
    CF API endpoint:
    Org:
    Space:
    ```

    >補足:  
    >既に入力済の情報がある場合はスキップされる場合があります。  
    >例えば，APIエンドポイント(リージョン)が`au-syd`に自動設定されている場合があります。後の手順で`us-south`を指定しますのでそのまま進めてください。


4. APIエンドポイント(リージョン)を `us-south` に指定

    本日はダラス(=us-south)上に IKSクラスターを構築しているので，ibmcloud cliが指すリージョンを `us-south` に設定します。
    
    実行例: 
    
    ```bash.sh
    $ ibmcloud cs region-set us-south
    ```
    
    >補足:  
    >`cs`プラグインは`ibmcloud`CLIを拡張させるプラグインです。IBM Cloud Kubernetes Service (IKS)の制御に役立ちます。もちろんK8sクラスター自体の操作・制御は`kubectl (後述)`で行えますが，csプラグインを使用することで，K8sクラスターへの接続情報(後述)や，IBM Cloud (PaaS)上でのIPアドレスの振られ方，ノード構成など情報抽出が可能です。現在はcsプラグインの後継版の`ks`プラグインが使用可能です。

5. 接続情報の取得
   
    `$ ibmcloud cs cluster-config <name-of-cluster>` を実行し，K8sクラスターへの接続情報を取得して，コピーします。
    (この後の手順でペーストします)

    実行例:

    ```bash.sh
    $ ibmcloud cs cluster-config mycluster
    OK
    The configuration for mycluster was downloaded successfully.

    Export environment variables to start using Kubernetes.

    export KUBECONFIG=/Users/capsmalt/.bluemix/plugins/container-service/clusters/mycluster/kube-config-hou02-mycluster.yml
    ```
 
    **export KUBECONFIG=/Users/capsmalt/.bluemix/plugins/container-service/clusters/mycluster/kube-config-hou02-mycluster.yml**
    
    上記をコピーします。
 
    > トラブルシュート:  
    > お使いのCLIやプラグインのインストール状況によっては，「アップデート」「新規インストール」を必要とする場合があります。必要に応じて，ログ出力に従って，以下のようなコマンドを実行してください。  
    > サポートが必要な場合は，お声がけください。
    >
    >```bash.sh
    > 例) プラグインの新規インストールが必要なケース (csプラグインのインストール例)
    > $ ibmcloud plugin install container-service -r Bluemix 
    > 
    > 例) プラグインのアップデートが必要なケース (csプラグインのアップデート例)
    > $ ibmcloud plugin update container-service -r Bluemix
    > ```
 
6. 5.で取得した 自身の`KUBECONFIG` の情報をexport(set)します
    
    4.でコピーしているはずなので，ペーストします。

    実行例:
    
    **(Windowsの場合は，exportではなく "set" を使用します)**

    ```bash.sh
    $ export KUBECONFIG=/Users/capsmalt/.bluemix/plugins/container-service/clusters/mycluster/kube-config-hou02-mycluster.yml
    ```

    >補足:  
    > Kubernetesクラスターを操作する際には，Kubernetesのクライアント用CLI `kubectl` を使用します。その際に **KUBECONFIG(K8sクラスターへの接続情報)** が必要になります。
    > クライアント(自PC)から `kubectl` CLIを実行して，サーバー(kube-apiserverコンポーネント)に通信することで，**直接K8s上のコンテナを制御するのではなく**， kube-apiserverからK8sクラスターを制御しています。kube-apiserverはMasterノード上で動作するコンポーネントです。


6. K8sクラスターに接続できるか確認します
    
    ```bash.sh
    $ kubectl get nodes
    NAME            STATUS   ROLES    AGE   VERSION
    10.76.217.175   Ready    <none>   12h   v1.10.12+IKS
    
    上記のように，K8sクラスターの情報(IPアドレス，STATUS，など)が得られればOKです
    ```

# 1. K8sクラスターへのアプリケーションデプロイ

`guestbook` アプリケーションをK8sクラスターにデプロイします。
DockerHub上に，`ibmcom/guestbook:v1` という名前でビルド済Dockerイメージがアップロード済です。

1. `guestbook`を実行します。

   ```$ kubectl run guestbook --image=ibmcom/guestbook:v1```

   アプリケーションの実行ステータスを確認してみましょう。
   `$ kubectl get pods`

   実行例:

   ```console
   $ kubectl get pods
   NAME                          READY     STATUS              RESTARTS   AGE
   guestbook-59bd679fdc-bxdg7    0/1       ContainerCreating   0          1m
   ```
   少し待つと，実行中を示すSTATUS属性である `Running` に変わります。
   
   ```console
   $ kubectl get pods
   NAME                          READY     STATUS              RESTARTS   AGE
   guestbook-59bd679fdc-bxdg7    1/1       Running             0          1m
   ```
   
   runコマンドの最終結果は，アプリケーションコンテナを含むPodだけではなく，
   これらのPodのライフサイクルを管理するDeploymentリソースです。
 
   
3. ステータスが「実行中」になったら，ワーカーノードのIPを介して外部からアクセスできるようにするために，DeploymentをServiceを使用して公開する必要があります。

   `guestbook` アプリケーションが，3000ポートでLISTENするようにします。
   
   実行例:

   ```console
   $ kubectl expose deployment guestbook --type="NodePort" --port=3000
   service "guestbook" exposed
   ```

4. ワーカー・ノードで使用されているポート番号を調べるために，Service情報を取得します。
   
   実行例:

   ```console
   $ kubectl get service guestbook
   NAME        TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
   guestbook   NodePort   10.10.10.253   <none>        3000:31208/TCP   1m
   ```
   
   上記の出力例の場合，`<nodeport>`は31208です。Podは31208ポートで公開され，3000ポートにフォワードされます。
   31000の範囲のポート番号が自動的に選択され，割り当てられます。受講者ごとにポート番号は異なります。

5. 現在 `guestbook` アプリケーションは，ご自身のK8sクラスター上で動作しており，インターネットに公開されている状態です。
   アクセスするために必要な情報を取得します。

   Container Service内のワーカーノードの外部IPアドレスを次のコマンドで取得します。  
   `$ ibmcloud cs workers <name-of-cluster>` を実行すると，`<public-IP>`の列の値を取得できます。
   
   ```console
   $ ibmcloud cs workers <name-of-cluster>
   OK
   ID                                                 Public IP        Private IP     Machine Type   State    Status   Zone    Version  
   kube-hou02-pa1e3ee39f549640aebea69a444f51fe55-w1   173.193.99.136   10.76.194.30   free           normal   Ready    hou02   1.5.6_1500*
   ```
   
   上記の例では，`<public-IP>` の値は `173.193.99.136` です。
   
6. 4.および5.の手順で取得した，IPアドレスと，ポート番号を使用してアプリケーションにアクセスします。
   ブラウザ上で， `<public-IP>:<nodeport>` のように指定します。今回の例では， `173.193.99.136:31208` です。



おめでとうございます。あなたのアプリケーションをK8sクラスター上にデプロイ完了しました。

次のハンズオンはこちら [Lab2](../Lab2/README.md) です。 
Lab1で作成したK8sコンテンツを削除する場合は，以下のコマンドを実行します。

  1. Deploymentを削除する `$ kubectl delete deployment guestbook`.

  2. Serviceを削除する `$ kubectl delete service guestbook`.

