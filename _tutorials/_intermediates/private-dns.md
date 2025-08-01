---
title: "プライベートDNSを使って名前解決をする"
excerpt: "インスタンスなどに独自の名前をつけることで一目で分かりやすい名前解決を行えます。"
order: "055"
tags:
  - intermediate
  - network
header:
  teaser: "/intermediates/private-dns/04.png"
  overlay_image: "/intermediates/private-dns/title.png"
  overlay_filter: rgba(34, 66, 55, 0.7)
#link: https://community.oracle.com/tech/welcome/discussion/4474277/%E3%82%B7%E3%83%AA%E3%82%A2%E3%83%AB-%E3%82%B3%E3%83%B3%E3%82%BD%E3%83%BC%E3%83%AB%E3%81%A7ssh%E3%81%A7%E3%81%8D%E3%81%AA%E3%81%84%E3%82%A4%E3%83%B3%E3%82%B9%E3%82%BF%E3%83%B3%E3%82%B9%E3%81%AE%E3%83%88%E3%83%A9%E3%83%96%E3%83%AB%E3%82%B7%E3%83%A5%E3%83%BC%E3%83%88%E3%82%92%E3%81%99%E3%82%8B-oracle-cloud-infrastructure%E3%82%A2%E3%83%89%E3%83%90%E3%83%B3%E3%82%B9%E3%83%89
---

**チュートリアル一覧に戻る :** [Oracle Cloud Infrastructure チュートリアル](../..)
<br>

OCIではプライベートDNSが使用可能で独自のプライベートDNSドメイン名を使用し、関連付けられたゾーンおよびレコードを管理して、VCN内またはVCN間で実行されているアプリケーションなどのホスト名で名前解決を行えます。

プライベートDNSのゾーン、クエリ、およびリゾルバーエンドポイントは無料です。

※無料トライアル環境ではプライベートDNSビューの作成はできないのでご注意ください。

**所要時間 :** 約30分

**前提条件 :**
   1. Oracle Cloud Infrastructure で、作成済みのVCNおよびLinuxインスタンスがあること

**注意 :** チュートリアル内の画面ショットについては Oracle Cloud Infrastructure の現在のコンソール画面と異なっている場合があります


<br><br>

<a id="anchor0"></a>

# 1. PrivateDNSとは？
## 概要
プライベートDNSはVCNの名前解決を行います。

VCNを作る際にDHCPオプションをオンにしていると、デフォルトでプライベートビューとプライベートDNSゾーンがあり、インスタンスを作成したときに「`*.oraclevcn.com`」というFQDNが割り当てられています。

![画面ショット01](01.png)

DNSリゾルバは問い合わせを受けるとビュー、ゾーン、転送ルール、インターネットDNSの順に応答を返します。

今回のチュートリアルでは新たにDNSゾーンを作成し、「`aoyama.com`」という任意のドメイン名を付けて名前解決を行います。

![画面ショット02](02.png)

## 用語説明

- DNSゾーンレコード : DNSに定義するレコード。AレコードにはIPアドレスとホスト名の関連付けを定義する。Aレコード以外も定義可能。

- プライベートDNSゾーン : hogehoge.comなど、ユーザーが定義した任意のドメイン名に属するレコードの集合。VCN内のプライベートIPなどVCN内のリソースを定義できる。

- プライベートDNSビュー : プライベートDNSゾーン情報をまとめたもの。

- プライベートDNSリゾルバ : DNSクエリに対して名前解決を行う仕組み。デフォルトではVCNのプライベートDNSビューだけだが、他の同リージョンのプライベートDNSビュー、他リージョンのプライベートDNSやオンプレミスのDNSと連携できる。

## ポリシーの付与

必要なIAMポリシー
管理者グループ内のユーザーの場合、必要な権限を持っています。ユーザーが管理者グループに属していない場合、次のようなポリシーにより、特定のグループがプライベートDNSを管理できます。

```
Allow group <GroupName> to manage dns in tenancy where target.dns.scope = 'private'
```

<br>

<a id="anchor1"></a>

# 2. 同一リージョン・同一VCNでの名前解決

## 環境構築

まず下記のように同一リージョン・同一VCN内にインスタンスを構成します。インスタンスの名前は自由につけてもらって構いません。

![画面ショット03](03.png)

この章では赤線で囲った**aoyama.com**という名前の独自のプライベートDNSゾーンを作成してDNSレコードを追加します。

![画面ショット04](04.png)

## DNSゾーンの追加

1. コンソールメニューから [**ネットワーキング**] → [**DNS管理**] → [**ゾーン**]

1. [**プライベート・ゾーン**]タブを選択し、[**ゾーンの作成**]
    - ゾーン名`aoyama.com`
    - プライベートビュー: Tokyo_VCNという名前のプライベートビューに`aoyama.com`のゾーンを追加

![画面ショット05](05.png)

<br>

## DNSレコードの追加

1. 新しく作成した[**aoyama.comゾーン**] → [レコードの追加]を押す
2. ここからAoyama_1_Instanceのレコードを追加していく。
    - レコード型 : A – Ipv4アドレス
    - 名前 : soccer
    - TTL : 30
    - RDATAモード : 基本
    - ADDRESS : Aoyama_1_Instanceのプライベートアドレス ![画面ショット06](06.png)
