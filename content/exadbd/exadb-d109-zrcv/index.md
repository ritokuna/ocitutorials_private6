---
title: "109 : ExaDB-DでAutonomous Recovery Service (RCV/ZRCV) をセットアップしよう"
excerpt: "自動バックアップの取得先としてAutonomous Recovery Service (RCV/ZRCV) をExadata Database Service on Dedicated Infrastructure (ExaDB-D)に設定する方法を紹介します。"
order: "2_109"
layout: single
header:
  teaser: "/exadbd/exadb-d109-zrcv/ExaDB-D_zrcv01.png"
  overlay_image: "/exadbd/exadb-d109-zrcv/ExaDB-D_zrcv01.png"
  overlay_filter: rgba(34, 66, 55, 0.7)
#link: https://apexapps.oracle.com/pls/apex/dbpm/r/livelabs/view-workshop?wid=797
---

<a id="anchor0"></a>

# はじめに

Oracle Database Autonomous Recovery Service（以下、リカバリ・サービス）は、Oracle Cloud Infrastructure (OCI) で実行する Oracle Database 向けのフル・マネージド型データ保護サービスです。  
オンプレミス製品の Zero Data Loss Recovery Appliance (RA) のデータ保護技術をベースとしながら、クラウドならではの自動化機能も兼ね備えています。リカバリ・サービスは Oracle Database の変更をリアルタイムで保護し、本番データベースのオーバーヘッドなしでバックアップを検証するほか、任意の時点への高速で予測可能なリカバリを実現します。

このチュートリアルでは、ExaDB-D にリカバリ・サービスを設定する手順についてご紹介します。  
このチュートリアルを完了すると、以下のような構成図となります。

![img](ExaDB-D_zrcv01.png)

**前提条件 :**

- [101：ExaDB-D を使おう](../exadb-d101-create-exadb-d){:target="\_blank"} を通じて Oracle Database の作成が完了していること

- Autonomous Recovery Service(RCV)を利用する場合、Oracle Database 19.16 以上

- Zero Data Loss Autonomous Recovery Service (ZRCV) を利用する場合、Oracle Database 19.18 以上

<br>

**注意** チュートリアル内の画面ショットについては現在の画面と異なっている場合があります。

<br>

**目次**

