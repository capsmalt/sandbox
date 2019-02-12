# IKS ハンズオン
IKS (IBM Cloud Kubernetes Service) を使用して以下のことを学びます。

1. IKSの使用方法
1. Kubenretes CLIの操作
1. Kubernetes リソースの操作
1. Kubernetes マニュフェストの操作
1. Helm Chartを使用したアプリケーションデプロイ
1. クラウドサービスとの連携

# ハンズオン構成 
本ハンズオンは，xxつのLabから成ります。
Lab1-3では，Kubernetesの基礎を学びます。具体的には，Kubernetesの泥臭い作業(kubectl run xxx)やマニュフェストファイル(xxx.yaml)を使用した宣言的な定義などを経験します。
Lab4以降では，Kubernetesのパッケージング技術の1つである **Helm** を使用した一括デプロイや，PaaSサービスの1つである IBM Cloudの画像認識サービス(Visual Recognition)と連携させます。

1. [Lab 1](Lab1): xxx (kubectl run 操作)
1. [Lab 2](Lab2): xxx (ロールバックなどのk8s機能操作)
1. [Lab 3](Lab3): xxx (yaml編集でマニュフェストファイルによるデプロイ)
1. Lab4: xxx (JPetstoreアプリをHelmデプロイ，Helm編集，再デプロイ)
1. Lab5: xxx (JPetstoreアプリの編集。Visual RecognitionのAPI KeyをGoアプリで使用するjsonに書き込み)
1. Lab6: xxx (オプション。Opentoolchainで自動デプロイ)
1. xxxx


# 前提

|名称|タイプ|備考|
|:--|:--|:--|
|IKS|K8sサービス|無料のフリークラスターを作成できること|
|GitHub|ソースコードリポジトリ|Publicアクセス可能なリポジトリ|
|エディタ|IDE|任意のIDE (VS Code, Atom, etd.)|