3. 同様にAoyama_2_Instanceについてもレコードを追加する
4. 合計二つのプライベートビューとその中にAレコードのゾーン情報が入っていることを確認した後、[**変更の公開**]を押してゾーン情報を更新

## Aoyama_1_instanceでFQDNを確認する

以下のコマンドを打ちAoyama_1_instanceのホスト名とIPアドレスを確認します。

```
cat /etc/hosts
```

続いて以下のコマンドを打ちデフォルトのゾーン情報と作成したゾーン情報は同じIPアドレスのAレコードが登録されていることを確認します。

```
host <FQDN名 or Aレコードに追加した名前>
```



![画面ショット7](07.png)

## Aoyama_2_instanceからAoyama_1_InstanceへFQDNで名前解決をする

nslookupで名前解決が行われていることを確認します。

```
nslookup <Aoyama_1_Instanceの作成したAレコード名>
```

![画面ショット8](08.png)

<br>
以上で、同一リージョン・同一VCNでの名前解決は完了です。

<br>

> ***Note サーバー名だけで名前解決をする場合***
> 
> 検索ドメインはデフォルトのVCNドメイン名しかないため、nslookupなどでsoccerというサーバー名だけで名前解決を行えません。そのため検索ドメインにaoyama.comというドメインを追加する必要があります。ただし、DHCPオプションのセットに指定できる検索ドメインは1つのみです。

1. [**VCN**] → [**DHCPオプション**] → デフォルトのDHCPオプションを[**編集**]

1. カスタム検索ドメインを選択し、`aoyama.com`のゾーン情報を追加して[**変更の保存**]

1. **インスタンスを再起動**して設定を反映させます。

![画面ショット09](09.png)

![画面ショット10](10.png)

DNSリゾルバの設定は/etc/resolv.confにあり、検索ドメインに「`aoyama.com`」が追加されていることを確認します。

```
cat /etc/resolv.conf
```

![画面ショット11](11.png)

Aoyama_2_instanceからAoyama_1_Instanceのサーバー名で名前解決を行う。

```
$ nslookup soccer
Server:         169.254.169.254
Address:        169.254.169.254#53

Non-authoritative answer:
Name:   soccer.aoyama.com
Address: 10.0.0.64
```



サーバー名だけで名前解決を行えました。

<br>

<a id="anchor2"></a>

# 3. 同一リージョン・異なるVCNでの名前解決

## 環境構築
続いて下記のように新しくVCNとインスタンスを作成した後、VCN同士をローカルピアリングゲートウェイで接続します。ここでは**Akasaka_VCN**を新しく作りインスタンスを作成後、**akasaka.com**という名前のプライベートゾーンを作成し、インスタンスのゾーン情報を追加します。

![画面ショット13](13.png)

![画面ショット14](14.png)

AoyamaVCNのプライベートDNSリゾルバは自身のプライベートビューだけを参照しているため、新しく作ったAkasaka_VCNの赤線で囲ったプライベートビューを参照する設定を行います。

![画面ショット15](15.png)

## DNSリゾルバの設定

VCNの詳細から[**DNSリゾルバ**]→デフォルト・プライベートビューを確認後、[**関連付けられたプライベートビューの管理**]から相手のプライベートビューを追加します。

![画面ショット16](16.png)

同様に、もう一つのDNSリゾルバに対しても**同じ作業**を行い、両方のDNSリゾルバがお互いのプライベートビューを参照していることを確認してください。



## 疎通確認
 
 デフォルトのセキュリティ・ルールではpingが許可されてないため、それぞれのサブネットのセキュリティリストに以下のICMPタイプ8のIngressルールを追加しておきます。

|Source|Protocol|Type|
|:-----|:----:|-----:|
|10.0.0.0/16|ICMP|8|
|10.1.0.0/16|ICMP|8|

Aoyama_1_instanceからakasaka_1へping疎通確認をします。

ここでpingが通らない場合、ローカルピアリングを見直して疎通を確認してください。

```
ping <相手のプライベートIPアドレス> -c 3
```

![画面ショット18](18.png)

## 異なるVCNにあるインスタンスの名前解決

疎通を確認した後、お互いのインスタンスについて名前解決を行ってください。

Aoyama_1_instanceからAkasaka_1へnslookup

```
$ nslookup rugby.akasaka.com
Server:         169.254.169.254
Address:        169.254.169.254#53

Non-authoritative answer:
Name:   rugby.akasaka.com
Address: 10.1.0.107
```
<br>
Akasaka_1 からAoyama_1_instanceへnslookup

```
$ nslookup soccer.aoyama.com
Server:         169.254.169.254
Address:        169.254.169.254#53

Non-authoritative answer:
Name:   soccer.aoyama.com
Address: 10.0.0.64

```

DNSリゾルバは相手のプライベートビューを参照して名前解決を行えました。

<br>

以上で、この章の作業は終了です。

<br>

<!--
<a id="anchor3"></a>

# 3. 異なるリージョン・異なるVCNでの名前解決


<br>

<a id="anchor4"></a>

# 4. 

<br>

<a id="anchor5"></a>

# 5. 

<br> -->

**チュートリアル一覧に戻る :** [Oracle Cloud Infrastructure チュートリアル](../..)