- [1. サービス制限の引き上げリクエスト](#1-サービス制限の引き上げリクエスト)
- [2. IAM ポリシーを設定しよう](#2-iam-ポリシーを設定しよう)
- [3. サブネットおよびセキュリティ・ルールを設定しよう](#3-サブネットおよびセキュリティ・ルールを設定しよう)
- [4. リカバリ・サービス・サブネットを登録しよう](#4-リカバリ・サービス・サブネットを登録しよう)
- [5. 保護ポリシーを作成しよう](#5-保護ポリシーを作成しよう)
- [6. ExaDB-D でリカバリ・サービスを有効化しよう](#6-ExaDB-Dでリカバリ・サービスを有効化しよう)
- [7. 護されたデータベースの詳細を確認しよう](#7-保護されたデータベースの詳細を確認しよう)
- [8. 保護されたデータベースの一覧を確認しよう](#8-保護されたデータベースの一覧を確認しよう)

<br>
**所要時間 :** 約40分
<br>

# 1. サービス制限の引き上げリクエスト

**サービス制限について**  
Autonomous Recovery Service (RCV/ZRCV) は全リージョンにデフォルトで以下のサービス制限が付与されています。
これらの容量以上にリソースが必要な場合は[1. サービス制限設定](#1-サービス制限の引き上げリクエスト)を実行してください。現在のサービス制限および使用状況情報を確認し、必要に応じてリソース制限の引上げをリクエストしましょう。
{: .notice--info}

**デフォルト付与されるサービス制限**

| 説明                                | 制限名                               | サービス制限 | 内容                                         |
| ----------------------------------- | ------------------------------------ | ------------ | -------------------------------------------- |
| Space Used for Recovery Window (GB) | protected-database-backup-storage-gb | 10,240       | データベースの回復ウィンドウに必要な GB 容量 |
| Protected Database Count            | protected-database-count             | 10           | 保護されたデータベース数                     |

<br>

**サービス制限設定**  
リカバリ・サービスのサービス制限の引き上げリクエストを申請します。

まず、OCI コンソールのナビゲーション・メニューから[ガバンスと管理]をクリックし、[制限、割当ておよび使用状況]をクリックします。

![img](ExaDB-D_zrcv02.png)

次に、「制限、割当ておよび使用状況」の「サービス制限の引き上げリクエスト」をクリックします。
![img](ExaDB-D_zrcv03.png)
<br>

クリックすると、サービス制限の更新リクエストを申請するフォームに遷移します。
以下の項目を選択し、必要な数の制限を入力します。

- Service Category：Autonomous Recovery Service
- Resource：Protected Database Count
- Resource：Space Used for Recovery Window (GB)

入力が完了したら「サポート・リクエストの作成」からサービス制限の引き上げリクエストを申請しましょう。

![img](ExaDB-D_zrcv04.png)

<br>

# 2. IAM ポリシーを設定しよう

次に、リカバリ・サービスおよび関連リソースへのアクセスを有効にするポリシーを設定します。

この設定をすると、サポートされている OCI データベース・サービスでデータ保護にリカバリ・サービスを使用できるようになります。

**リカバリ・サービスの使用に必要なポリシー・ステートメント**  
以下のポリシーを設定してきます。

| No. | ポリシー・ステートメント                                                           | 作成場所         | 説明                                                                                                                                                                                               |
| --- | ---------------------------------------------------------------------------------- | ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Allow group <グループ名> to manage all-resources in compartment <コンパートメント> |                  | コンパートメント管理者ポリシー                                                                                                                                                                     |
| 2   | Allow service database to manage recovery-service-family in tenancy                | Root Compartment | リカバリ・サービス作成に必要なポリシー。OCI Database サービスは、コンパートメント内の保護されたデータベース、保護ポリシーおよびリカバリ・サービス・サブネットにアクセスできます。                  |
| 3   | Allow service database to manage tag-namespaces in tenancy                         | Root Compartment | リカバリ・サービス作成に必要なポリシー。OCI Database サービスでテナンシのタグ・ネームスペースにアクセスできます。                                                                                  |
| 4   | Allow service rcs to manage recovery-service-family in tenancy                     | Root Compartment | リカバリ・サービス作成に必要なポリシー。リカバリ・サービスが、コンパートメント内の保護されたデータベース、リカバリ・サービス・サブネットおよび保護ポリシーにアクセスして管理できるようにします。　 |
| 5   | Allow service rcs to manage virtual-network-family in tenancy                      | Root Compartment | リカバリ・サービス作成に必要なポリシー。リカバリ・サービスがコンパートメント内の各データベース VCN 内のプライベート・サブネットにアクセスして管理できるようにします。                              |
| 6   | Allow group <グループ名> to manage recovery-service-family in tenancy              | Root Compartment | 指定したグループのユーザーがすべてのリカバリ・サービス・リソースにアクセスできるようにします。                                                                                                     |

<br>

**ポリシー画面**

「ナビゲーション・メニュー」の[アイデンティティとセキュリティ]を選択し、[アイデンティティ]の[ポリシー]をクリックします。

![img](ExaDB-D_zrcv05.png)

「ポリシーの作成」ボタンをクリックし、ポリシーの作成をします。

- **名前**：任意（スペースは使用できません。英字、数字、ハイフン、ピリオドまたはアンダースコアのみです。）
- **説明**：任意
- **ポリシー**・ビルダー：「手動エディタの表示」を有効化し、以下のポリシーを入力します。

```sh
Allow group <グループ名> to manage all-resources in compartment <コンパートメント>
Allow service database to manage recovery-service-family in tenancy
Allow service database to manage tag-namespaces in tenancy
Allow service rcs to manage recovery-service-family in tenancy
Allow service rcs to manage virtual-network-family in tenancy
Allow group <グループ名> to manage recovery-service-family in tenancy
```

リカバリ・サービスの使用に必要なポリシー・ステートメントは**Root Compartment**へ作成する必要があることに注意してください。
{: .notice--warning}

![img](ExaDB-D_zrcv06.png)

<br>

# 3. サブネットおよびセキュリティ・ルールを設定しよう

リカバリ・サービスはリカバリ・サービス・サブネットという専用のサブネットを使用します。新規、既存、どちらのサブネットでもリカバリ・サービス・サブネットとして利用できます。プライベート・サブネットをリカバリ・サービス・サブネットとして利用することを推奨としています。

まず、データベースとリカバリ・サービス間のバックアップ・トラフィックを許可するために、データベースの VCN に以下のイングレス・ルールを追加します。
<br>

**リカバリ・サービスで使用されるプライベート・サブネットのサブネット・サイズ要件およびイングレス・ルール**

| イングレス・ルール                               | ソース CIDR                        | IP プロトコル | ソース・ポート範囲 | 宛先ポート範囲 |
| ------------------------------------------------ | ---------------------------------- | ------------- | ------------------ | -------------- |
| リカバリ・サービスからの HTTPS トラフィック許可  | データベースが存在する VCN の CIDR | TCP           | All                | 8005           |
| リカバリ・サービスからの SQLNet トラフィック許可 | データベースが存在する VCN の CIDR | TCP           | ALL                | 2484           |

<br>

**セキュリティ・リスト画面**
![img](ExaDB-D_zrcv07.png)

<br>

# 4. リカバリ・サービス・サブネットを登録しよう

次に、先ほどの VCN のサブネットをリカバリ・サービス・サブネットとして、リカバリ・サービス側に登録します。

複数の保護されたデータベースを作成したい場合は、リカバリ・サービス専用のサブネットを新規作成し、それを利用することを検討します。

まず、ナビゲーション・メニューで、[Oracle Database]をクリックし、[データベース・バックアップ]を選択します。

![img](ExaDB-D_zrcv08.png)

[データベース・バックアップ]ページが表示されたら、左のメニュー・バーにある[リカバリ・サービス・サブネット]をクリックします。
そして、「リカバリ・サービス・サブネットの登録」をクリックします。

![img](ExaDB-D_zrcv09.png)

次の内容を入力し、[登録]をクリックします。

- 名前：任意　リカバリ・サービス・サブネットの名前を入力します
- コンパートメント：リカバリ・サービス・サブネットを作成するコンパートメントを選択します
- 仮想クラウド・ネットワーク：使用したいサブネットが存在するデータベース VCN を選択します
- サブネット：リカバリ・サービス用にセキュリティリストを操作したプライベート・サブネットを選択します。ここで指定したサブネットにリカバリ・サービスのエンドポイントが設定されます。

入力が完了したら「登録」からリカバリ・サービス・サブネットを登録しましょう。

![img](ExaDB-D_zrcv10.png)

リカバリ・サービス・サブネットの登録が完了すると、リカバリ・サービス・サブネットの詳細画面に遷移します。

![img](ExaDB-D_zrcv11.png)

<br>

# 5. 保護ポリシーを作成しよう

次に、保護ポリシーを作成していきます。
保護ポリシーは、リカバリ・サービスによって作成されたバックアップを保持する最大期間(日数)を決定します。ビジネス要件に基づいて、保護されているデータベースごとに個別のポリシーを割り当てるか、VCN 内のすべての保護されているデータベースにまたがって単一のポリシーを使用できます。
<br>

[データベース・バックアップ]画面の [保護ポリシー]を開き、[保護ポリシーの作成]をクリックします

![img](ExaDB-D_zrcv12.png)

次の項目を入力し、[作成]をクリックします。

- **名前**：任意　ポリシーの名前を指定します
- **コンパートメントに作成**：保護ポリシーを作成するコンパートメントを選択します
- **バックアップ保持期間(日)**：このポリシーを使用してバックアップを保持する最大日数を指定します。最小 14 日間～最大 95 日間を指定できます。
- **ロックの有効化**：保護ポリシーに対して保持ロックの有効化をします。有効化する場合は、ロックする期間を指定します。ロック期間中は保護ポリシーの延長操作のみができるようになります。イミュータブル・バックアップをご利用になりたい場合、こちらの機能をお使いください。
- **データベースと同じクラウド・プロバイダにバックアップを格納します**：バックアップの保管場所を選択します。マルチクラウドでデータベース・サービスを利用する場合に、この項目を有効化すると、データベース・サービスが配置されているクラウドにバックアップも取得・保管されます。無効化（デフォルト）もしくは OCI のデータベース・サービスの場合は、バックアップの保管場所は OCI のみです。

![img](ExaDB-D_zrcv13.png)

作成が終わると、保護ポリシーが表示されます。

![img](ExaDB-D_zrcv14.png)

左の「保護ポリシー」からコンパートメント内に存在する保護ポリシーの一覧を確認することもできます。

<br>

# 6. ExaDB-D でリカバリ・サービスを有効化しよう

コンソールを使用して、データベース・サービスに対して自動増分バックアップの有効化、オンデマンドでのフル・バックアップの作成、管理されたバックアップのリストの表示を行うことができます。
<br>

データベース・サービスにリカバリ・サービスを設定する方法は以下の 2 つあります。

**A：** データベース・サービスを新規作成すると同時に、リカバリ・サービスも有効化

**B：** 既存のデータベース・サービスにリカバリ・サービスを有効化
<br>

今回のチュートリアルでは B の手順を行いますが、参考として A の方法もご紹介します。
<br>

## A：データベース・サービスを新規作成すると同時に、リカバリ・サービスも有効化

データベース・サービスを新規作成する際に、リカバリ・サービスを設定し、有効化することが可能です。
<br>

データベース・サービスの作成画面の自動バックアップの有効化でリカバリ・サービスの設定をすることができます。この設定はプロビジョニング後でも変更可能です。

※前半でご紹介したリカバリ・サービス・サブネットやネットワークの事前準備が完了している必要があります。

![img](ExaDB-D_zrcv19.png)

データベース・サービスの新規作成方法は[Oracle Cloud で Oracle Database を使おう](../dbcs101-create-db){:target="\_blank"}で学習できます。
<br>

## B：既存のデータベース・サービスにリカバリ・サービスを有効化

既存のデータベース・サービスの自動バックアップの取得先として、リカバリ・サービスを指定します。
<br>

ナビゲーション・メニューの「Oracle Database」をクリックし、「Oracle ベース・データベース・サービスをクリックします。

![img](ExaDB-D_zrcv15.png)
<br>

リカバリ・サービスを設定したい DB システムを選択し、データベース詳細画面に遷移します。

そして、「自動バックアップの構成」をクリックします。

![img](ExaDB-D_zrcv16.png)
<br>

次に、以下項目を入力し、[変更の保存]をクリックします。
<br>

![img](ExaDB-D_zrcv17.png)
<br>

**入力項目と入力内容**

- **自動バックアップの有効化**: 有効化するためにチェック
- **バックアップの保存先**：「自律型リカバリ・サービス(推奨)」（デフォルト）を選択します
- **保護ポリシー**：事前設定された保持期間のポリシー、または、事前定義したカスタム・ポリシーを選択します。保護ポリシーの設定に従って、バックアップの保管場所と保持ロックの有無の情報も表示されます。
- **リアルタイム・データ保護**：<img src="coin.png" alt="coin" width="30"/> REDO 転送オプションの有無を選択します。

**REDO 転送オプションについて**  
リカバリ・サービスには 2 種類のタイプがあります。Autonomous Recovery Service (RCV) と Zero Data Loss Autonomous Recovery Service (ZRCV) です。
この 2 種類の違いは、REDO 転送オプションの有無です。RCV は REDO 転送オプション無し、ZRCV が REDO 転送オプションありのタイプです。
REDO 転送オプションを有効化すると、リアルタイム REDO 転送が実施されるため、DB ストレージ上の REDO ログを損失する障害においても、0 に近いリカバリ・ポイント目標(RPO)が提供されます。
{: .notice--info}

![img](ExaDB-D_zrcv18.png)

> - チェックボックスにチェックあり ＝ Zero Data Loss Autonomous Recovery Service (ZRCV) を利用
> - チェックボックスにチェックなし ＝ Autonomous Recovery Service (RCV) を利用
>   <br>

- **データベース終了後の削除オプション**：データベースの終了後に保護されたデータベース・バックアップを保持するために使用できるオプション。データベースに偶発的または悪意のある障害が発生した場合にバックアップからデータベースをリストアする場合にも役立ちます。
- **日次バックアップのスケジュール時間(UTC)**：増分バックアップが開始される時間ウィンドウを指定します。
- **最初のバックアップをすぐに作成します**：最初の完全バックアップを延期することを選択した場合、データベース障害が発生してもデータベースがリカバリできない可能性があります。

[変更の保存]をクリック後、データベースの詳細画面に入力した内容が表示されます。
「最初のバックアップをすぐに作成します」のチェックボックスにチェックを入れた場合、すぐにバックアップが開始されます。

また、「リアルタイム・データ保護: 有効」になっていると データ損失の危険性の項目は 0 秒になります。

![img](ExaDB-D_zrcv24.png)
<br>

初期バックアップが完了すると、取得されたバックアップが「リソース」の「バックアップ」に表示されます

![img](ExaDB-D_zrcv25.png)

<br>

# 7. 保護されたデータベースの詳細を確認しよう

リカバリ・サービスの有効後、保護されたデータベースの詳細を確認することができます。

データベースの詳細画面の「バックアップ」にある「自律型リカバリ・サービス」をクリックすると、保護されたデータベースの詳細画面に遷移します。

![img](ExaDB-D_zrcv19.png)
<br>

**保護されたデータベースの詳細画面**  
保護されたデータベースの詳細画面には以下の項目が表示されます。
<br>

![img](ExaDB-D_zrcv20.png)

**保護サマリー**

- **ヘルス**：保護されたデータベースのステータス
- **リアルタイム保護**：REDO 転送オプションの有無
- **データ損失の可能性**：最後の有効なバックアップ以降の時間（データ損失の可能性がある期間）
- **保護ポリシー**：適用されている保護ポリシー
- **現在のリカバリ・ウィンドウ**：現在の時刻から開始して、遡ってデータベースをリカバリできる期間
  <br>

**領域使用量**

- **リカバリ・ウィンドウの現在の使用済領域**：現在から遡ってデータベースをリカバリできる期間を満たすために、使用された領域
- **リカバリ・ウィンドウのポリシーに対して予測される使用済領域**：保護ポリシーに基づいた予測使用領域
- **保護されたデータベースのサイズ**：保護されたデータベースのサイズ

※「リカバリ・ウィンドウに使用された領域」は現在と保護ポリシーに基づいた予測の 2 種類表示されます。
<br>

**データベース・バックアップのサマリー**

- **最後に失敗したバックアップ**：最後にバックアップに失敗した時刻
- **最後に完了したバックアップ**：最後に完了したバックアップを取得した時刻
- **最終バックアップ期間**：バックアップにかかった時間
  <br>

**保護されたデータベース**

- **データベース詳細**：保護されたデータベースの名前
- **一意のデータベース名**：保護されたデータベースの一意のデータベース名
- **データベースのバージョン**：保護されたデータベースのバージョン
- **コンパートメント**：保護されたデータベースが存在するコンパートメント
  <br>

**一般情報**

- リカバリ・サービスが有効化されている保護 DB の情報
  <br>

**リカバリ・サービスの領域使用量に必要な要素な次のようなものがあります。**

1. **データベースのサイズ**: 保護されたデータベースのサイズ
2. **週次仮想フルバックアップ**: ZRCV のデータ保持期間の週数分のフルバックアップのサイズ
3. **Redo ログサイズ**: 随時出力される Redo サイズ
4. **日次増分バックアップ**: 日次増分バックアップの日数分のサイズ  
   {: .notice--info}

「モニタリング」のタブから、リカバリ・サービスのメトリックを確認できます。
<br>

**メトリック**

- **リカバリ・ウィンドウに使用された領域**：保護されたデータベースのリカバリ・ウィンドウの目標を満たすために現在使用されているストレージ領域の量
- **保護されたデータベースのサイズ**：保護されたデータベースによって消費されたストレージ領域の合計
- **保護されたデータベース・ヘルス**：データベースの現在の保護ステータス
- **データ損失の危険性**：最後の有効なバックアップ以降の時間(データ損失の可能性がある期間)

![img](ExaDB-D_zrcv21.png)

<br>

「ネットワークの詳細」のタブから、リカバリ・サービス。サブネットとして登録されているサブネットの情報を確認できます。
<br>

# 8. 保護されたデータベースの一覧を確認しよう

同一コンパートメント内で保護されているデータベースの一覧を見ることができます。

保護されたデータベースの詳細画面の左上にある「保護されたデータベース」をクリックします。

![img](ExaDB-D_zrcv23.png)

すると、コンパートメント内で保護されているデータベースの一覧表示されます。
複数の保護されたデータベースの保護状態を 1 つの画面で確認できます。

![img](ExaDB-D_zrcv22.png)
<br>

<br>
以上で、この章の作業は完了です。

<br>

# 参考資料

- [製品サイト] [Oracle Database Autonomous Recovery Service](https://www.oracle.com/jp/database/zero-data-loss-autonomous-recovery-service/){:target="\_blank"}

- [マニュアル] [Oracle Database Autonomous Recovery Service](https://docs.oracle.com/cd/E83857_01/paas/recovery-service/index.html){:target="\_blank"}

- [ブログ] [Zero Data Loss Autonomous Recovery Service (ZRCV) を Exadata Cloud Service へ設定してみてみた](https://qiita.com/shirok/items/c257d52984442a7977f8){:target="\_blank"}

- [ブログ] [Autonomous Recovery Service セットアップ・チェックリスト](https://blogs.oracle.com/oracle4engineer/post/ja-autonomous-recovery-service-checklist){:target="\_blank"}
  <br>

<br>
[ページトップへ戻る](#anchor0)
