# VPCの作成

- 大変なのでCfnで一発

https://ap-northeast-1.console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/quickcreate?templateURL=https%3A%2F%2Fs3.ap-northeast-1.amazonaws.com%2Fcf-templates-nxbtnduaoqyw0-ap-northeast-1%2Ftemplate-1740722027707.yaml&stackName=jawsugniigata21

# RDSの作成

- サブネットグループの作成
  - WPVPC
  - RDSSubnetの2つを追加

- データベースの作成
  - 標準作成
  - mysql
  - 8.0.40
  - 無料利用枠
  - 単一のDBインスタンス
  - 認証情報管理: セルフマネージド

  - パスワード自動生成
  - バースト可能クラス: t4gmicro
  - gp2
  - 20GB
  - EC2に接続しない
  - VPC: WPVPC
  - サブネットグループ: 先に作ったやつ
  - パブリックアクセス: なし
  - セキュリティグループ: RDSSecuritygroup
  - データベース認証: パスワード認証
  - 追加設定
    - 最初のデータベース名: wordpress
    - 自動バックアップ: 無効
  - 上から出てくるやつから認証情報の表示からパスワードをコピ
    - 最悪ミスったら後で修正は可能
- 詳細を開いて、エンドポイントをコピー

# ロールの作成
- コードデプロイのロール
  - AWSサービス
  - ユースケース: CodeDeployECS
  - ポリシー: AWSCodeDeployRoleForECS(勝手になる)

# CodeDeploy
 - アプリケーションの作成
 - AmazonECS
 (あれ、いらなかったかもしれない...?)

# EFSの作成
 - カスタマイズ
 - リージョン
 - 自動バックアップ有効化を外す
 - ネットワーク

# ecsの作成
クラスタの作成

タスク定義の作成
  - 一応メモリを2GBに
  - タスクロールの作成(不要)
  - タスク実行ロールの作成(新しいロールの作成)
  - コンテナ
    `public.ecr.aws/docker/library/wordpress:6.6.2-php8.2-apache`
  - 環境変数
     ```sh
      WORDPRESS_DB_HOST: <rdsのDBエンドポイント>
      WORDPRESS_DB_USER: admin
      WORDPRESS_DB_PASSWORD: <RDS作成時にコピーした認証情報>
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_AUTH_KEY: <リンク参照>
      WORDPRESS_SECURE_AUTH_KEY: <リンク参照>
      WORDPRESS_LOGGED_IN_KEY: <リンク参照>
      WORDPRESS_NONCE_KEY: <リンク参照>
      WORDPRESS_AUTH_SALT: <リンク参照>
      WORDPRESS_SECURE_AUTH_SALT: <リンク参照>
      WORDPRESS_LOGGED_IN_SALT: <リンク参照>
      WORDPRESS_NONCE_SALT: <リンク参照>
      https://api.wordpress.org/secret-key/1.1/salt/
      ```
  - ストレージ
    - ボリューム
      - ファイルタイプ: EFS
      - ファイルシステムID: 作ったやつ
    - マウントポイント
      `/var/www/html/wp-content`


サービスの作成
  - 起動タイプ: fargate
  - アプリケーションタイプ: サービス
  - タスク定義: ファミリー: 作ったやつ
  - サービス名: 任意(jawsugniigata21)
  - 必要なタスク
    - 一旦0(後で変える)

  - デプロイオプション
    - ブルー/グリーン
    - サービスロール: 作ったやつ

  - ネットワーキング
    - vpc: MyVPC
    - FargateSubnet
    - セキュリティグループ: fargatesg
    - public ip不要

  - ロードバランサ
    - ロードバランシングを使用
    - 新しいロードバランサALB
    - ヘルスチェックパス: `/wp-includes/images/blank.gif`
    - 時間をいじる

  - ロードバランサを修正
    - ネットワークマッピング->サブネットの編集->ALBように変更
    - セキュリティ->編集->ALB用に変更

  - 起動
    - 必要なタスク2

# 起動確認
ALBから見てみる
ALBの詳細にDNS名があるので開く

軽くいじる
- メディファイルをテキトーに入れてみる
  - EFSの動作確認のため



バージョンが6.7.2なのを確認
# バージョンを上げる
- タスク定義を更新する
  - 新しいリビジョンの作成
    - `public.ecr.aws/docker/library/wordpress:6.7.2-php8.2-apache`
- サービスを更新する
  - 新しいデプロイ強制
  - タスク定義のリビジョンを変更する
  - デプロイオプションでいい感じに選ぶ


  - codedeploy見ると前にやつ生きてるのがわかる
    - ロールバックすれば多分戻る
    - 戻ることを確認

    ※実際はDBのマイグレートなどが走るので、そこを意識していないと壊れる

# プロダクションとして使うには
- ドメインを取る
- CloudFrontを挟む
  - wp-admin以外にキャッシュを効かせる
- ALBにドメインを当てる
- ALBをhttpsにする
- rdsにちゃんとユーザーを作る
- 環境変数はsecrets managerに置く

# 片付け
*がついてるやつは消し忘れるとやばめ

- *ECSサービス
- ECSクラスタ
- ECSタスク定義
  -  普通には消せない
  - リビジョンを全部選択して登録解除
  - その後削除
- *ALB
- ターゲットグループ
- *RDS
  - スナップショット(あれば)
- サブネットグループ
- *EFS
- CloudFormation
  - 作成されたリソースも消える
- iam
  - ロール
    - code_deployのやつ
    - ecsTaskExecutionRole
    - rds-monitoring-role(あれば)
- CloudWatchLogs
  - ecsのやつ
