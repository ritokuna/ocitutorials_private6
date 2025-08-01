---
title: "603 : データ・カタログを使ってメタデータを収集しよう"
excerpt: "データ・カタログを使えば、技術、ビジネスおよび運用に役立つメタデータを管理することができます。Autonomous DatabaseやObject Storageなどのデータ・ソースに接続し、メタデータを収集することができます。"
order: "3_603"
layout: single
header:
  teaser: "../adb603-data-catalog/3-1.png"
  overlay_image: "../adb603-data-catalog/3-1.png"
  overlay_filter: rgba(34, 66, 55, 0.7)
#link: https://apexapps.oracle.com/pls/apex/dbpm/r/livelabs/view-workshop?wid=776
---
<a id="anchor0"></a>

# はじめに
昨今、多くの企業が自社の持つデータを分析してビジネスに役立てようとしています。しかし実際のところ、分析シナリオに最適なデータ準備やデータの把握、管理が困難で、データマネジメントには多くの課題が存在しています。

Oracle Cloud Infrastructure Data Catalogは、そのような企業データのフルマネージドのデータ検出および管理を行うソリューションです。Data Catalogを使うと、技術、ビジネスおよび運用に役立つメタデータを管理するための単一のコラボレーション環境を作成できます。データを必要とする誰もが、単一のインタフェースから、専門知識不要でデータを検索することができます。

<br>

# Data Catalogの主要な機能
+ **メタデータの収集**：カタログ化したいデータストア（データベースやオブジェクトストレージ）を指定し、Data Catalogの中にメタデータを抽出します。定期的にスケジュール実行も可能です。

+ **データへのタグ付け**：表や列、ファイルなどのデータを論理的に識別するためのキーワード（自由書式）を設定できます。これにより、特定のキーワードでタグ付けされた全てのデータを検索することができます。

+ **ビジネス用語集**：組織内であらかじめ決められた用語を使って、データに検索、分類のための目印を付与することができます。

+ **データの検索**：SQLやRESTではなくキーワードでの検索、表名・列名・ファイル名での検索、特定のタグやビジネス用語に合致するデータの検索を全て行うことができます。

