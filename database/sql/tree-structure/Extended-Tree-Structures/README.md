# Extended Tree Structures in SQL

除了基本的樹狀結構實作方式外，還有一些其他常見的實作方法值得探討。本文將介紹幾種進階的樹狀結構實作方式，並說明它們各自的使用場景。

## Hybrid Adjacency List

這是 Adjacency List 的改良版本，通過增加額外的欄位來優化查詢效能。

```sql
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    parent_id     BIGINT UNSIGNED,
    root_id       BIGINT UNSIGNED,           -- 新增：紀錄根節點
    depth         INT NOT NULL DEFAULT 0,    -- 新增：紀錄深度
    path_order    VARCHAR(1000),             -- 新增：排序用路徑
    bug_id        BIGINT UNSIGNED NOT NULL,
    author        TEXT NOT NULL,
    comment_text  TEXT NOT NULL,
    
    FOREIGN KEY (parent_id) REFERENCES comments(comment_id),
    FOREIGN KEY (root_id) REFERENCES comments(comment_id)
);
```

資料範例：
```
comment_id  parent_id   root_id  depth  path_order   author  comment
--------------------------------------------------------------------------------
1           NULL        1        0      0001         Fran    為什麼會有這個 bug ?
2           1           1        1      0001.0001    Ollie   我想應該是 null pointer。
3           2           1        2      0001.0001.0001 Fran  不是，我確認過了。
4           1           1        1      0001.0002    Kukla   應該要去檢查錯誤資料。
```

### 優點

1. 保持了 Adjacency List 的簡單性
2. 通過 root_id 快速查詢整個討論串
3. 通過 depth 控制顯示層級
4. path_order 支援自然排序

### 查詢範例

```sql
-- 取得某個討論串的所有回覆
SELECT * FROM comments 
WHERE root_id = 1 
ORDER BY path_order;

-- 取得特定層級的回覆
SELECT * FROM comments 
WHERE root_id = 1 
AND depth = 2;

-- 取得直接回覆
SELECT * FROM comments 
WHERE parent_id = 1 
ORDER BY path_order;
```

## Materialized Path with Array

這是 Path Enumeration 的變體，使用陣列型態來儲存路徑。此方法在 PostgreSQL 特別有用，因為它支援陣列型態。

```sql
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    path_array    INTEGER[],                  -- 使用陣列儲存路徑
    bug_id        BIGINT UNSIGNED NOT NULL,
    author        TEXT NOT NULL,
    comment_text  TEXT NOT NULL
);
```

資料範例（PostgreSQL 語法）：
```
comment_id  path_array    author  comment
------------------------------------------------------------
1           {1}           Fran    為什麼會有這個 bug ?
2           {1,2}         Ollie   我想應該是 null pointer。
3           {1,2,3}       Fran    不是，我確認過了。
4           {1,4}         Kukla   應該要去檢查錯誤資料。
```

### 優點

1. 容易理解和操作陣列結構
2. 支援原生陣列操作和函數
3. 可以直接比較路徑層級

### 查詢範例（PostgreSQL）

```sql
-- 取得子樹
SELECT * FROM comments 
WHERE path_array[1] = 1 
ORDER BY path_array;

-- 取得特定深度的節點
SELECT * FROM comments 
WHERE array_length(path_array, 1) = 2;

-- 取得父路徑
SELECT * FROM comments 
WHERE path_array = (
    SELECT path_array[1:array_length(path_array,1)-1]
    FROM comments 
    WHERE comment_id = 3
);
```

## Temporal Adjacency List

這種方法特別適合需要處理版本歷史或時間序的樹狀結構。

```sql
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    parent_id     BIGINT UNSIGNED,
    valid_from    TIMESTAMP NOT NULL,
    valid_to      TIMESTAMP,
    bug_id        BIGINT UNSIGNED NOT NULL,
    author        TEXT NOT NULL,
    comment_text  TEXT NOT NULL,
    version       INT NOT NULL DEFAULT 1,
    
    FOREIGN KEY (parent_id) REFERENCES comments(comment_id)
);
```

