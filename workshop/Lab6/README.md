# Lab6: サンプル・チャートでHelmを理解する

Lab6では、Helmの理解のために、サンプルのチャートを作ってKubernetesにデプロイします。
チャートの構造を理解することで、提供されるチャートをただ使うのではなく、理解した上で利用できるようになります。

## チャートを作るための参考資料
Helmの公式サイトにチャート開発のためのドキュメントがまとめられています。

https://docs.helm.sh/developing_charts/
https://docs.helm.sh/chart_template_guide/
https://docs.helm.sh/chart_best_practices/

## チャートの作成
チャートの雛形を作成してみます。任意の作業ディレクトリで以下のコマンドを実行してください。

   ```bash
   # 任意のディレクトリでhelm createコマンドを
   $helm create mychart
   Creating mychart
   ```
   
できあがるディレクトリの構造は以下の通りです:
   
   ```bash
   mychart/
   ├── Chart.yaml             # チャートの情報を含むyaml
   ├── charts                 # このチャートが依存するチャートを格納するディレクトリー
   ├── templates              # マニフェストのテンプレートを格納するディレクトリー
   │   ├── NOTES.txt          # OPTIONAL: チャートの使用方法を記載したプレーンテキスト
   │   ├── _helpers.tpl       # 
   │   ├── deployment.yaml    # deployment作成用のyaml
   │   ├── ingress.yaml       # Ingress設定用のyaml
   │   └── service.yaml       # サービス作成用のyaml
   └── values.yaml            # このチャートのデフォルト値を記載したyaml
   ```

## deployment.ymlを紐解く
作成されたtemplates/deployment.ymlをみてみましょう。
Go Template言語で環境により異なる値が記載されています

   ```bash
   $cat templates/deployment.yaml 
   apiVersion: apps/v1beta2
   kind: Deployment
   metadata:
     name: {{ template "mychart.fullname" . }}
     labels:
       app: {{ template "mychart.name" . }}
       chart: {{ template "mychart.chart" . }}
       release: {{ .Release.Name }}
       heritage: {{ .Release.Service }}
   spec:
     replicas: {{ .Values.replicaCount }}
     selector:
       matchLabels:
         app: {{ template "mychart.name" . }}
         release: {{ .Release.Name }}
     template:
       metadata:
         labels:
           app: {{ template "mychart.name" . }}
           release: {{ .Release.Name }}
       spec:
         containers:
           - name: {{ .Chart.Name }}
   :
   (以下省略)
   ```

{{ .Values.<変数名> }}となっている部分はvalues.yamlにあるデフォルト値が埋め込まれます。
以下の設定の場合、例えばvalues.yamlにあるreplicaCountという設定項目が上記のdeployment.ymlのレプリカ数を指定する項目(spec.replicas)に反映されます。

   ```
   $cat values.yaml 
   # Default values for mychart.
   # This is a YAML-formatted file.
   # Declare variables to be passed into your templates.

   replicaCount: 1

   image:
     repository: nginx
     tag: stable
     pullPolicy: IfNotPresent

   service:
     type: ClusterIP
     port: 80

   :
   (以下省略)
   ```

## サンプル・チャートを利用する
まずはこのままサンプルを利用してデプロイしてみましょう。「helm install <任意の名前> ＜チャート・ディレクトリー＞」を実行します。

   ```bash
   $helm install --name sample ./mychart
   NAME:   sample
   LAST DEPLOYED: Wed Feb 13 18:40:59 2019
   NAMESPACE: default
   STATUS: DEPLOYED

   RESOURCES:
   ==> v1beta2/Deployment
   NAME            DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
   sample-mychart  1        1        1           0          1s

   ==> v1/Pod(related)
   NAME                            READY  STATUS             RESTARTS  AGE
   sample-mychart-6cc9cb59d-vnm5g  0/1    ContainerCreating  0         1s

   ==> v1/Service
   NAME            TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)  AGE
   sample-mychart  ClusterIP  172.21.153.99  <none>       80/TCP   1s


   NOTES:
   1. Get the application URL by running these commands:
     export POD_NAME=$(kubectl get pods --namespace default -l "app=mychart,release=sample" -o jsonpath="{.items[0].metadata.name}")
     echo "Visit http://127.0.0.1:8080 to use your application"
     kubectl port-forward $POD_NAME 8080:80
   ```

問題なくデプロイができたかは、以下のコマンドで確認します:

   ```bash
   $helm ls
   NAME  	REVISION	UPDATED                 	STATUS  	CHART        	NAMESPACE
   sample	1       	Wed Feb 13 18:40:59 2019	DEPLOYED	mychart-0.1.0	default 
   ```

   ```bash
   $kubectl get po
   NAME                             READY     STATUS    RESTARTS   AGE
   sample-mychart-6cc9cb59d-vnm5g   1/1       Running   0          4m
   ```

