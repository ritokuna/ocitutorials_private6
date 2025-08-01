---
title: "304 : OCI Database Migration Serviceを使用したデータベース移行の前準備"
excerpt: "OCI Database Migration Serviceの使用に必要なネットワークやストレージの設定、データベースの登録など事前に必要な設定について紹介します。"
order: "3_304"
layout: single
header:
  teaser: "../adb304-database-migration-prep/teaser.png"
  overlay_image: "../adb304-database-migration-prep/teaser.png"
  overlay_filter: rgba(34, 66, 55, 0.7)
#link: https://apexapps.oracle.com/pls/apex/dbpm/r/livelabs/view-workshop?wid=797
---
<a id="anchor0"></a>

# はじめに
**Oracle Cloud Infrastructure Database Migration Service (DMS)** は、オンプレミスまたはOCI上のOracle DatabaseからOCI上のデータベースに移行する際に利用できるマネージド・サービスです。エンタープライズ向けの強力なオラクル・ツール(Zero Downtime Migration、GoldenGate、Data Pump)をベースとしています。

DMSでは下記の2つの論理的移行が可能です。
+ **オフライン移行** - ソース・データベースのポイント・イン・タイム・コピーがターゲット・データベースに作成されます。移行中のソース・データベースへの変更はコピーされないため、移行中はアプリケーションをオフラインのままにする必要があります。
+ **オンライン移行** - ソース・データベースのポイント・イン・タイム・コピーがターゲット・データベースに作成されるのに加え、内部的にOracle GoldenGateによるレプリケーションを利用しているため、移行中のソース・データベースへの変更も全てコピーされます。そのため、アプリケーションをオンラインのまま移行を行うことが可能で、移行に伴うアプリケーションのダウンタイムを極小化することができます。

DMSに関するチュートリアルは[304 : OCI Database Migration Serviceを使用したデータベース移行の前準備](../adb304-database-migration-prep)、[305 : OCI Database Migration Serviceを使用したデータベースのオフライン移行](../adb305-database-migration-offline)、[306 : OCI Database Migration Serviceを使用したデータベースのオンライン移行](../adb306-database-migration-online)の計3章を含めた3部構成となっています。
DMSを使用してBaseDBで作成したソース・データベースからADBのターゲット・データベースにデータ移行を行います。

[305 : OCI Database Migration Serviceを使用したデータベースのオフライン移行](../adb305-database-migration-offline)または[306 : OCI Database Migration Serviceを使用したデータベースのオンライン移行](../adb306-database-migration-online)を実施する前に必ず[304 : OCI Database Migration Serviceを使用したデータベース移行の前準備](../adb304-database-migration-prep)を実施するようにしてください。

この章では、DMSを使用したデータベース移行の前準備について紹介します。
![](2022-03-24-10-00-25.png)
   