**目次 :**
  + [1.データの準備](#anchor1)
  + [2.データ・カタログの作成](#anchor2)
  + [3.Autonomous Databaseからメタデータを収集](#anchor3)
  + [4.Object Storageからメタデータを収集](#anchor4)
  + [5.ビジネス用語集とカスタム・プロパティの作成](#anchor5)
  + [6.メタデータの補完](#anchor6)
  + [7.データの検索](#anchor7)
  + [8.おわりに](#anchor8)

**前提条件**
+ Autonomous Data Warehouse(ADW)インスタンスが構成済みであること
    <br>※ADBインタンスの作成方法については、
    [101:ADBインスタンスを作成してみよう](../adb101-provisioning) を参照ください。

+ Data Catalogを使用するためのユーザーグループ、ポリシーが設定済みであること
    <br>※本チュートリアルを進めるうえで必要なポリシーは[こちら](https://docs.oracle.com/ja-jp/iaas/data-catalog/using/policies.htm#policy-examples)を参照ください。

<br>

**所要時間 :** 約2時間

<br>

<a id="anchor1"></a>

# 1. データの準備
データ・カタログのタスクを行うために必要なデータベース・オブジェクトをSQLスクリプトを実行することで作成します。

1. ADMINユーザーでADWに接続し、以下のスクリプトを実行して、**sales_history**というユーザーを作成します。
```sql
CREATE USER sales_history IDENTIFIED BY Welcome12345#;
GRANT CREATE table TO sales_history;
GRANT CREATE view TO sales_history;
GRANT CREATE type TO sales_history;
GRANT CREATE session to sales_history;
GRANT UNLIMITED TABLESPACE TO sales_history;
GRANT CREATE sequence TO sales_history;
GRANT CREATE procedure to sales_history;
GRANT CREATE trigger to sales_history;
```

1. [Data Catalogで使用するファイル](../adb603-data-catalog/DCAT_Workshopfiles.zip)からzipファイルをダウンロードし、解凍します。

2. zipファイルの中の**DCAT_Livelabs.sql**というSQLスクリプトを実行します。実行すると、いくつかの表が作成されます。

これでデータの準備は完了です。
<br>

<a id="anchor2"></a>

# 2. データ・カタログの作成
Data Catalogインスタンスの作成にあたり、1つのフォームを埋めていきます。なお、サービス管理、パッチ適用、バックアップとリストアおよびその他のサービスライフサイクルタスクは、オラクルが行います。

1. OCIコンソールのメニューから*アナリティクスとAI*をクリックし、*データ・レイク*の下の*データ・カタログ*をクリックします。
![2-1](2-1.png)

1. データ・カタログの概観が表示されるので、左のリストから*データ・カタログ*をクリックします。
![2-2](2-2.png)

1. データ・カタログのページが表示されるので、左のパネルからカタログを作成するコンパートメントを選択します。

1. **データ・カタログの作成**をクリックします。
![2-3](2-3.png)

1. 再度データ・カタログインスタンスを作成するコンパートメントを選択します。

1. インスタンス名を入力します。
![2-4](2-4.png)


これでデータ・カタログの作成は完了です。
<br>

<a id="anchor3"></a>

# 3. Autonomous Databaseからメタデータを収集

データソースは、カタログ内のデータ・アセットで表されます。本チュートリアルでは、**前提条件**で作成済みのADW用のデータ・アセットを作成します。

その後、接続情報を追加し、ADWからスキーマと表のメタデータを収集します。

## 3-1. Autonomous Databaseのデータ・アセットの作成
1. データ・カタログインスタンスのホーム画面の*クイック・アクション*から**データ・アセットの作成**をクリックします。
![3-1](3-1.png)

1. データ・アセットの作成パネルから、一意のデータ・アセット名を入力します。

1. オプションの*説明*に、作成の目的等を入力します。

1. ドロップダウンリスト*タイプ*に、**Autonomous Data Warehouse**を選択します。

1. *データベース名*にADWのデータベース名を入力します。

1. ADWがプライベート・エンドポイントの場合は、**プライベート・エンドポイントの使用**にチェックを入れます。

※今回はパブリック・エンドポイントのADWなのでチェックは入れません。
![3-2](3-2.png)

<br>

## 3-2. 接続の追加
データソースをデータ・アセットとして、データ・カタログに登録後、データ・アセットへの接続を作成します。なお、データソースへの接続は複数作成できます。
1. データ・アセット詳細画面のサマリータブの**接続の追加**をクリックします。
![3-3](3-3.png)

1. 接続の追加パネルで、一意の接続名を入力します。

1. オプションの*説明*に、追加の目的等を入力します。

1. ドロップダウンリスト*タイプ*で、**Generic**をクリックします。

1. ラジオボタン*ウォレットの使用*を選択し、ADWのクレデンシャル・ウォレット(zipファイル)をアップロードします。

1. ドロップダウンリスト*TNS別名*で、ADWの接続サービスを選択します。今回は_lowを指定します。

1. *ユーザー名*は、**ADMIN**ユーザー、*パスワード*はADMINユーザーのパスワードを入力します。

1. この接続をデータ・アセットのデフォルトの接続にしたい場合は、チェックを入れます。

1. **接続のテスト**をクリックすると、テスト接続が成功or失敗したか表示されます。
![3-4](3-4.png)

<br>

## 3-3. Autonomous Databaseからメタデータを収集
1. 接続を追加後、データ・アセットのサマリータブの**収集**をクリックします。
![3-5](3-5.png)

1. 接続の選択では、デフォルト接続が選択された状態で表示されます。
![3-6](3-6.png)

1. データ・エンティティの選択では、利用可能な全てのスキーマが表示されます。スキーマをエンティティとして追加もできますし、テーブル単位で追加することもできます。**SALES_HISTORY**スキーマを選択し、追加します。
![3-7](3-7.png)

1. ジョブの作成タブが表示されるので、ジョブ名に一意の名前を入力します。オプションで*説明*を追加します。

1. このジョブの最初の実行以降に変更されたデータ・エンティティのみ収集する場合は、増分収集にチェックを入れます。

1. *実行時間*でジョブの実行時間を指定します。スケジューリングも可能ですが、今回はすぐに実行します。
![3-8](3-8.png)

1. 収集ジョブが正常に作成され、ジョブタブが表示されます。ここではジョブのステータスを確認したり、詳細を表示できます。

1. データ・アセットの詳細ページで**Schemas**をクリックし、**リフレッシュ**を行うと、収集したスキーマが表示されます。
![3-9](3-9.png)

<br>

<a id="anchor4"></a>

# 4. Object Storageからメタデータを収集
一般的に、データレイクには、半構造化ファイルや非構造化ファイルが含まれます。ファイルはさまざまな形式の独立したファイルであったり、Sparkジョブから得られるパーティション化されたファイルのときもあります。

データ・カタログを使用すると、これらのファイルを簡単に見つけ、解釈することに役立ちます。ここではObject Storageデータ・アセットを作成し、ファイル名パターンからデータを収集します。

## 4-1. 動的グループとポリシーの作成
OCI Object Storageに対してAPIコールを行うことを許可するポリシーを作成します。

ここの手順については、[こちら](https://oracle-livelabs.github.io/oci-core/data-catalog/workshops/freetier/index.html?customTrackingParam=:ow:lp:cpo::::RC_WWMK211125P00013:llid=919&lab=harvest-object-store)をご参照ください。

<br>

## 4-2. Object Storageのデータ・アセットの作成
データ・カタログインスタンスのホーム画面の*クイック・アクション*から**データ・アセットの作成**をクリックします。
![3-1](3-1.png)

データ・アセットの作成パネルでデータ・アセットの詳細を次のように指定します。

+ **名前**：Oracle-Object-Storage-Data-Asset
+ **タイプ**：Oracle Object Storage
+ **URL**：https://swiftobjectstorage.us-ashburn-1.oraclecloud.com
  > このチュートリアルでは、データが格納されている既存のOracle Object Storageバケットにアクセスします。このバケットは、us-ashburn-1リージョンのc4u04テナンシ―の中にあります。
  > 次の手順では、事前認証済み要求(PAR)を使用して、データへの接続を追加します。PARの詳細は[こちら](https://docs.oracle.com/ja-jp/iaas/Content/Object/Tasks/usingpreauthenticatedrequests.htm)をご参照ください。
+ **ネームスペース**：c4u04

![4-1](4-1.png)

<br>

## 4-3. 接続の追加
[3-2. 接続の追加](#3-2-接続の追加)と同様の手順でObject Storageへの接続を追加します。
接続の追加パネルで接続の詳細を次のように指定します。

+ **名前**：moviestream-landing-bucket-connection
+ **タイプ**：Pre-Authenticated Request
+ **事前認証済リクエストのURL**：https://objectstorage.us-ashburn-1.oraclecloud.com/p/YtpqXpUpPx1pPXFQa4Githwxx4bxp12q2yZJsCyzN0Y9-kpYr5nAOvLvwZfLHxXF/n/c4u04/b/moviestream_landing/o/

![4-2](4-2.png)


接続が正常に追加されると、moviestream-landing-bucket-connection データソース接続が接続セクションに表示されます。

![4-3](4-3.png)

<br>

## 4-4. ファイル名パターンの作成とデータ・アセットへの割当て
データレイクには通常、1つのデータセットを表す多数のファイルがあります。データ・カタログでは、ファイル名パターンを使用して、複数のObject Storageファイルを論理データ・エンティティにグループ化することができます。

ファイル名パターンとは、ファイルの検索と検出に使用できる論理データ・エンティティを作成するための正規表現です。論理データ・エンティティを作成すると、データレイクのコンテンツを整理し、カタログのエンティティや属性が爆発的に増加するのを防ぐことができます。

1. データ・カタログインスタンスのホーム画面から＋をクリックし、**ファイル名パターン**をクリックします。
![4-4](4-4.png)

1. **ファイル名パターンの作成**をクリックし、パネルで次のように入力します。

+ **名前**：Map Object Storage Folders to DCAT Logical Entities
+ **正規表現**を選択
+ **式**：{bucketName:[A-Za-z0-9\.\-_]+}/{logicalEntity:[^/]+}/\S+$
  >> *パターンの例の表示*をクリックすると、ファイル名、パターン式、およびパターン式に基づいて派生する論理データ・エンティティ名の例を表示できます。
+ **テスト式**：以下のファイル名を*テスト・ファイル名*に入力
  ```
  moviestream_landing/customer/customer.csv
  moviestream_gold/sales/time=jan/file1.parquet
  moviestream_gold/sales/time=feb/file1.parquet
  ```
![4-5](4-5.png)

1. ファイル名パターンの作成後、それをデータ・アセットに割当てます。データ・アセットのリストから、**Oracle Object Storage Data Asset**を選択します。
![4-6](4-6.png)

1. サマリータブの下部、*ファイル名パターン*セクションの**ファイル名パターンの割当て**をクリックします。
![4-7](4-7.png)

1. ファイル名パターンの割当てパネルから、このデータ・アセットに割り当てるファイル名パターンを選択します。**Map Object Storage Folders to DCAT Logical Entities**にチェックを入れ、割当てます。
![4-8](4-8.png)

1. 割当てが成功すると、このように表示されます。これにより、Object Storageバケット内のファイル名がパターン式と照合され、論理データ・エンティティが形成されます。
![4-9](4-9.png)

<br>

## 4-5. Object Storageからメタデータを収集
データ・アセットの作成後、Object Storageにあるデータの構造情報をデータ・カタログに収集し、そのデータ・エンティティと属性を表示することができます。

1. データ・アセットのリストから、**Oracle Object Storage Data Asset**を選択します。
![4-6](4-6.png)

1. Oracle Object Storage Assetページが表示されるので、**収集**をクリックします。
![4-10](4-10.png)

1. 収集するデータ・アセットへの接続は、**moviestream-landing-bucket-connection**を選択します。
![4-11](4-11.png)

1. データ・エンティティの選択では、**moviestream_landing**を選択します。タイプは以下のように*BUCKET*となっています。
![4-12](4-12.png)

1. ジョブの作成ページでは以下のように入力します。
+ **ジョブ名**：デフォルトの名前のままにします。
+ **ジョブ説明**：オプションの説明を入力します。今回は空白のままにしておきます。
+ **増分収集**：チェックは入れません。
+ **認識できないファイルを含める**：チェックは入れません。
+ **一致ファイルのみを含む**：チェックを入れます。Object Storage内の指定した割当てファイル名パターンに一致するファイルのみ収集する場合に選択します。一致しないファイルは収集されず、スキップされたファイルに追加されます。
+ **実行時間**：今すぐジョブを実行を選択
![4-13](4-13.png)

1. ジョブのタブが表示されています。このObject Storageのアセットに割り当てたファイル名パターンを使用して収集された論理エンティティの数として11と表示されています。この数は、moviestream_landingルートバケット下のサブフォルダの数を表しています。また、対応するファイルが57個あります。
![4-14](4-14.png)

1. Oracle Object Storage Data Assetのページで、*Buckets*タブをクリックし、**リフレッシュ**をクリックすると、**Landing**が表示されます。
![4-15](4-15.png)

1. **Landing**の詳細ページに行き、*データ・エンティティ*タブをクリックし、**custsales**をクリックします。
![4-16](4-16.png)

1. サマリータブが表示されます。ここでは、データ・エンティティのデフォルトプロパティ、カスタムプロパティ、ビジネス用語集の用語とカテゴリ、および推奨事項を確認できます。
![4-17](4-17.png)

1. *属性*タブをクリックすると、データ・エンティティの属性を確認できます。
![4-18](4-18.png)

<br>

<a id="anchor5"></a>

# 5. ビジネス用語集とカスタム・プロパティの作成
多くの組織では、ビジネスの概念や関連するデータに関する共通の用語がないために、コミュニケーションの問題や誤解が発生しています。例えば、「顧客」という単純な概念が、5つの異なる部門にとって5つの異なる意味を持つ可能性があります。顧客によっては、これらの概念をエクセル・シートやワード・ドキュメントで管理していますが、これは拡張性のあるモデルと言えません。ビジネス用語集は、組織で使用される用語を定義するものです。さらに、技術的なメタデータを用語集とリンクさせることも可能です。

本章では、データ管理者として、用語集を定義してみます。また、データの所有者を把握するため、Data Ownerというカスタムプロパティを作成します。

## 5-1. ビジネス用語集の作成

1. データ・カタログのホームタブから**用語集**をクリックします。
![5-1](5-1.png)

1. **用語集の作成**をクリックし、*名前*を入力します。**data-catalog-tutorial-Glossary**とします。
![5-2](5-2.png)

<br>

## 5-2. 用語集のインポート
カテゴリーと用語は、一つずつ作成するか、Excelファイルからインポートする方法があります。

1. **インポート**をクリックします。
![5-3](5-3.png)

1. インポート時にリッチ・テキストの書式が失われる可能性があることを示すダイアログが表示されます。確認し、**続行**をクリックします。
![5-4](5-4.png)

1. zipファイル内の**data-catalog-ADB-GlossaryExport.xlsx**を選択します。

1. 用語集のインポートジョブが正常に完了すると、Excelファイルの内容が用語集にインポートされ、左パネルに表示されます。
![5-5](5-5.png)

<br>

## 5-3. カテゴリーの作成
用語集では、各用語はカテゴリ内に作成する必要があるので、まずはカテゴリーを作成します。

1. 用語集の詳細タブで**カテゴリの作成**をクリックします。
![5-6](5-6.png)

1. カテゴリの作成パネルで、名前と説明を入力します。
![5-7](5-7.png)

1. カテゴリが作成されると、用語集の詳細タブの階層リストに表示され、このカテゴリ内に用語を作成することができます。また、別のカテゴリの中にカテゴリを作成し、ネストされたカテゴリを作成することもできます。
![5-8](5-8.png)

<br>

## 5-4. 用語の作成
収集されたデータ・エンティティおよび属性を分類するには、用語を使用します。各用語は、カテゴリ内で作成する必要があります。

1. カテゴリーの詳細タブで**用語の作成**をクリックします。
![5-9](5-9.png)

1. 用語の作成パネルで名前と説明を入力します。
![5-10](5-10.png)

1. 用語の作成が完了すると、カテゴリー**リージョン**の階層に国という用語が表示されます。
![5-11](5-11.png)

<br>

## 5-5. 用語集のエクスポート
カタログ内の用語集は、別のカタログにインポートするためにエクスポートすることができます。

1. 用語集の詳細タブで**エクスポート**をクリックします。
![5-12](5-12.png)

1. インポート時にリッチ・テキストの書式が失われる可能性があることを示すダイアログが表示されます。確認し、**続行**をクリックします。
![5-4](5-4.png)

1. Excelファイルで用語集のエクスポートできます。

1. ファイルを開いて詳細を確認します。
![5-13](5-13.png)

<br>

## 5-6. カスタム・プロパティの作成
カスタム・プロパティは、データ・カタログオブジェクトのビジネスのコンテキストを把握するためのメタデータを定義するために作成します。

データ・カタログを使ってメタデータを収集すると、データ・エンティティと属性のためのいくつかのデフォルト・プロパティがデータ・カタログに作成されます。例えば、説明や更新者、最終更新日などです。

しかし、デフォルト・プロパティだけでは、全てのコンテキストを把握するには十分でない場合があります。そのような場合に、ビジネス記述、更新頻度、認証ステータス、データ所有者などを追加することができます。これにより、データ利用者の理解や分類がしやすくなります。

1. データ・カタログのホームタブから**カスタム・プロパティ**をクリックします。
![5-14](5-14.png)

1. **カスタム・プロパティの作成**をクリックし、作成パネルで以下を入力します。
+ **名前**：Data Owners
+ **データ型**：String(Plain Text)
+ **値リストの使用**：チェックを入れます。
+ **複数値の許可**：チェックを入れます。
+ **値リスト**：Andrew, Brian, Clara, David
+ **データ・カタログ・オブジェクト・タイプ**：Data Entity
+ **検索結果に表示**：チェックを入れます。
+ **フィルタリングを許可**：チェックを入れます。
+ **ソートを許可**：チェックを入れます。
![5-15](5-15.png)

1. カスタム・プロパティが作成できました。
![5-16](5-16.png)

<br>

<a id="anchor6"></a>

# 6. メタデータの補完
データソースから取得できるメタデータには、表名、列名、データ型などが含まれます。しかし、データ利用者としてはこれらのメタデータはデータを理解するのに十分な情報とは言えません。例えば、そのデータは何についてのもので、どのような用途に使われるのか、そのデータの所有者は誰かなどの情報です。

本章では、用語集とカスタム・プロパティを使用して、前章で収集したメタデータを補完します。

## 6-1. カタログ・オブジェクトのカスタム・プロパティの設定
カスタム・プロパティの値は、関連するタイプの各オブジェクトに設定することができます。

1. データ・カタログのホームタブから**データ・エンティティ**をクリックします。
![6-1](6-1.png)

1. **Table：CHANNELS**をクリックし、詳細ページの**Custom Properties**の**編集**をクリックします。
![6-2](6-2.png)

1. Data Ownersに**Andrew**と**Brian**を追加します。
![6-3](6-3.png)

<br>

## 6-2. エクスポート/インポートを使ったカスタム・プロパティの一括設定
カタログ内のオブジェクトに、カスタム・プロパティの値を入力することは、データ量によっては大変な作業になり得ます。そのような場合には、カスタム・プロパティのエクスポート/インポートを使用して、一括して設定することができます。

1. データ・カタログのホームタブで、**SALES_HISTORY**と入力し、検索します。
![6-4](6-4.png)

1. SALES_HISTORYの詳細ページの**カスタム・プロパティのエクスポート**をクリックします。
![6-5](6-5.png)

1. カスタム・プロパティをエクスポートするオブジェクト・タイプを選択します。ここでは、**Data Entities**と**Attributes**を選択します。
![6-6](6-6.png)

1. エクスポートしたファイルから、データ・エンティティシートを開き、既存のカスタム・プロパティを確認します。
![6-7](6-7.png)

1. 各オブジェクトのカスタム・プロパティに、値のセットを入力します。
![6-8](6-8.png)

1. **カスタム・プロパティのインポート**をクリックし、Excelファイルをインポートします。
![6-9](6-9.png)

1. 正常にインポートが完了すると、データ・エンティティのリストに**COUNTRIES**が追加されます。
![6-10](6-10.png)

<br>

## 6-3. タグのリンク
タグは、オブジェクトレベルで特定の詳細を記述するための自由形式の注釈です。タグのリンク機能を使うと、複数のデータ・エンティティに一括でタグを設定することができます。

1. データ・アセットの詳細ページで**Sales**というタグを作成します。
![6-11](6-11.png)

1. データ・エンティティのリストページで**PRODUCTS**、**PROMOTIONS**、**SALES**にチェックを入れます。
![6-12](6-12.png)

1. **タグのリンク**をクリックします。
![6-13](6-13.png)

1. タグの一覧が表示されるので、**Sales**を選択しリンクします。
![6-14](6-14.png)


これでタグのリンクができました。

<br>

## 6-4. 用語のリンク
カタログのオブジェクトに用語集をリンクすることも可能です。

1. データ・エンティティのリストページから、用語をリンクさせたいエンティティを選択します。
![6-15](6-15.png)

1. ページ下部**用語とカテゴリのリンク**をクリックします。
![6-16](6-16.png)

1. 一覧表示されるので、検索ボックスに**Country**と入力し、検索します。
![6-17](6-17.png)

1. 用語の詳細ページで**リンクされたオブジェクト**タブをクリックします。
![6-18](6-18.png)

1. 推奨事項欄で全てをチェックし、**同意**をクリックします。
![6-19](6-19.png)

1. これで用語がリンクされた全てのオブジェクトが表示されるようになります。
![6-20](6-20.png)

<br>

<a id="anchor7"></a>

# 7. データの検索
データ・カタログを使用することでメタデータを収集し、ユーザーはビジネス用語や名前を使用して、関連するデータを検索することができます。

## 7-1. 検索フィールドの使用
ここでは、データアナリストとして、所得水準に基づく顧客の与信限度額を分析します。まずは検索フィールドを使用して、ブライアンが所有する顧客関連のデータ・エンティティを検索します。その後、与信限度額と所得水準に関連する属性を確認します。

1. 検索フィールドに**CUSTOMER**と入力します。すると、候補となるテキストが表示されますので、選択します。
![7-1](7-1.png)

1. CUSTOMERが名前に含まれるオブジェクト一覧が表示されるので、フィルタを使用して結果を絞り込みます。まずオブジェクト・タイプを**データ・エンティティ**とし、さらにData Ownersを**Brian**でフィルタリングします。すると、これらのフィルタ条件に該当するエンティティが1つだけ表示されるはずです。
![7-2](7-2.png)

1. CUSTOMERSの詳細ページを見ると、**属性の数**が23となっています。
![7-3](7-3.png)

1. 属性タブをクリックして、さらに属性を調べてみます。今回のユースケースに関連する属性である所得水準と与信限度額があるかどうか確認します。2ページ目に**CUST_INCOME_LEVEL**と**CUST_CREDIT_LIMIT**があります。
![7-4](7-4.png)

<br>

## 7-2. データ・アセットの参照
*データ・アセットの参照*を使用すると、データ・カタログで作成されたすべてのオブジェクトを簡単に表示することができます。

1. ホームタブのクイック・アクションから**データ・アセットの参照**をクリックします。
![7-5](7-5.png)

1. 利用可能なすべてのデータ・アセットが表示されます。**data-catalog-ADB-Data-Asset**をクリックすると、その詳細を概要タブに表示されます。
![7-6](7-6.png)

1. Schemasタブをクリックすると、data-catalog-ADB-Data-Assetで利用可能なスキーマが表示されます。ここでは、**SALES_HISTORY**スキーマが表示されます。
![7-7](7-7.png)

1. さらに左枠の**data-catalog-ADB-Data-Asset**をクリックすると、スキーマを掘り下げることができます。
![7-8](7-8.png)

1. さらに**SALES_HISTORY**をクリックすると、このデータ・アセットの下で利用可能なデータ・エンティティのリストが表示されます。
![7-9](7-9.png)

1. さらに**CUSTOMERS**をクリックすると、CUSTOMERS表の属性のリストが表示されます。
![7-10](7-10.png)

<br>

<a id="anchor8"></a>

# おわりに
本章では、データ・カタログを使ってメタデータを収集して、データの検索や理解に役立つ情報を確認する方法をご紹介しました。他にもビジネス用語集を使用して共通の用語を作成及び管理する方法やユーザー定義のカスタム・プロパティを作成し、各オブジェクトに固有の情報を設定する方法をご紹介しました。これにより、データアナリストやデータ利用者は効率的にデータを探索することができるようになります。

<br>

# 参考資料
<!--
* [Get started with Oracle Cloud Infrastructure Data Catalog](https://oracle.github.io/learning-library/data-management-library/data-catalog/workshops/freetier/?lab=introduction) 
-->
* [データ・カタログの概要ドキュメント](https://docs.oracle.com/ja-jp/iaas/data-catalog/using/overview.htm)

<br/>
以上でこの章は終了です。次の章にお進みください。

<br>

[ページトップへ戻る](#anchor0)