資料範例：
```
comment_id  parent_id   valid_from    valid_to      version  author  comment
------------------------------------------------------------------------------------
1           NULL        2023-01-01    NULL          1        Fran    為什麼會有這個 bug ?
2           1           2023-01-01    2023-01-02    1        Ollie   我想應該是 null pointer。
2           1           2023-01-02    NULL          2        Ollie   [編輯] 確認是 null pointer。
3           2           2023-01-02    NULL          1        Fran    不是，我確認過了。
```

### 優點

1. 支援歷史版本追蹤
2. 可以查詢特定時間點的樹狀結構
3. 適合需要稽核追蹤的系統

### 查詢範例

```sql
-- 取得當前有效的樹狀結構
SELECT * FROM comments 
WHERE valid_to IS NULL 
START WITH parent_id IS NULL 
CONNECT BY PRIOR comment_id = parent_id;

-- 取得特定時間點的樹狀結構
SELECT * FROM comments 
WHERE '2023-01-01' BETWEEN valid_from AND COALESCE(valid_to, '9999-12-31');

-- 取得評論的修改歷史
SELECT * FROM comments 
WHERE comment_id = 2 
ORDER BY version;
```

## Nested Intervals

這是一種較少見但很有趣的實作方式，使用數學原理來維護樹狀結構。

```sql
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    numerator     BIGINT NOT NULL,           -- 分子
    denominator   BIGINT NOT NULL,           -- 分母
    bug_id        BIGINT UNSIGNED NOT NULL,
    author        TEXT NOT NULL,
    comment_text  TEXT NOT NULL,
    
    UNIQUE(numerator, denominator)
);
```

資料範例：
```
comment_id  numerator  denominator  author  comment
----------------------------------------------------------------
1           1          1            Fran    為什麼會有這個 bug ?
2           2          3            Ollie   我想應該是 null pointer。
3           3          5            Fran    不是，我確認過了。
4           3          2            Kukla   應該要去檢查錯誤資料。
```

### 原理說明

- 每個節點都用分數表示（numerator/denominator）
- 子節點的分數值必須在父節點的分數區間內
- 使用連分數理論來產生新節點的值

### 優點

1. 不需要更新現有節點
2. 支援無限層級
3. 數學運算快速

### 查詢範例

```sql
-- 取得子樹（數學區間查詢）
SELECT * FROM comments c1
WHERE c1.numerator/c1.denominator > 1/1 
AND c1.numerator/c1.denominator < 2/1
ORDER BY c1.numerator/c1.denominator;

-- 判斷父子關係
SELECT * FROM comments c1, comments c2
WHERE c2.numerator/c2.denominator > c1.numerator/c1.denominator
AND c2.numerator/c2.denominator < (c1.numerator+1)/(c1.denominator);
```

## 使用建議

1. **Hybrid Adjacency List**
   - 適合一般的討論串系統
   - 需要簡單的層級控制
   - 重視查詢效能但不想太複雜

2. **Materialized Path with Array**
   - 使用支援陣列型態的資料庫
   - 需要靈活的路徑操作
   - 路徑長度變化不大

3. **Temporal Adjacency List**
   - 需要版本控制
   - 需要稽核追蹤
   - 有時間維度的應用

4. **Nested Intervals**
   - 極少更動的樹狀結構
   - 需要支援無限層級
   - 重視數學運算效能

## 效能考量

以下是各種方法在不同操作下的效能比較：

| 操作 | Hybrid Adjacency | Path Array | Temporal | Nested Intervals |
|------|-----------------|------------|----------|------------------|
| 讀取子樹 | O(1) | O(1) | O(log n) | O(1) |
| 插入節點 | O(1) | O(n) | O(1) | O(1) |
| 移動子樹 | O(n) | O(n) | O(n) | O(1) |
| 刪除節點 | O(n) | O(n) | O(1)* | O(1) |

*: 使用軟刪除（soft delete）時

## 總結

每種方法都有其特定的使用場景：

1. **一般應用**：使用 Hybrid Adjacency List
2. **需要版本控制**：使用 Temporal Adjacency List
3. **需要強大路徑操作**：使用 Materialized Path with Array
4. **特殊數學需求**：使用 Nested Intervals

選擇適當的方法時，需要考慮：

- 資料的使用模式（讀多寫少？寫多讀少？）
- 樹的特性（深度？寬度？變動頻率？）
- 業務需求（版本控制？稽核追蹤？）
- 維護複雜度