**目次 :**
  + [1. 環境のセットアップ](#anchor1)
    + [1-1. 仮想クラウド・ネットワーク・サブネットのセキュリティ・リストの更新](#anchor2)
    + [1-2. Vaultの作成](#anchor3)
    + [1-3. オブジェクト・ストレージ・バケットの作成](#anchor4)
  + [2. ソース・データベースの設定](#anchor5)
    + [2-1. CDBとPDBの接続情報の確認](#anchor6)
    + [2-2. データ・ポンプ・エクスポートで使用されるディレクトリの作成](#anchor7)
    + [2-3. 初期化パラメータSTREAMS_POOL_SIZEの設定](#anchor8)
  + [3. データベース接続の作成](#anchor9)
    + [3-1. ソースCDBの接続の作成](#anchor10)
    + [3-2. ソースPDBの接続の作成](#anchor11)
    + [3-3. ターゲットADBの接続の作成](#anchor12)


**前提条件 :**
+ [「その2 - クラウドに仮想ネットワーク(VCN)を作る」](https://oracle-japan.github.io/ocitutorials/beginners/creating-vcn/)を参考に、VCNが作成されていること。
+ [「101: Oracle Cloud で Oracle Database を使おう(BaseDB)」](https://oracle-japan.github.io/ocitutorials/basedb/dbcs101-create-db/)を参考に、BaseDBでデータベースとスキーマの作成が完了していること。本チュートリアルではデータベース・バージョンは19.13.0.0.0を使用しています。また、DMSではSSH秘密鍵はRSA形式のみサポートしています。（OPENSSH形式は使用できません）
+ [「101:ADBインスタンスを作成してみよう」](../adb101-provisioning/)を参考に、ADBの作成が完了していること。
+ [「Oracle Cloud Infrastructure Database移行サービスの使用 - 2 Oracle Cloud Infrastructure Database移行の開始 - データベース移行ユーザーへの権限の付与」](https://docs.oracle.com/cd/E83857_01/paas/database-migration/dmsus/getting-started-oracle-cloud-infrastructure-database-migration.html#GUID-478467C2-662A-4C06-8077-EBB5A9F94E64)を参考に、データベース移行ユーザへ権限が付与されていること。権限が付与されていない場合、DMSの利用ができません。

**所要時間 :** 約40分

<BR>

<a id="anchor1"></a>

# 1. 環境のセットアップ
DMSの実行に必要なネットワークやストレージなどの環境のセットアップを行います。

<a id="anchor2"></a>

## 1-1. 仮想クラウド・ネットワーク・サブネットのセキュリティ・リストの更新

1. OCIコンソール・メニューから **ネットワーキング** → **仮想クラウド・ネットワーク** に移動し、前提条件で作成したVCNを選択します。

    ![](2022-02-28-17-47-16.png)

2. **サブネット** の一覧から **Public Subnet-VCN名** を選択します。

    ![](2022-02-28-17-55-37.png)

3. **セキュリティリスト** の一覧から **Default Security List for VCN名** を選択します。

    ![](2022-02-28-17-56-00.png)

4. **イングレス・ルールの追加** を選択します。

    ![](2022-02-28-17-58-17.png)

5. 以下のように設定します。その他の入力項目はデフォルトのままにします。
    + **ソースCIDR** - 0.0.0.0/0
    + **宛先ポート範囲** - 443
    + **イングレス・ルールの追加** をクリックします。

    ![](2022-02-28-18-06-34.png)

6. **イングレス・ルールの追加** を選択します。

7. 以下のように設定します。その他の入力項目はデフォルトのままにします。
    + **ソースCIDR** - 10.0.0.0/16
    + **宛先ポート範囲** - 1521
    + **イングレス・ルールの追加** をクリックします。

    ![](2022-02-28-18-08-25.png)

    ![](2022-02-28-18-09-27.png)

<a id="anchor3"></a>

## 1-2. Vaultの作成

既にVaultが存在する場合、この項目は必要ないため次の項目へ進んでください。

1. OCIコンソール・メニューから **アイデンティティとセキュリティ** → **ボールト** に移動します。

    ![](2022-02-28-18-13-55.png)

2. 画面左側にある **コンパートメント** の一覧から使用したいコンパートメントを選択します。

3. **ボールトの作成** をクリックします。

    ![](2022-02-28-18-17-31.png)

4. **ボールトの作成** ダイアログで、任意の名前を入力します。

5. **ボールトの作成** をクリックしてダイアログを閉じます。

    ![](2022-03-10-17-42-39.png)

6. 作成したボールトの状態が **アクティブ** になるまで待ちます。（5分ほどかかります。）

    ![](2022-02-28-18-22-02.png)

7. 作成したボールトをクリックし、 **マスター暗号化キー** で **キーの作成** をクリックします。

    ![](2022-03-24-12-17-09.png)

8. **キーの作成** ダイアログで、任意の名前を入力します。

9. **キーの作成** をクリックしてダイアログを閉じます。

    ![](2022-03-10-17-43-12.png)

    ![](2022-02-28-18-28-30.png)

<a id="anchor4"></a>

## 1-3. オブジェクト・ストレージ・バケットの作成

移行で使用する空のオブジェクト・ストレージ・バケットを作成します。

1. OCIコンソール・メニューから **ストレージ** → **オブジェクト・ストレージとアーカイブ・ストレージ**に移動します。

    ![](2022-02-28-18-30-18.png)

2. **バケットの作成** をクリックします。

    ![](2022-02-28-18-32-33.png)

3. **バケットの作成** ダイアログで **バケット名** に任意の名前を入力します。

4. その他の設定項目はデフォルトのまま **作成** をクリックします。

    ![](2022-02-28-18-34-19.png)

<BR>

<a id="anchor5"></a>

# 2. ソース・データベースの設定

<a id="anchor6"></a>

## 2-1. ソース・データベースのCDBとPDBの接続情報の確認

1. OCIコンソール・メニューから **Oracle Database** → **ベア・メタル、VMおよびExadata** を選択します。

    ![](2022-02-28-18-37-12.png)

2. ソース・データベース（前提条件で作成したBaseDB）のDBシステム名をクリックします。

    ![](2022-02-28-18-39-33.png)

3. ソース・データベースのデータベース名をクリックします。

    ![](2022-02-28-18-41-21.png)

4. **DB接続** をクリックします。

    ![](2022-02-28-18-47-26.png)

    <a id="anchor21"></a>

5. **簡易接続** の接続文字列の右にある **コピー** をクリックし、メモ帳に貼り付けます。（後の手順で使用するため）

6. ダイアログを閉じます。

    ![](2022-02-28-18-46-49.png)

7. **リソース** の一覧から **プラガブル・データベース** をクリックします。

    ![](2022-02-28-18-49-14.png)

8. ソース・データベースのプラガブル・データベース名をクリックします。

    ![](2022-02-28-18-50-11.png)

9. **PDB接続** をクリックします。

    ![](2022-02-28-18-51-19.png)

    <a id="anchor22"></a>

10. **簡易接続** の接続文字列の右にあるコピーをクリックし、メモ帳に貼り付けます。（後の手順で使用するため）

11. ダイアログを閉じます。

    ![](2022-02-28-18-53-09.png)

12. **DBシステムの詳細** をクリックします。

    ![](2022-02-28-18-54-11.png)

13. **リソース** の一覧から **ノード** をクリックします。

    ![](2022-03-10-17-44-46.png)

14. ソース・データベースの**パブリックIPアドレス** と **プライベートIPアドレス** をそれぞれコピーし、メモ帳に貼り付けます。（後の手順で使用するため）

    ![](2022-02-28-18-57-32.png)

<a id="anchor7"></a>

## 2-2. データ・ポンプ・エクスポートで使用されるディレクトリの作成

移行の実行の際、**初期ロードオプション** として **オブジェクト・ストレージ経由のデータポンプ** を選択する場合、データ・ポンプは、エクスポートされたデータベースをオブジェクト・ストレージ・バケットに一時的に格納します。そこで使用するディレクトリを作成します。

1. ソース・データベースのDBシステムに対して、Tera Termなどのsshクライアントで接続します。

2. ユーザー・ボリュームに新しいディレクトリを作成します。

    ```
    sudo su - oracle
    ```

    ```
    mkdir /u01/app/oracle/dumpdir
    ```

<a id="anchor8"></a>

## 2-3. 初期化パラメータSTREAMS_POOL_SIZEの設定

 sysユーザーでSQL*Plusに接続し、STREAMS_POOL_SIZEを2GBに変更します。

```
sqlplus / as sysdba
```

```sql
ALTER SYSTEM SET STREAMS_POOL_SIZE=2G SCOPE=BOTH;
```

```
exit;
```

<BR>

<a id="anchor9"></a>

# 3. データベース接続の作成

<a id="anchor10"></a>

## 3-1. ソースCDBの接続の作成

1. OCIコンソール・メニューから **移行とディザスタ・リカバリ** → **データベース接続** を選択します。

    ![alt text](image.png)

2. **接続の作成** をクリックします。

    ![alt text](image-1.png)

3. **一般情報** の各項目は以下のように設定します。その他の入力項目はデフォルトのままにします。
    + **名前** - 任意
    + **タイプ** - Oracle Database
    + **ボールト** - 登録したいボールトを選択します。
    + **暗号化キー** - 登録したい暗号化キーを選択します。

    設定後、**Next** をクリックします。

    ![alt text](image-2.png)

4. **接続の詳細** の各項目は以下のように設定します。その他の入力項目はデフォルトのままにします。
    + **データベース詳細** - データベース情報の入力
    + **接続文字列** - [2-1.ソース・データベースのCDBとPDBの接続情報の確認の5.](#anchor21)でメモ帳にコピーした接続文字列のホスト名部分をデータベース・ノードのプライベートIPアドレスと入れ替えて入力します。例：10.0.0.28:1521/sourcedb_phx1rp.sub01270428300.vcndms.oraclevcn.com
    + **初期ロード・データベース・ユーザー名** - system
    + **初期ロード・データベース・パスワード** - ＜管理者パスワード＞
    + **サブネット** - データベースが配置されているサブネットを選択します。
    
    設定後、**作成** をクリックします。

    ![alt text](image-3.png)
    

<a id="anchor11"></a>

## 3-2. ソースPDBの接続の作成

1. OCIコンソール・メニューから **移行とディザスタ・リカバリ** → **データベース接続** を選択します。

    ![alt text](image.png)

2. **接続の作成** をクリックします。

    ![alt text](image-1.png)

3. **一般情報** の各項目は以下のように設定します。その他の入力項目はデフォルトのままにします。
    + **名前** - 任意
    + **タイプ** - Oracle Database
    + **ボールト** - 登録したいボールトを選択します。
    + **暗号化キー** - 登録したい暗号化キーを選択します。

    設定後、**Next** をクリックします。
    ![alt text](image-4.png)

4. **接続の詳細** の各項目は以下のように設定します。その他の入力項目はデフォルトのままにします。
    + **データベース詳細** - データベース情報の入力
    + **接続文字列** - 本チュートリアルの[2-1.ソース・データベースのCDBとPDBの接続情報の確認の10.](#anchor22)でメモ帳にコピーした接続文字列のホスト名部分をデータベース・ノードのプライベートIPアドレスと入れ替えて入力します。例：10.0.0.28:1521/pdb.sub01270428300.vcndms.oraclevcn.com
    + **初期ロード・データベース・ユーザー名** - system
    + **初期ロード・データベース・パスワード** - ＜管理者パスワード＞
    + **サブネット** - データベースが配置されているサブネットを選択します。

    設定後、**作成** をクリックします。

    ![alt text](image-5.png)

<a id="anchor12"></a>

## 3-3. ターゲットADBの接続の作成

1. OCIコンソール・メニューから **移行とディザスタ・リカバリ** → **データベース接続** を選択します。

    ![alt text](image.png)

2. **接続の作成** をクリックします。

    ![alt text](image-1.png)

3. **一般情報** の各項目は以下のように設定します。その他の入力項目はデフォルトのままにします。
    + **名前** - 任意
    + **タイプ** - Oracle Autonomous Database
    + **ボールト** - 登録したいボールトを選択します。
    + **暗号化キー** - 登録したい暗号化キーを選択します。

    設定後、**Next** をクリックします。

    ![alt text](image-6.png)

4. **接続の詳細** の各項目は以下のように設定します。その他の入力項目はデフォルトのままにします。
    + **データベース** - 登録したいADBを選択します。
    + **初期ロード・データベース・ユーザー名** - admin
    + **初期ロード・データベース・パスワード** - ＜管理者パスワード＞

    設定後、**作成** をクリックします。

    ![alt text](image-7.png)

以上で **DMSを使用したデータベース移行の前準備** は終了です。

オフライン移行を実行したい場合は、[305 : OCI Database Migration Serviceを使用したデータベースのオフライン移行](../adb305-database-migration-offline)にお進みください。
オンライン移行を実行したい場合は、[306 : OCI Database Migration Serviceを使用したデータベースのオンライン移行](../adb306-database-migration-online)にお進みください。

<BR>

<a id="anchor13"></a>

# 参考資料
+ [Oracle Cloud Infrastructure Database移行サービスの使用](https://docs.oracle.com/cd/E83857_01/paas/database-migration/dmsus/index.html)
+ [OCI Database Migration Workshop](https://apexapps.oracle.com/pls/apex/dbpm/r/livelabs/view-workshop?wid=856)

<BR>

[ページトップへ戻る](#anchor0)