実際にアプリケーションにアクセスするために、「kubectl port-forward <Pod名> <任意のポート番号>:80」でポートフォワーディングします。

   ```bash
   $kubectl port-forward sample-mychart-6cc9cb59d-vnm5g 8080:80
   Forwarding from 127.0.0.1:8080 -> 80
   Forwarding from [::1]:8080 -> 80
   ```

この状態で、Webブラウザから「http://localhost:<指定したポート番号>」でアクセスすればサンプルのWebページが表示されます。

## 設定を変更する
では、次にIKSのフリークラスターに合わせ、KubernetesのNodePortで公開できるように、テンプレートを修正してみましょう。

   ```bash
   $cat mychart/templates/service.yaml 
   apiVersion: v1
   kind: Service
   metadata:
     name: {{ template "mychart.fullname" . }}
     labels:
       app: {{ template "mychart.name" . }}
       chart: {{ template "mychart.chart" . }}
       release: {{ .Release.Name }}
       heritage: {{ .Release.Service }}
   spec:
     type: {{ .Values.service.type }}
     ports:
       - port: {{ .Values.service.port }}
         targetPort: http
         protocol: TCP
         name: http
         {{- if .Values.service.nodePort }}
         nodePort: {{ .Values.service.nodePort }}
         {{- end}}
     selector:
       app: {{ template "mychart.name" . }}
       release: {{ .Release.Name }}
   ```

以下の設定を追記しています。

   ```bash
      {{- if and (.Values.service.nodePort) (eq .Values.service.type "NodePort") }}
      nodePort: {{ .Values.service.nodePort }}
      {{- end}}
   ```

変更したらテンプレートの記載が正しいかのチェックを行います。「helm lint <helmチャートのディレクトリ>」を実行します。

   ```bash
   $helm lint ./mychart/
   ==> Linting ./mychart/
   [INFO] Chart.yaml: icon is recommended

   1 chart(s) linted, no failures
   ```
   
次に設定した値を変更していきましょう。
デフォルト値が定義されているvalue.yamlをコピーします。

   ```bash
   $cp -p mychart/values.yaml value-new.yaml
   ```

コピーしたファイル(value-new.yaml)を開き、以下のようにserviceの項目にあるtypeの設定を修正、そしてnodePortの項目を追加します。
   
   ```bash
   ＃ value-new.yamlに設定追加 (serviceの項目にnodePortを追加)
   
   service:
      type: NodePort
      port: 80
      nodePort: 30001
   ```

上記の設定後、helm upgradeコマンドでhelmリリースを更新します。
先ほどと同じように処理が実行されれば問題なく実行できています。

   ```bash
   $helm upgrade -f value-new.yaml sample ./mychart/
   Release "sample" has been upgraded. Happy Helming!
   LAST DEPLOYED: Wed Feb 13 19:51:15 2019
   NAMESPACE: default
   STATUS: DEPLOYED

   RESOURCES:
   ==> v1/Service
   NAME            TYPE      CLUSTER-IP     EXTERNAL-IP  PORT(S)       AGE
   sample-mychart  NodePort  172.21.153.99  <none>       80:30001/TCP  1h

   ==> v1beta2/Deployment
   NAME            DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
   sample-mychart  1        1        1           1          1h

   ==> v1/Pod(related)
   NAME                            READY  STATUS   RESTARTS  AGE
   sample-mychart-6cc9cb59d-vnm5g  1/1    Running  0         1h


   NOTES:
   1. Get the application URL by running these commands:
     export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services sample-mychart)
     export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
     echo http://$NODE_IP:$NODE_PORT
   ```
  
今度は実際にNodePortでアクセスしてみましょう。「ibmcloud ks workers <クラスター名>」を実行し、パブリックIPアドレスを確認します。
確認したあとで「http://<パブリックIPアドレス>:30001」でアクセスすれば、再びサンプルのアプリケーションにアクセスできます。

   ```bash
   $ibmcloud ks workers mycluster
   OK
   ID                         パブリック IP     プライベート IP   マシン・タイプ   状態     状況    ゾーン   バージョン   
   kube-hou02-xxxxxxxxxx-w1   184.xxx.x.xx    10.76.194.59    free             normal   Ready   hou02    1.10.12_1543 
   ```

## お片付け

```bash
# Use "helm delete" to delete the two apps
helm delete sample --purge

# Delete your entire cluster!
ibmcloud cs cluster-rm yourclustername
```
