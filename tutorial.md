# SRE, DevOps ハンズオン

## Google Cloud Platform（GCP）プロジェクトの選択

ハンズオンを行う GCP プロジェクトを選択し、 **Start** をクリックしてください。

<walkthrough-project-setup>
</walkthrough-project-setup>

## ハンズオンの内容

下記の内容をハンズオン形式で学習します。

- 環境準備：10 分

  - gcloud コマンドラインツール設定
  - GCP 機能（API）有効化設定
  - サービスアカウント設定

- [Kubernetes Engine（GKE）](https://cloud.google.com/kubernetes-engine/) を用いたアプリケーションの公開：40 分

  - サンプルアプリケーションのコンテナ化
  - コンテナの [Artifact Registry](https://cloud.google.com/artifact-registry) への登録
  - GKE クラスタの作成
  - コンテナの GKE へのデプロイ、外部公開

- [Cloud Build](https://cloud.google.com/cloud-build/) によるビルド、デプロイの自動化：30 分

  - [Cloud Source Repositories](https://cloud.google.com/source-repositories/) へのリポジトリの作成
  - [Cloud Build トリガー](https://cloud.google.com/cloud-build/docs/running-builds/automate-builds) の作成
  - Git クライアントの設定
  - ソースコードの Push をトリガーにした、アプリケーションのビルド、GKE へのデプロイ

- [Cloud Operations](https://cloud.google.com/products/operations) を活用したマイクロサービスの運用、SRE の体験：50 分

  - [Anthos Service Mesh](https://cloud.google.com/anthos/service-mesh)の導入
  - サンプルアプリケーションのデプロイ
  - 稼働時間チェック、アラートの確認
  - The Four Golden Signal をベースにしたダッシュボードの作成
  - （チャレンジ問題）高負荷環境でのトラブルシューティング

- クリーンアップ：10 分

  - プロジェクトごと削除

- Q & A：20 分

## 環境準備

<walkthrough-tutorial-duration duration=10></walkthrough-tutorial-duration>

最初に、ハンズオンを進めるための環境準備を行います。

下記の設定を進めていきます。

- gcloud コマンドラインツール設定
- GCP 機能（API）有効化設定
- サービスアカウント設定

## gcloud コマンドラインツール

GCP は、CLI、GUI から操作が可能です。ハンズオンでは主に CLI を使い作業を行いますが、GUI で確認する URL も合わせて掲載します。

### gcloud コマンドラインツールとは?

gcloud コマンドライン インターフェースは、GCP でメインとなる CLI ツールです。このツールを使用すると、コマンドラインから、またはスクリプトや他の自動化により、多くの一般的なプラットフォーム タスクを実行できます。

たとえば、gcloud CLI を使用して、以下のようなものを作成、管理できます。

- Google Compute Engine 仮想マシン
- Google Kubernetes Engine クラスタ
- Google Cloud SQL インスタンス

**ヒント**: gcloud コマンドラインツールについての詳細は[こちら](https://cloud.google.com/sdk/gcloud?hl=ja)をご参照ください。

<walkthrough-footnote>次に gcloud CLI をハンズオンで利用するための設定を行います。</walkthrough-footnote>

## gcloud コマンドラインツール設定 - プロジェクト

gcloud コマンドでは操作の対象とするプロジェクトの設定が必要です。

### CLI（gcloud コマンド） から利用する GCP のデフォルトプロジェクトを設定

操作対象のプロジェクトを設定します。

```bash
gcloud config set project {{project-id}}
```

<walkthrough-footnote>CLI（gcloud）で利用するプロジェクトの指定が完了しました。次にハンズオンで利用する機能を有効化します。</walkthrough-footnote>

## GCP 環境設定

GCP では利用したい機能ごとに、有効化を行う必要があります。
ここでは、以降のハンズオンで利用する機能を事前に有効化しておきます。

### ハンズオンで利用する GCP の API を有効化する

```bash
gcloud services enable cloudbuild.googleapis.com sourcerepo.googleapis.com cloudresourcemanager.googleapis.com container.googleapis.com stackdriver.googleapis.com cloudtrace.googleapis.com cloudprofiler.googleapis.com logging.googleapis.com iamcredentials.googleapis.com artifactregistry.googleapis.com
```

**GUI**: [API ライブラリ](https://console.cloud.google.com/apis/library?project={{project-id}})

<walkthrough-footnote>必要な機能が使えるようになりました。次にコマンドラインツールに関する残りの設定を行います。</walkthrough-footnote>

## gcloud コマンドラインツール設定 - リージョン、ゾーン

### デフォルトリージョンを設定

コンピュートリソースを作成するデフォルトのリージョンとして、東京リージョン（asia-northeast1）を指定します。

```bash
gcloud config set compute/region asia-northeast1
```

### デフォルトゾーンを設定

コンピュートリソースを作成するデフォルトのゾーンとして、東京リージョン内の 1 ゾーン（asia-northeast1-c）を指定します。

```bash
gcloud config set compute/zone asia-northeast1-c
```

<walkthrough-footnote>必要な機能が使えるようになりました。次にサービスアカウントの設定を行います。</walkthrough-footnote>

## サービスアカウントの作成、権限設定

アプリケーションから他の GCP サービスを利用する場合、個々のエンドユーザーではなく、専用の Google アカウント（サービスアカウント）を作ることを強く推奨しています。

### ハンズオン向けのサービスアカウントを作成する

`sre-gsa` という名前で、ハンズオン専用のサービスアカウントを作成します。

```bash
gcloud iam service-accounts create sre-gsa --display-name "SRE/DevOps HandsOn Service Account"
```

**ヒント**: サービスアカウントについての詳細は[こちら](https://cloud.google.com/iam/docs/service-accounts)をご参照ください。
**GUI**: [サービスアカウント](https://console.cloud.google.com/iam-admin/serviceaccounts?project={{project-id}})

## サービスアカウントに権限（IAM ロール）を割り当てる

作成したサービスアカウントには GCP リソースの操作権限がついていないため、ここで必要な権限を割り当てます。

下記の権限を割り当てます。

- Cloud Profiler Agent ロール
- Cloud Trace Agent ロール
- Cloud Monitoring Metric Writer ロール
- Cloud Monitoring Metadata Writer ロール
- Cloud Debugger Agent ロール

```bash
gcloud projects add-iam-policy-binding {{project-id}}  --member serviceAccount:sre-gsa@{{project-id}}.iam.gserviceaccount.com --role roles/cloudprofiler.agent
gcloud projects add-iam-policy-binding {{project-id}}  --member serviceAccount:sre-gsa@{{project-id}}.iam.gserviceaccount.com --role roles/cloudtrace.agent
gcloud projects add-iam-policy-binding {{project-id}}  --member serviceAccount:sre-gsa@{{project-id}}.iam.gserviceaccount.com --role roles/monitoring.metricWriter
gcloud projects add-iam-policy-binding {{project-id}}  --member serviceAccount:sre-gsa@{{project-id}}.iam.gserviceaccount.com --role roles/stackdriver.resourceMetadata.writer
gcloud projects add-iam-policy-binding {{project-id}}  --member serviceAccount:sre-gsa@{{project-id}}.iam.gserviceaccount.com --role roles/clouddebugger.agent
```

<walkthrough-footnote>アプリケーションから利用する、サービスアカウントの設定が完了しました。次に GKE を利用したアプリケーションの公開に進みます。</walkthrough-footnote>

## Google Kubernetes Engine を用いたアプリケーションの公開

<walkthrough-tutorial-duration duration=40></walkthrough-tutorial-duration>

コンテナ、Kubernetes を利用したアプリケーションの公開を体験します。

下記の手順で進めていきます。

- サンプルアプリケーションのコンテナ化
- コンテナの [Artifact Registry](https://cloud.google.com/artifact-registry/) への登録
- GKE クラスタの作成、設定
- コンテナの GKE へのデプロイ、外部公開

## サンプルアプリケーションのコンテナ化

### コンテナを作成する

Go 言語で作成されたサンプル Web アプリケーションをコンテナ化します。
ここで作成したコンテナはローカルディスクに保存されます。

```bash
docker build -t asia-northeast1-docker.pkg.dev/{{project-id}}/gcp-getting-started-lab-jp/sre-app:v1 .
```

**ヒント**: `docker build` コマンドを叩くと、Dockerfile が読み込まれ、そこに記載されている手順通りにコンテナが作成されます。

### Cloud Shell 上でコンテナを起動する

上の手順で作成したコンテナを Cloud Shell 上で起動します。

```bash
docker run -d -p 8080:8080 \
--name sre-app \
asia-northeast1-docker.pkg.dev/{{project-id}}/gcp-getting-started-lab-jp/sre-app:v1
```

**ヒント**: Cloud Shell 環境の 8080 ポートを、コンテナの 8080 ポートに紐付け、バックグラウンドで起動します。

<walkthrough-footnote>アプリケーションをコンテナ化し、起動することができました。次に実際にアプリケーションにアクセスしてみます。</walkthrough-footnote>

## 作成したコンテナの動作確認

### CloudShell の機能を利用し、起動したアプリケーションにアクセスする

画面右上にあるアイコン <walkthrough-web-preview-icon></walkthrough-web-preview-icon> をクリックし、"プレビューのポート: 8080"を選択します。
これによりブラウザで新しいタブが開き、Cloud Shell 上で起動しているコンテナにアクセスできます。

正しくアプリケーションにアクセスできると、下記のような画面が表示されます。

![BrowserAccessToFrontend](https://storage.googleapis.com/jp-devops-handson/frontend_normal.png)

<walkthrough-footnote>ローカル環境（Cloud Shell 内）で動いているコンテナにアクセスできました。次に GKE で動かすための準備を進めます。</walkthrough-footnote>

## コンテナのレジストリへの登録

先程作成したコンテナはローカルに保存されているため、他の場所から参照ができません。
他の場所から利用できるようにするために、GCP 上のプライベートなコンテナ置き場（コンテナレジストリ）に登録します。

### Docker リポジトリ（Artifact Registry）の作成

```bash
gcloud artifacts repositories create gcp-getting-started-lab-jp --repository-format=docker \
--location=asia-northeast1 --description="Artifact repository for SRE/DevOps Handson"
```

### Docker に対する認証の設定

```bash
gcloud auth configure-docker asia-northeast1-docker.pkg.dev --quiet
```

### 作成したコンテナをコンテナレジストリ（Artifact Registry）へ登録（プッシュ）する

```bash
docker push asia-northeast1-docker.pkg.dev/{{project-id}}/gcp-getting-started-lab-jp/sre-app:v1
```

**GUI**: [Artifact レジストリ](https://console.cloud.google.com/artifacts/browse/{{project-id}})

<walkthrough-footnote>次にコンテナを動かすための基盤である GKE の準備を進めます。</walkthrough-footnote>

## GKE クラスタの作成、設定

コンテナレジストリに登録したコンテナを動かすための、GKE 環境を準備します。

### GKE クラスタを作成する

```bash
gcloud container clusters create "sre-cluster"  \
--image-type "COS" \
--enable-stackdriver-kubernetes \
--enable-ip-alias \
--release-channel regular \
--num-nodes 4 \
--machine-type e2-custom-4-8192 \
--workload-pool {{project-id}}.svc.id.goog
```

**参考**: クラスタの作成が完了するまでに、最大 10 分程度時間がかかることがあります。

**GUI**: [クラスタ](https://console.cloud.google.com/kubernetes/list?project={{project-id}})

<walkthrough-footnote>クラスタが作成できました。次にクラスタを操作するツールの設定を行います。</walkthrough-footnote>

## GKE クラスタの作成、設定

### GKE クラスタへのアクセス設定を行う

Kubernetes には専用の [CLI ツール（kubectl）](https://kubernetes.io/docs/reference/kubectl/overview/)が用意されています。

認証情報を取得し、作成したクラスタを操作できるようにします。

```bash
gcloud container clusters get-credentials sre-cluster
```

<walkthrough-footnote>これで kubectl コマンドから作成したクラスタを操作できるようになりました。次に作成済みのコンテナをクラスタにデプロイします。</walkthrough-footnote>

## コンテナの GKE へのデプロイ、外部公開 - Workload Identity

今回デプロイするアプリケーションは Logging, Tracing など GCP の機能を利用します。アプリケーションに先の手順で作成した Google サービスアカウントの権限を付与するために [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) を利用します。

### アプリケーションを配置する名前空間を作成する

```bash
kubectl create namespace sre-ns
```

### アプリケーションで利用する Kubernetes サービスアカウントを作成する

```bash
kubectl create serviceaccount --namespace sre-ns sre-ksa
```

### Kubernetes サービスアカウントと Google サービスアカウント間のポリシーバインディングを作成する

```bash
gcloud iam service-accounts add-iam-policy-binding \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:{{project-id}}.svc.id.goog[sre-ns/sre-ksa]" \
  sre-gsa@{{project-id}}.iam.gserviceaccount.com
```

### Kubernetes サービスアカウントにアノテーションを追加する

```bash
kubectl annotate serviceaccount \
  --namespace sre-ns \
  sre-ksa \
  iam.gke.io/gcp-service-account=sre-gsa@{{project-id}}.iam.gserviceaccount.com
```

<walkthrough-footnote>これで GKE 上の sre-ns 名前空間に作成したアプリケーションが sre-gsa サービスアカウントの権限を利用できるようになりました。</walkthrough-footnote>

## コンテナの GKE へのデプロイ、外部公開 - 準備

### ハンズオン用の設定ファイルを修正する

Kubernetes のデプロイ用設定ファイルを、コンテナレジストリに登録済みのコンテナを使うように修正します。

```bash
sed -i".org" -e "s/FIXME/{{project-id}}/g" gke-config/deployment.yaml
```

<walkthrough-footnote>アプリケーションをクラスタにデプロイする準備ができました。次にデプロイを行います。</walkthrough-footnote>

## コンテナの GKE へのデプロイ、外部公開

### コンテナを Kubernetes クラスタへデプロイする

```bash
kubectl apply -f gke-config/
```

このコマンドにより、Kubernetes の 2 リソースが作成され、インターネットからアクセスできるようになります。

- [Deployment](https://cloud.google.com/kubernetes-engine/docs/concepts/deployment)
- [Service](https://kubernetes.io/ja/docs/concepts/services-networking/service/)

**GUI**: [Deployment](https://console.cloud.google.com/kubernetes/workload?project={{project-id}}), [Service/Ingress](https://console.cloud.google.com/kubernetes/discovery?project={{project-id}})

<walkthrough-footnote>コンテナを GKE にデプロイし、外部公開できました。次にデプロイしたアプリケーションにアクセスします。</walkthrough-footnote>

## コンテナの GKE へのデプロイ、外部公開 - 動作確認

### アクセスするグローバル IP アドレスの取得

デプロイしたコンテナへのアクセスを待ち受ける Service の IP アドレスを確認します。

```bash
kubectl get service sre-app-loadbalancer -n sre-ns -w
```

このコマンドは対象のリソース状態を監視（watch）します。グローバル IP アドレスが付与されたら Ctrl + C を押してキャンセルしてください。

**ヒント**: デプロイしたコンテナ自体はグローバルからアクセス可能な IP アドレスを持ちません。今回のように、外部からのアクセスを受け付けるリソース（Service）を作成し、そこを通してコンテナにアクセスする必要があります。

### コンテナへアクセス

下記のコマンドを実行し出力された URL をクリックし、アクセスします。

```bash
export SERVICE_IP=$(kubectl get service sre-app-loadbalancer -n sre-ns -ojsonpath='{.status.loadBalancer.ingress[0].ip}'); echo "http://${SERVICE_IP}/"
```

<walkthrough-footnote>アプリケーションにインターネット経由でアクセスすることができました。次にパイプラインを作り、今まで実施した手順を自動化します。</walkthrough-footnote>

## Cloud Build によるビルド、デプロイの自動化

<walkthrough-tutorial-duration duration=30></walkthrough-tutorial-duration>

Cloud Build を利用し今まで手動で行っていたアプリケーションのビルド、コンテナ化、リポジトリへの登録、GKE へのデプロイを自動化します。

下記の手順で進めていきます。

- [Cloud Source Repositories](https://cloud.google.com/source-repositories/) へのリポジトリの作成
- [Cloud Build トリガー](https://cloud.google.com/cloud-build/docs/running-builds/automate-builds) の作成
- Git クライアントの設定
- ソースコードの Push をトリガーにした、アプリケーションのビルド、GKE へのデプロイ

## Cloud Build サービスアカウントへの権限追加

Cloud Build を実行する際に利用されるサービスアカウントを取得し、環境変数に格納します。

```bash
export CB_SA=$(gcloud projects get-iam-policy {{project-id}} | grep cloudbuild.gserviceaccount.com | uniq | cut -d ':' -f 2)
```

上で取得したサービスアカウントに Cloud Build から自動デプロイをさせるため Kubernetes 管理者の権限を与えます。

```bash
gcloud projects add-iam-policy-binding {{project-id}} --member serviceAccount:$CB_SA --role roles/container.admin
```

<walkthrough-footnote>Cloud Build で利用するサービスアカウントに権限を付与し、Kubernetes に自動デプロイできるようにしました。次に資材を格納する Git リポジトリを作成します。</walkthrough-footnote>

## Cloud Source Repository（CSR）に Git レポジトリを作成

今回利用しているソースコードを配置するためのプライベート Git リポジトリを、Cloud Source Repository（CSR）に作成します。

```bash
gcloud source repos create sre-repo
```

**GUI**: [Source Repository](https://source.cloud.google.com/{{project-id}}/sre): 作成前にアクセスすると拒否されます。

<walkthrough-footnote>資材を格納する Git リポジトリを作成しました。次にこのリポジトリに更新があったときにそれを検知し、処理を開始するトリガーを作成します。</walkthrough-footnote>

## Cloud Build トリガーを作成

Cloud Build に前の手順で作成した、プライベート Git リポジトリに push が行われたときに起動されるトリガーを作成します。

```bash
gcloud beta builds triggers create cloud-source-repositories --description="sre-build-trigger" --repo=sre-repo --branch-pattern=".*" --build-config="cloudbuild.yaml"
```

**GUI**: [ビルドトリガー](https://console.cloud.google.com/cloud-build/triggers?project={{project-id}})

<walkthrough-footnote>リポジトリの更新を検知するトリガーを作成しました。次にリポジトリを操作する Git クライアントの設定を行います。</walkthrough-footnote>

## Git クライアント設定

### 認証設定

Git クライアントで CSR と認証するための設定を行います。

```bash
git config --global credential.https://source.developers.google.com.helper gcloud.sh
```

**ヒント**: git コマンドと gcloud で利用している IAM アカウントを紐付けるための設定です。

### 利用者設定

USERNAME を自身のユーザ名に置き換えて実行し、利用者を設定します。

```bash
git config --global user.name "USERNAME"
```

### メールアドレス設定

USERNAME@EXAMPLE.com を自身のメールアドレスに置き換えて実行し、利用者のメールアドレスを設定します。

```bash
git config --global user.email "USERNAME@EXAMPLE.com"
```

<walkthrough-footnote>Git クライアントの設定を行いました。次に先程作成した CSR のリポジトリと、Cloud Shell 上にある資材を紐付けます。</walkthrough-footnote>

## Git リポジトリ設定

CSR を Git のリモートレポジトリとして登録します。
これで git コマンドを使い Cloud Shell 上にあるファイル群を管理することができます。

```bash
git remote add google https://source.developers.google.com/p/{{project-id}}/r/sre-repo
```

<walkthrough-footnote>以前の手順で作成した CSR のリポジトリと、Cloud Shell 上にある資材を紐付けました。次にその資材をプッシュします。</walkthrough-footnote>

## CSR への資材の転送（プッシュ）

以前の手順で作成した CSR は空の状態です。
git push コマンドを使い、CSR に資材を転送（プッシュ）します。

```bash
git push google master
```

**GUI**: [Source Repository](https://source.cloud.google.com/{{project-id}}/sre-repo) から資材がプッシュされたことを確認できます。

<walkthrough-footnote>Cloud Shell 上にある資材を CSR のリポジトリにプッシュしました。次に資材の更新をトリガーに処理が始まっている Cloud Build を確認します。</walkthrough-footnote>

## Cloud Build トリガーの動作確認

### Cloud Build の自動実行を確認

[Cloud Build の履歴](https://console.cloud.google.com/cloud-build/builds?project={{project-id}}) にアクセスし、git push コマンドを実行した時間にビルドが実行されていることを確認します。

### 新しいコンテナのデプロイ確認

ビルドが正常に完了後、以下コマンドを実行し、Cloud Build で作成したコンテナがデプロイされていることを確認します。

```bash
kubectl describe deployment/sre-app-deployment -n sre-ns | grep Image
```

`error: You must be logged in to the server (Unauthorized)` というメッセージが出た場合は、再度コマンドを実行してみてください。

コマンド実行結果の例。

```
    Image:        asia-northeast1-docker.pkg.dev/{{project-id}}/gcp-getting-started-lab-jp/sre:COMMITHASH
```

Cloud Build 実行前は Image が `asia-northeast1-docker.pkg.dev/{{project-id}}/gcp-getting-started-lab-jp/sre-app:v1` となっていますが、実行後は `asia-northeast1-docker.pkg.dev/{{project-id}}/gcp-getting-started-lab-jp/sre-app:COMMITHASH` になっている事が分かります。
実際は、COMMITHASH には Git のコミットハッシュ値が入ります。

<walkthrough-footnote>資材を更新、プッシュをトリガーとしたアプリケーションのビルド、コンテナ化、GKE へのデプロイを行うパイプラインが完成しました。</walkthrough-footnote>

## 既存リソースの削除

次のセクションに備えて、ここまでで GKE 上に作成したアプリケーションを削除します。

```bash
kubectl delete -f gke-config/
```

## Cloud Operations を活用したマイクロサービスの運用、SRE の体験

<walkthrough-tutorial-duration duration=50></walkthrough-tutorial-duration>

Cloud Operations を使い、マイクロサービスアプリケーションの運用、トラブルシューティングを体験します。

下記の手順で進めていきます。

- [Anthos Service Mesh](https://cloud.google.com/anthos/service-mesh)の導入
- サンプルアプリケーションのデプロイ
- 稼働時間チェック、アラートの確認
- The Four Golden Signal をベースにしたダッシュボードの作成
- （チャレンジ問題）高負荷環境でのトラブルシューティング

## サンプルアプリケーション

ここからは Cloud Operations を使い、Google Cloud でのアプリケーション運用を体験します。まず、サンプルアプリケーションをデプロイします。

### サンプルアプリケーションの画面

今回利用するサンプルアプリケーションは EC サイトを模したアプリケーションです。下記のようなページが表示されます。

![Home Page](https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/master/docs/img/online-boutique-frontend-1.png)
![Checkout ページ](https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/master/docs/img/online-boutique-frontend-2.png)

## アプリケーション構成

サンプルアプリケーションはマイクロサービス アーキテクチャで構成されており、それぞれのサービスは下記のような構成となっています。

![アーキテクチャ図](https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/master/docs/img/architecture-diagram.png)

詳細は[こちら](https://github.com/GoogleCloudPlatform/microservices-demo)をご覧ください

## Anthos Service Mesh の導入

サンプルアプリケーションから有用なメトリクスを取得するために、[Anthos Service Mesh](https://cloud.google.com/anthos/service-mesh) を導入します。

### Anthos Service Mesh のインストール

```bash
curl https://storage.googleapis.com/csm-artifacts/asm/install_asm_1.8 > install_asm && chmod a+x install_asm
mkdir asm_output && ./install_asm \
  --project_id {{project-id}} \
  --cluster_name sre-cluster \
  --cluster_location asia-northeast1-c \
  --mode install \
  --output_dir asm_output \
  --enable_all
```

`Successfully installed ASM.` というメッセージが出たらインストール成功です。

### Anthos Service Mesh の初期設定

サービスメッシュの対象とする名前空間にサイドカーを入れるための設定を行います。

```bash
kubectl label namespace sre-ns istio-injection- istio.io/rev=asm-183-2 --overwrite
```

## サンプルアプリケーション、Load Generator のデプロイ

### サンプルアプリケーションのデプロイ

```bash
kubectl apply -f k8s-manifests/kubernetes-manifests.yaml -f k8s-manifests/istio-manifests.yaml -n sre-ns
```

### 外部アクセス IP アドレスの取得、変数への設定

```bash
export INGRESSGW_ADDR=$(kubectl get svc istio-ingressgateway -n istio-system -ojsonpath='{.status.loadBalancer.ingress[0].ip}')
kubectl create configmap address-config --from-literal=FRONTEND_ADDR=http://$INGRESSGW_ADDR
```

### Load generator のデプロイ

```bash
kubectl apply -f k8s-manifests/loadgen.yaml
```

## サンプルアプリケーション、Load Generator の動作確認

それぞれ下記の URL にアクセスし、ページが見えることを確認します。

### サンプルアプリケーション

```bash
echo http://$INGRESSGW_ADDR
```

前半のハンズオンで実施した画面が表示された場合は、ブラウザをリロードしてください。

### Load Generator

```bash
export LOADGEN_ADDR=$(kubectl get svc loadgenerator -n default -ojsonpath='{.status.loadBalancer.ingress[0].ip}')
echo http://$LOADGEN_ADDR
```

## Cloud Monitoring の有効化

[こちら](https://console.cloud.google.com/monitoring?project={{project-id}})にアクセスし、Cloud Monitoring が有効化が完了するまで 1 分程度待ちます。

## 稼働時間チェック

## 稼働時間チェックの設定

先程稼働させたサンプルアプリケーション（トップページ）の死活監視設定を行います。ここでは、HTTP でアクセスをし、200 以外が返ってきた場合、自分のメールアドレスにアラートを送る設定を行います。

[こちら](https://console.cloud.google.com/monitoring/uptime?project={{project-id}})にアクセスし、`稼働時間チェックの作成` をクリックします。

1. タイトルに `EC top page` と入力し `次へ` をクリックします。
1. `Hostname` にサンプルアプリケーションの IP アドレスを入力し、`次へ` をクリックします（それ以外の項目はデフォルトで大丈夫です）
1. そのまま `次へ` をクリックします。
1. `Duration` を `most recent value` に変更します。`Notification Channels`、`通知チャンネルを管理` の順にクリックします。
1. 新しく開いた画面で Email の `ADD NEW` をクリックし、自分のメールアドレス、表示名を入力し、`SAVE` をクリックします。その後、こちらのタブは閉じます。
1. 元の画面に戻り、`通知チャンネルを管理` の左のリフレッシュボタンを押し、先程追加した Email 項目にチェックを入れ、`OK` を押します。
1. `TEST` をクリックし、200 が返ってくることを確認した後に、`CREATE` をクリックします。

## 稼働時間チェック正常動作確認

作成した稼働時間チェックの各地域情報が緑のチェックになるまで、画面をリロードしながら数分待ちます。

###  サンプルアプリケーションのトップページを停止

```bash
kubectl delete deploy frontend -n sre-ns
```

少し待ち、先ほど設定したメールアドレスにアラートメールが来ることを確認します。また Cloud Monitoring の[概要ページ](https://console.cloud.google.com/monitoring?project={{project-id}})にもインシデントがでていることを確認します。

### サンプルアプリケーションの再デプロイ

```bash
kubectl apply -f k8s-manifests/kubernetes-manifests.yaml -n sre-ns
```

少し待つと、問題が解決したことを知らせるメールがきます。

## Load Generator の起動

Load Generator からサンプルアプリケーションに負荷をかけます。

### Web UI へアクセス

```bash
echo http://$LOADGEN_ADDR
```

`Number of total users to simulate` に `50`、`Spawn rate` に `1` を入力し、`Start swarming` をクリックし、負荷をかけ始めます。利用者のアクセス遷移を模したアクセスが事前に組み込まれています。

## ダッシュボードの作成

Google が提唱する SRE の考え方では、監視項目はサービス目線で設定し、できる限りすべてのサービスで揃えることを推奨しています。その様にすることで、事前知識の無いサービスの運用ををあるエンジニアが任されたとしても問題なくトラブルシューティングができる体制を目指しています。

今回はその中で提唱されている、どのサービスでも監視とされる[The Four Golden Signals](https://sre.google/sre-book/monitoring-distributed-systems/)（Saturation は除きます）についてダッシュボードを作成します。

- Latency
- Errors
- Traffic
- Saturation

[ダッシュボード](https://console.cloud.google.com/monitoring/dashboards?project={{project-id}}) に移動し、`CREATE DASHBOARD` をクリックして作成準備をしておきます。ダッシュボードの名前は `Sample App Overview` としておきます。

## Latency

ここではマイクロサービスごとの Latency を 95% tile で表示するグラフを作成します。

1. `グラフライブラリ` から Line を選びます
1. `ADVANCED` を選択します
1. `Resource type` で `Istio Canonical Service` を選びます
1. `Metric` で `Server Response Latencies` を選びます
1. `Group by` で `destination_service_name` を選びます
1. `Group by function` で `95th percentile` を選びます

## Errors

ここではマイクロサービスごとのエラー発生数を表示するグラフを作成します。

1. `グラフライブラリ` から Line を選びます
1. `ADVANCED` を選択します
1. `Resource type` で `Istio Canonical Service` を選びます
1. `Metric` で `Server Request Count` を選びます
1. `ADD FILTER` をクリックします
1. `Label` に `response_code` を選びます
1. `Value` に `500` を選び、`Done` をクリックします（500 が選択できない場合は、鉛筆マークをクリックし、自分で入力します）
1. `Group by` で `destination_service_name` を選びます
1. `Group by function` で `sum` を選びます

## Traffic

ここではマイクロサービスごとのアクセス数を表示するグラフを作成します。

1. `グラフライブラリ` から Line を選びます
1. `ADVANCED` を選択します
1. `Resource type` で `Istio Canonical Service` を選びます
1. `Metric` で `Server Response Count` を選びます
1. `Group by` で `destination_service_name` を選びます
1. `Group by function` で `sum` を選びます

## CPU utilization

さらにマイクロサービスごとにコンテナが利用している CPU の状況を表示するグラフを作成します。

1. `グラフライブラリ` から Line を選びます
1. `ADVANCED` を選択します
1. `Resource type` で `Kubernetes Container` を選びます
1. `Metric` で `CPU limit utilization` を選びます
1. `Group by` で `service_name` を選びます
1. `Group by function` で `max` を選びます

## Memory utilization

さらにマイクロサービスごとにコンテナが利用している Memory の状況を表示するグラフを作成します。

1. `グラフライブラリ` から Line を選びます
1. `ADVANCED` を選択します
1. `Resource type` で `Kubernetes Container` を選びます
1. `Metric` で `Memory limit utilization` を選びます
1. `Group by` で `service_name` を選びます
1. `Group by function` で `max` を選びます

これで合計 5 つのグラフが作成できました。見やすいようにサイズ、配置を調整してください。

## (チャレンジ問題) 高負荷環境でのトラブルシューティング

作成したダッシュボードから、現在の負荷状況ではキャパシティに問題が無いことを確認します。

### 負荷を上げる

Load Generator の画面から同時アクセス数を `500`、`Spawn rate` を `5` に変更します。その後、Load Generator、ダッシュボードの状況を確認し、システムに問題が起きていないかを確認します。

### トラブルシューティング

Latency が高い、もしくはエラーが多く発生している場合、現在の環境を修正して問題を解決してみましょう。

- 目標 1: マニュアルで対応し Latency を下げ、エラーも低減するようにする
- 目標 2: 自動でシステムがスケールするようにする

**ヒント**:

- 作成したダッシュボードを参考にしましょう
- [こちら](https://cloud.google.com/kubernetes-engine/docs/how-to/scaling-apps)を参考にしましょう

## Congraturations!

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

これにて GKE、Cloud Build を使ったデプロイパイプラインの作成、Operations を用いた運用、そしてアプリケーションのトラブルシューティングを体験するハンズオンは完了です！！

デモで使った資材が不要な方は、次の手順でクリーンアップを行って下さい。

## クリーンアップ（プロジェクトを削除）

ハンズオン用に利用したプロジェクトを削除し、コストがかからないようにします。

### GCP のデフォルトプロジェクト設定の削除

```bash
gcloud config unset project
```

### プロジェクトの削除

```bash
gcloud projects delete {{project-id}}
```

### ハンズオン資材の削除

```bash
cd $HOME && rm -rf cloudshell_open
```
