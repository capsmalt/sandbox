# Lab 1. Kubernetesクラスターへのアプリケーションデプロイ

Kubernetesクラスターへのアプリケーション・デプロイの方法を学びます。

## 1. K8sクラスターへのアプリケーションデプロイ

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

