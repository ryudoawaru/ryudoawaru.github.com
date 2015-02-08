---
layout: post
title: "PostgreSQL Materialized View簡介"
date: 2014-06-20 15:54
comments: true
categories: postgresql, pgsql
---

最近在開發某個主力產品時遇到了一個問題，就是當我們在後台編輯一些資料，包含新增 / 修改 / 刪除
，然後在沒有完成所有編輯前不想讓修改中的資料立刻在前台生效，正在苦思如何完成這項功能時，發現了PostgreSQL有一個超好用的功能叫「Materialized View」，於是就開始survey看看。

## Database View

在[Wikipedia](http://en.wikipedia.org/wiki/View_%28SQL%29)中的說明為：
```
檢視表 (View) 是在關聯式資料庫中，將一組查詢指令構成的結果集，在實體資料表中的改變都可以立刻反應在檢視表中
```

通常我們會為了特定目的，例如將某些特定條件的查詢變成一張虛擬表之類的原因，來製作一些View，例如：

{% img /assets/2014/06/20/use-materizlied-view-in-pgsql/ex-order-erd.png %}

我們會用以下的方式建立一個View：

```sql

CREATE VIEW orders_products_users AS
SELECT 
orders.user_id,
users.name,
orders.product_id,
products.name
FROM
orders JOIN users ON users.id = orders.user_id, 
JOIN products ON products.id = orders.product_id

```

以後我們要使用這個view時只要：
```sql
SELECT * FROM orders_products_users;
```

即可

但是普通的view本身並沒有真正的儲存資料，預設情況下它只是一個查詢的捷徑而已，而且也不能透過update view來更新資料，所以對於使用Rails等ORM的人來說，即使已經有一些「把view當model」的Gem（例如[activerecord-database-views](https://github.com/stevo/activerecord-database-views)）推出，還是會覺得是一個比較雞肋的功能。


##Materialized View（以下簡稱M-View）
今天要介紹的這個功能和普通的View的差別，就在建立出來的view是一個查詢狀態的「快照」（Snapshot），在快照建立後，只要不更新（Refresh），即使對原本的table有任何的變更，這個M-View裡的資料也不會有任何變化，因此就可以做到前面所說的，讓後台更新的資料不影響到前台的功能，接下來讓我們來看看如何使用這個功能。


##Postgresql的Materialized View
這個功能原本是出自於Oracle，號稱「開源的Oracle」的PostgreSQL也在本文撰時最新的穩定版本9.3加入了該功能，[參考資料](http://www.postgresql.org/docs/9.3/static/sql-creatematerializedview.html)。

###建立Materialized View

```sql
CREATE MATERIALIZED VIEW table_name
    AS query
    [ WITH [ NO ] DATA ]
```
例如：
```sql
CREATE MATERIALIZED VIEW mymatview AS SELECT * FROM mytab;
```
以上指令會以「SELECT * FROM mytab」的查詢式建立一個名為「mymatview」的M-View，而且在refresh之前，mymatview的內容不會受到原本mytab更新的影響。

結尾的WITH DATA 和 WITH NO DATA的差別是在，會不會在建立時就把當時的資料填進去，如果WITH NO DATA的話，就必需要再使用refresh才可以把資料填進去。

###更新Materialized View
```sql
REFRESH MATERIALIZED VIEW  M-View的名稱
```
一旦下了這個指令後，就可以將現在的資料內容更新到指定的M-View上

###移除Materialized View
```sql
DROP MATERIALIZED VIEW  M-View的名稱 [CASCADE | RESTRICT]
```
顧名思義，即完全移除該M-View，CASCADE意即在移除此物件實「一併移除相關物件」。

###Materialized View的注意事項
1.	和普通View一樣，不可以用UPDATE或INSERT等更新內容
1.	如果建立了M-View後，要移除關聯的資料表時需要用CASADE方式一併移除相關M-View，否則會無法移除。
1.	可以多層重疊，意即可以透過「查詢M-View建立其它M-View」

##Materialized View的用途
1.	加速部份查詢
	和一般View不同，M-View佔有獨立表空間，所以查詢M-View不會影響到關聯的資料表，意即如果我們將一些複雜或費時的查詢的結果用M-View建立快照的話，可以透過查詢該M-View節省下一次查詢的時間，例如某個交易資料表如果有10年份1000萬筆的話，將某一年份的交易資料獨立查詢到一個M-View之後，要查詢該年份的交易資料只要查詢該M-View，就不需要在查詢中scan全部10年份的交易資料了。
2.	實現資料版本控制
	這是我原本想要使用的目的，這個功能可以玩的地方還很多，例如我們
	
##範例

今天假設有如下的表

```sql
CREATE TABLE spree_products
(
  id serial NOT NULL,
  name character varying(255) NOT NULL DEFAULT ''::character varying,
  description text,
  available_on timestamp without time zone,
  deleted_at timestamp without time zone,
  slug character varying(255),
  meta_description text,
  meta_keywords character varying(255),
  tax_category_id integer,
  shipping_category_id integer,
  created_at timestamp without time zone,
  updated_at timestamp without time zone,
  CONSTRAINT spree_products_pkey PRIMARY KEY (id)
)
```
然後我們建立如下的M-View
```sql
CREATE MATERIALIZED VIEW spree_products_mview AS 
 SELECT * FROM spree_products;
```
然後我們看現在的 spree_products表內容如下：
![](/assets/2014/06/20/use-materizlied-view-in-pgsql/mview-original.png)
此時table和M-View的資料應該是一致的，此時我們將資料表的一些資料刪除：
```sql
DELETE FROM spree_products WHERE id = 16;
```
此時id為16的紀錄已經被移除
![](/assets/2014/06/20/use-materizlied-view-in-pgsql/after-remove.png)
但是M-View的資料還是一樣的，如下圖
![](/assets/2014/06/20/use-materizlied-view-in-pgsql/mview-after-remove.png)
這時如果我們下了REFRESH的話：

```sql
REFRESH MATERIALIZED VIEW spree_products_mview;
```

M-View的資料就會和資料表上一樣了

以上是簡介Materizlied View在PostgreSQL的作用，接下來我們還會針對如何用Rails / ActiveRecord操作這個功能作出說明。