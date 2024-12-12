# Tree structure in SQL

開發應用程式時，樹狀結構的資料其實很常見，只是當需要將這些資料用 SQL 資料庫儲存下來時，就會發現問題忽然複雜了許多；能保存資料間的階層關連性還不夠，還必須考慮到各類查詢和修改的效能。前陣子讀了 [SQL Antipatterns: Avoiding the Pitfalls of Database Programming](https://pragprog.com/book/bksqla/sql-antipatterns)；在這本書裡面，作者就儲存樹狀結構的幾種 pattern 做了介紹，並比較每個方式的優缺點。我覺得這是很實用的技巧，所以這篇文章就以書中的資料為主，來介紹不同的實作方式。

以下的範例都是以書中的例子改寫成 MySQL 的程式 (不包括 common table expression，因為 MySQL 不支援)，並且附上建立資料的 query，方便自己實驗時使用。例圖部份是以書中的圖修改的。

[migration.sql](https://blog.rangilin.idv.tw/tree-structure-in-sql/migration.sql)

## 前置

假設現在要做 bug-tracking 軟體的評論功能，使用者除了可以在 bug 內發表評論外，還可以回應特定的評論。可能會像這個樣子：

```
Fran: 為什麼會有這個 bug ?
├─ Ollie: 我想應該是 null pointer。
│  └─ Fran: 不是，我確認過了。
│
└─ Kukla: 應該要去檢查錯誤資料。
   ├─ Ollie: 對，這是個 bug。
   │
   └─ Fran: 請加上個檢查。
      └─ Kukla: 加上去後就好了。
```

在不考慮階層關係的情況下，我們可以用這樣的一張表把每個評論記下來。

```sql
CREATE TABLE comments (
  comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  bug_id        BIGINT UNSIGNED NOT NULL,
  author        TEXT NOT NULL,
  comment_text  TEXT NOT NULL,
);
```

資料會長這個樣子:

```
comment_id  bug_id  author  comment_text
--------------------------------------------------
1           1234    Fran    為什麼會有這個 bug ?
2           1234    Ollie   我想應該是 null pointer。
3           1234    Fran    不是，我確認過了。
4           1234    Kukla   應該要去檢查錯誤資料。
5           1234    Ollie   對，這是個 bug。
6           1234    Fran    請加上個檢查。
7           1234    Kukla   加上去後就好了。
```

## Adjacency List

這個 pattern 應該是最常見也最容易實作與理解的方式：藉由讓每一筆 comment 記住自己的 parent 來維持住階層的關係。在此增加了 `parent_id` 這個欄位。

```sql
CREATE TABLE comments (
  comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  bug_id        BIGINT UNSIGNED NOT NULL,
  parent_id     BIGINT UNSIGNED,                -- 新欄位
  author        TEXT NOT NULL,
  comment_text  TEXT NOT NULL,

  FOREIGN KEY (parent_id) REFERENCES comments(comment_id) ON DELETE CASCADE
);
```

而資料看起來會長這個樣子:

```
comment_id  parent_id   author  comment
---------------------------------------------------------
1           NULL        Fran    為什麼會有這個 bug ?
2           1           Ollie   我想應該是 null pointer。
3           2           Fran    不是，我確認過了。
4           1           Kukla   應該要去檢查錯誤資料。
5           4           Ollie   對，這是個 bug。
6           4           Fran    請加上個檢查。
7           6           Kukla   加上去後就好了。
```

### 查詢

我們可以用 Join 靠一次查詢拿到下一階的資料

```sql
SELECT c1.*, c2.* FROM comments c1
  LEFT OUTER JOIN comments c2 ON c2.parent_id = c1.comment_id
WHERE c1.comment_id = 1;
```

可是想要一口氣查一階以上時，就得再 Join 下去

```sql
SELECT c1.*, c2.*, c3.*, c4.*
FROM comments c1                        -- 第一階
  LEFT OUTER JOIN comments c2
    ON c2.parent_id = c1.comment_id     -- 第二階
  LEFT OUTER JOIN comments c3
    ON c3.parent_id = c2.comment_id     -- 第三階
  LEFT OUTER JOIN comments c4
    ON c4.parent_id = c3.comment_id     -- 第四階
WHERE c1.comment_id = 1;
```

而且這種方式在階層數不固定時，就還要再想辦法動態產生 SQL。不然就是得要分次查詢。

部份資料庫如 PostgreSQL 有支援 Common Table Expression，可以使用 WITH 語法做到遞迴查詢，達成一次呼叫的目的。

```sql
WITH comment_tree
(comment_id, bug_id, parent_id, author, comment_text, depth)
AS (
    SELECT *, 0 AS depth FROM comments
    WHERE parent_id IS NULL
    UNION ALL
    SELECT c.*, ct.depth+1 AS depth FROM comment_tree ct
    JOIN comments c ON (ct.comment_id = c.parent_id)
)
SELECT * FROM comment_tree WHERE bug_id = 1234 WHERE commend_id = 1;
```

除此之外，一口氣將所有資料讀出來，組成樹狀資料結構後放在 cache 提供查詢也是一種妥協的方式。只是這個方法在資料量大，或者是資料常修改次數高的時候能帶來的效益也有限。

### 修改

雖然查詢很糟，可是修改樹狀結構的時候倒是十分容易。

增加一筆 comment:

```sql
INSERT INTO comments (bug_id, parent_id, author, comment_text)
  VALUES (1234, 7, 'Kukla', 'Thanks!');
```

搬移:

```sql
UPDATE comments SET parent_id = 3 WHERE comment_id = 6;
```

如果有針對外來鍵設定 `ON DELETE CASCADE` 的話，刪除也是一次做完:

```sql
DELETE FROM comments WHERE comment_id = 4;
```

### 小結

優點:
1. 實作容易，易於理解。
2. 大部份時候修改資料結構都很單純。

缺點:
1. 針對部份樹狀結構做查詢時效能不佳，實作也較麻煩。

## Path Enumeration (Materialized Path)

所謂的 Path Enumeration，指的就是將整個樹的路徑列舉出來，舉例來講，像檔案系統路徑 `/usr/share/applications` 就是一種 Path Enumeration。

在 Adjacency List 中，每一筆 comment 只知道自己上一層的資料，因此在查詢時會花費額外的功夫。Path Enumeration 透過記錄完整路徑的方式，來解決 Adjacency Lists 難以處理的狀況。表格結構上，增加了 path 這個欄位。

```sql
CREATE TABLE comments (
  comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  path          VARCHAR(1000),                              -- 新欄位
  bug_id        BIGINT UNSIGNED NOT NULL,
  author        TEXT NOT NULL,
  comment_text  TEXT NOT NULL
);
```

資料:

```
comment_id  path        author  comment
---------------------------------------------------------------------
1           1/          Fran    為什麼會有這個 bug ?
2           1/2/        Ollie   我想應該是 null pointer。
3           1/2/3/      Fran    不是，我確認過了。
4           1/4/        Kukla   應該要去檢查錯誤資料。
5           1/4/5/      Ollie   對，這是個 bug。
6           1/4/6/      Fran    請加上個檢查。
7           1/4/6/7/    Kukla   加上去後就好了。
```

### 查詢

靠著 `path` 的資料，我們可以很輕易地查詢子樹狀結構：

comment 7 的所有父階：

```sql
SELECT * FROM comments AS c WHERE '1/4/6/7/' LIKE CONCAT(c.path, '%');
```

這個查詢會將 `1/`, `1/4/`, `1/4/6/` 三個父階以及自己全部查詢出來。

comment 4 的所有子階：

```sql
SELECT * FROM comments AS c WHERE c.path LIKE '1/4/%';
```

這個查詢會將 `1/4/5/`, `1/4/6/`, `1/4/6/7/` 三個子階及自己全部查詢出來。

由於查詢子樹狀結構變得十分簡單，所以套用 aggregation function 也很直接：

```sql
SELECT COUNT(*), author FROM comments AS c WHERE c.path LIKE '1/4/%' GROUP BY c.author;
```

得到

```
COUNT(*)    author
------------------
1           Fran
2           Kukla
1           Ollie
```

### 修改

為了要維護路徑，修改資料時就麻煩了。例如在 comment 7 下一層新增一筆評論：

```sql
-- 先新增資料
INSERT INTO comments (author, bug_id, comment_text) VALUES ('Ollie', 1234, 'Good job!');
-- 再更新路徑
UPDATE comments AS c, (SELECT path FROM comments WHERE comment_id = 7) AS c2
  SET c.path = CONCAT(c2.path, LAST_INSERT_ID(), '/')
WHERE comment_id = LAST_INSERT_ID();
```

搬移的話必須要將所有子階的路徑全部 update，由於每一個路徑要更新的值都不一樣，因此需要用到 CASE..WHEN 語法來動態決定更新的值。例如將 comment 4 搬到 comment 2 之下:

```sql
UPDATE comments SET path = CASE WHEN path LIKE '1/4/%' THEN REPLACE(path, '1/', '1/2/') END WHERE path LIKE '1/4/%';
```

刪除的時候因為查詢方便，所以不是問題：

```sql
DELETE FROM comments WHERE path LIKE '1/4/%';
```

### 小結

優點:
1. 實作容易，易於理解。
2. 查詢容易。

缺點:
1. 依賴 LIKE 語法，索引帶來的效能改善受限於語法的使用方式。
2. 修改資料時需要維護所有子階的路徑資料。
3. 路徑根據資料庫、primary key 長度等條件，可能會遇到長度限制。
4. 路徑是自己編碼過的資料，沒有辦法用外來鍵確保資料完整性。

## Nested Sets

Nested Sets 是一種藉由記錄樹的 [Pre-Order Tree Traversal](http://en.wikipedia.org/wiki/Tree_traversal#Pre-order) 順序來達到記錄階層結構的方法。這個方式需要增加以下的欄位：

```sql
CREATE TABLE comments (
  comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  nsleft        INTEGER NOT NULL,                           -- 新欄位
  nsright       INTEGER NOT NULL,                           -- 新欄位
  bug_id        BIGINT UNSIGNED NOT NULL,
  author        TEXT NOT NULL,
  comment_text  TEXT NOT NULL
);
```

`nsleft`, `nsright` 指的是在 Pre-Order Tree Traversal 時，第一次和第二次經過此節點的步數。第一次經過時將步字寫在節點左邊，第二次經過時寫在右邊，最後就會形成如下圖所示的樣子。

因此資料看起來會是這個樣子：

```
comment_id  nsleft  nsright author  comment_text
------------------------------------------------
1           1       14      Fran    為什麼會有這個 bug ?
2           2       5       Ollie   我想應該是 null pointer。
3           3       4       Fran    不是，我確認過了。
4           6       13      Kukla   應該要去檢查錯誤資料。
5           7       8       Ollie   對，這是個 bug。
6           9       12      Fran    請加上個檢查。
7           10      11      Kukla   加上去後就好了。
```

### 查詢

有了節點左右方的數字後，就可以利用這些數字查詢子樹結構:

```sql
-- comment 4 以及其下層的資料
SELECT c2.* FROM comments AS c1 JOIN comments as c2 ON c2.nsleft BETWEEN c1.nsleft AND c1.nsright WHERE c1.comment_id = 4;

-- 如果已經有 comment 4 的 nsleft/nsright 資料，query 可以簡化成
SELECT * FROM comments WHERE nsleft BETWEEN 6 AND 13;
```

查詢所有父階資料也沒問題:

```sql
-- comment 6 以及其上層的資料
SELECT c2.* FROM comments AS c1 JOIN comments AS c2 ON c1.nsleft BETWEEN c2.nsleft AND c2.nsright WHERE c1.comment_id = 6;

-- 如果已經有 comment 6 的 nsleft/nsright 資料，query 可以簡化成
SELECT * FROM comments WHERE