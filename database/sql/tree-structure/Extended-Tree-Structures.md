# Extended Tree Structures in SQL

## 前言

在理解了基本的樹狀結構實現方法後，我們可以進一步探討一些更進階的實作方式。這些方法通常結合了多種基礎方法的優點，或者針對特定使用場景進行了優化。本文將詳細介紹幾種進階的樹狀結構實現方法，並分析它們的優缺點和適用場景。

## 情境說明

讓我們延續之前的 bug 追蹤系統範例，但加入一些更進階的需求：

1. 需要快速存取任意節點的完整路徑
2. 支援大量的評論資料
3. 需要保留評論的修改歷史
4. 支援跨資料庫的分散式架構
5. 需要高效的樹狀結構操作

### 範例資料結構

```
Fran: 為什麼會有這個 bug？
├─ Ollie: 我想應該是 null pointer。
│  └─ Fran: 不是，我確認過了。
│     └─ Ollie: [已編輯] 確認是 memory leak。
└─ Kukla: 應該要去檢查錯誤資料。
   ├─ Fran: 請提供更多細節。
   └─ Ollie: 我來補充說明。
      └─ Kukla: 已更新相關文件。
```

## 1. Hybrid Adjacency List (混合鄰接表)模型

### 概念
這是 Adjacency List 的改良版本，透過增加額外的欄位來優化查詢效能。

### 資料結構
```sql
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    parent_id     BIGINT UNSIGNED,
    root_id       BIGINT UNSIGNED,           -- 指向根節點
    depth         INT NOT NULL DEFAULT 0,    -- 儲存深度
    path_order    VARCHAR(1000),             -- 排序用路徑
    bug_id        BIGINT UNSIGNED NOT NULL,
    author        TEXT NOT NULL,
    comment_text  TEXT NOT NULL,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (parent_id) REFERENCES comments(comment_id),
    FOREIGN KEY (root_id) REFERENCES comments(comment_id),
    INDEX(path_order)
);
```

### 預期資料範例
```
comment_id  parent_id   root_id  depth  path_order   author  comment_text
--------------------------------------------------------------------------------
1           NULL        1        0      0001         Fran    為什麼會有這個 bug？
2           1           1        1      0001.0001    Ollie   我想應該是 null pointer。
3           2           1        2      0001.0001.0001 Fran  不是，我確認過了。
4           1           1        1      0001.0002    Kukla   應該要去檢查錯誤資料。
5           4           1        2      0001.0002.0001 Fran  請提供更多細節。
```

### 查詢操作

1. 查詢特定評論的所有回覆：
```sql
SELECT * FROM comments 
WHERE root_id = 1 
AND depth > (SELECT depth FROM comments WHERE comment_id = 1)
ORDER BY path_order;
```

2. 查詢直接回覆：
```sql
SELECT * FROM comments 
WHERE parent_id = 1 
ORDER BY path_order;
```

3. 查詢特定層級的評論：
```sql
SELECT * FROM comments 
WHERE root_id = 1 
AND depth = 2;
```

### 修改操作

1. 新增評論：
```sql
INSERT INTO comments (
    parent_id, root_id, depth, path_order, 
    bug_id, author, comment_text
) 
SELECT 
    4,                              -- parent_id
    root_id,                        -- 繼承根節點
    depth + 1,                      -- 深度加一
    CONCAT(path_order, '.', LPAD(LAST_INSERT_ID(), 4, '0')), -- 構建排序路徑
    bug_id, 'Kukla', '已更新相關文件。'
FROM comments 
WHERE comment_id = 4;
```

2. 移動評論：
```sql
UPDATE comments c1, comments c2
SET 
    c1.parent_id = 2,
    c1.depth = c2.depth + 1,
    c1.path_order = CONCAT(c2.path_order, '.', LPAD(c1.comment_id, 4, '0'))
WHERE c1.comment_id = 4
AND c2.comment_id = 2;
```

### 主要困難點

1. **路徑維護**
   - path_order 需要定期重整理
   - 移動大量節點時效能較差
   - 需要處理路徑長度限制

2. **一致性維護**
   - 多個欄位需要同步更新
   - 需要確保 root_id 正確性
   - depth 計算需要額外處理

3. **效能瓶頸**
   - 大量移動操作時效能降低
   - 深層節點的路徑可能過長
   - 需要額外的索引維護成本

### 小結

優點：
1. 查詢效能優異
2. 支援多種查詢模式
3. 可以快速得到節點深度
4. 容易實現排序功能

缺點：
1. 需要維護多個欄位
2. 移動操作較複雜
3. 存儲空間較大
4. 需要定期維護路徑資料

## 2. Materialized Path with Array (陣列化路徑)模型

### 概念
這種方法使用陣列型態來儲存路徑資訊，特別適合在 PostgreSQL 這類支援陣列型態的資料庫中使用。

### 資料結構
```sql
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    path_array    INTEGER[],                  -- 使用陣列儲存路徑
    bug_id        BIGINT UNSIGNED NOT NULL,
    author        TEXT NOT NULL,
    comment_text  TEXT NOT NULL,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX(path_array)
);
```

### 預期資料範例
```
comment_id  path_array    author  comment_text
------------------------------------------------------------
1           {1}           Fran    為什麼會有這個 bug？
2           {1,2}         Ollie   我想應該是 null pointer。
3           {1,2,3}       Fran    不是，我確認過了。
4           {1,4}         Kukla   應該要去檢查錯誤資料。
5           {1,4,5}       Ollie   對，這是個 bug。
```

### 查詢操作

1. 查詢子樹：
```sql
SELECT * FROM comments 
WHERE path_array[1:array_length((
    SELECT path_array 
    FROM comments 
    WHERE comment_id = 1
), 1)] = (
    SELECT path_array 
    FROM comments 
    WHERE comment_id = 1
)
ORDER BY path_array;
```

2. 查詢特定層級：
```sql
SELECT * FROM comments 
WHERE array_length(path_array, 1) = 2;
```

3. 查詢祖先路徑：
```sql
SELECT * FROM comments 
WHERE comment_id = ANY(
    SELECT unnest(path_array) 
    FROM comments 
    WHERE comment_id = 3
);
```

### 修改操作

1. 新增節點：
```sql
INSERT INTO comments (path_array, bug_id, author, comment_text)
SELECT 
    array_append(path_array, nextval('comment_id_seq')),
    bug_id,
    'Kukla',
    '新的回覆'
FROM comments 
WHERE comment_id = 4;
```

2. 移動節點：
```sql
UPDATE comments 
SET path_array = (
    SELECT array_cat(
        (SELECT path_array FROM comments WHERE comment_id = 2),
        path_array[array_length(path_array,1):array_length(path_array,1)]
    )
)
WHERE path_array @> ARRAY[4];
```

### 主要困難點

1. **資料庫相容性**
   - 需要資料庫支援陣列型態
   - 不同資料庫實現方式不同
   - 可能需要自訂函數支援

2. **陣列操作複雜度**
   - 需要熟悉陣列操作語法
   - 複雜查詢效能較差
   - 索引策略需要特別設計

3. **資料維護**
   - 陣列更新較為複雜
   - 需要處理資料一致性
   - 可能需要定期重組陣列

### 小結

優點：
1. 資料結構直觀
2. 支援豐富的陣列操作
3. 容易實現樹遍歷
4. 適合靜態樹結構

缺點：
1. 資料庫支援限制
2. 複雜查詢效能問題
3. 維護成本較高
4. 不適合頻繁修改

## 3. Temporal Adjacency List (時序鄰接表)模型

### 概念
這種方法擴展了基本的鄰接表模型，加入了時間維度，特別適合需要追蹤歷史變更的場景。

### 資料結構
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
    
    FOREIGN KEY (parent_id) REFERENCES comments(comment_id),
    INDEX(valid_from),
    INDEX(valid_to)
);
```

### 預期資料範例
```
comment_id  parent_id   valid_from    valid_to      version  author  comment_text
------------------------------------------------------------------------------------
1           NULL        2024-01-01    NULL          1        Fran    為什麼會有這個 bug？
2           1           2024-01-01    2024-01-02    1        Ollie   我想應該是 null pointer。
2           1           2024-01-02    NULL          2        Ollie   [修改] 確認是 memory leak。
3           2           2024-01-02    NULL          1        Fran    不是，我確認過了。
```

### 查詢操作

1. 查詢當前有效的樹狀結構：
```sql
WITH RECURSIVE comment_tree AS (
    SELECT * FROM comments 
    WHERE parent_id IS NULL 
    AND valid_to IS NULL
    
    UNION ALL
    
    SELECT c.* FROM comments c
    INNER JOIN comment_tree t ON c.parent_id = t.comment_id
    WHERE c.valid_to IS NULL
)
SELECT * FROM comment_tree;
```

2. 查詢特定時間點的狀態：
```sql
SELECT * FROM comments 
WHERE '2024-01-01 12:00:00' BETWEEN valid_from 
AND COALESCE(valid_to, '9999-12-31');
```

3. 查詢修改歷史：
```sql
SELECT * FROM comments 
WHERE comment_id = 2 
ORDER BY version;
```

### 修改操作

1. 更新評論：
```sql
UPDATE comments 
SET valid_to = CURRENT_TIMESTAMP 
WHERE comment_id = 2 
AND valid_to IS NULL;

INSERT INTO comments (
    comment_id, parent_id, valid_from, 
    bug_id, author, comment_text, version
)
SELECT 
    comment_id, parent_id, CURRENT_TIMESTAMP,
    bug_id, author, '修改後的內容',
    version + 1
FROM comments 
WHERE comment_id = 2 
AND valid_to = CURRENT_TIMESTAMP;
```

2. 移動節點：
```sql
-- 結束當前版本
UPDATE comments 
SET valid_to = CURRENT_TIMESTAMP 
WHERE comment_id = 4 
AND valid_to IS NULL;

-- 建立新版本
INSERT INTO comments (
    comment_id, parent_id, valid_from,
    bug_id, author, comment_text, version
)
SELECT 
    comment_id, 2, CURRENT_TIMESTAMP,
    bug_id, author, comment_text,
    version + 1
FROM comments 
WHERE comment_id = 4 
AND valid_to = CURRENT_TIMESTAMP;
```

### 主要困難點

1. **歷史資料管理**
   - 資料量快速增長
   - 需要歷史資料清理策略
   - 查詢效能隨時間降低
   
2. **時間一致性**
   - 需要處理時區問題
   - 確保版本轉換的連續性
   - 處理並發修改衝突

3. **查詢複雜度**
   - 需要考慮時間維度
   - 效能優化較為困難
   - 索引設計更為複雜

### 小結

優點：
1. 完整的歷史追蹤能力
2. 支援時間點查詢
3. 資料更新透明化
4. 適合稽核追蹤需求

缺點：
1. 存儲空間需求大
2. 查詢較為複雜
3. 效能隨資料量增加而降低
4. 維護成本較高

## 4. Nested Intervals (嵌套區間)模型

### 概念
這是一種使用數學原理來管理樹狀結構的方法，通過分數值來表示節點之間的關係。每個節點都用一個分數值來表示其在樹中的位置，這個分數值可以唯一確定節點的層級關係。

### 資料結構
```sql
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    numerator     BIGINT NOT NULL,           -- 分子
    denominator   BIGINT NOT NULL,           -- 分母
    bug_id        BIGINT UNSIGNED NOT NULL,
    author        TEXT NOT NULL,
    comment_text  TEXT NOT NULL,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE(numerator, denominator)
);
```

### 預期資料範例
```
comment_id  numerator  denominator  author  comment_text
----------------------------------------------------------------
1           1          1            Fran    為什麼會有這個 bug？
2           2          3            Ollie   我想應該是 null pointer。
3           3          5            Fran    不是，我確認過了。
4           3          2            Kukla   應該要去檢查錯誤資料。
5           5          7            Ollie   對，這是個 bug。
```

### 數學原理解釋

在這個模型中，每個節點都用一個分數（numerator/denominator）來表示。子節點的分數值必須在其父節點的分數值和下一個同級節點的分數值之間。例如：

- 根節點：1/1
- 第一個子節點：2/3（介於 1/1 和 1/2 之間）
- 第二個子節點：3/5（介於 2/3 和 3/4 之間）

這種表示方法保證了：
1. 所有分數值都是唯一的
2. 可以在任意兩個分數之間插入新的分數
3. 節點的祖先關係可以通過分數大小直接判斷

### 查詢操作

1. 查詢子樹：
```sql
SELECT * FROM comments 
WHERE numerator/denominator > 2/3     -- 父節點的分數
AND numerator/denominator < 3/4       -- 下一個同級節點的分數
ORDER BY numerator/denominator;
```

2. 查詢祖先節點：
```sql
SELECT * FROM comments 
WHERE numerator/denominator < 3/5     -- 目標節點的分數
AND denominator < 5                   -- 目標節點的分母
ORDER BY numerator/denominator DESC;
```

3. 查詢同級節點：
```sql
SELECT * FROM comments 
WHERE numerator/denominator > 2/3     -- 上界
AND numerator/denominator < 4/5       -- 下界
AND denominator = 3                   -- 相同層級的分母
ORDER BY numerator/denominator;
```

### 修改操作

1. 插入新節點：
```sql
-- 在 2/3 和 3/4 之間插入新節點
INSERT INTO comments (
    numerator, denominator,
    bug_id, author, comment_text
) VALUES (
    5, 7,                            -- 新的分數值
    1234, 'Ollie', '新的評論'
);
```

2. 移動子樹：
```sql
-- 計算新的分數區間
WITH new_interval AS (
    SELECT 
        a.numerator as start_num,
        a.denominator as start_den,
        b.numerator as end_num,
        b.denominator as end_den
    FROM comments a, comments b
    WHERE a.comment_id = 2           -- 目標父節點
    AND b.comment_id = 3             -- 下一個同級節點
)
UPDATE comments
SET 
    numerator = ...,                 -- 計算新的分子
    denominator = ...                -- 計算新的分母
WHERE numerator/denominator > 3/5;   -- 移動整個子樹
```

### 主要困難點

1. **數值溢出問題**
   - 需要處理大數運算
   - 分數精度限制
   - 可能需要重新平衡樹

2. **分數計算複雜**
   - 插入節點時的分數計算
   - 移動子樹時的區間重算
   - 需要避免分數衝突

3. **效能考量**
   - 分數比較的效能
   - 索引優化困難
   - 需要定期維護重整理

### 小結

優點：
1. 不需要額外的關係表
2. 支援高效的樹操作
3. 節點關係判斷簡單
4. 適合靜態樹結構

缺點：
1. 實作複雜度高
2. 數值計算開銷大
3. 可能需要定期重整理
4. 不適合頻繁修改

## 效能比較

下表比較了各種延伸樹狀結構在不同操作下的效能表現：

| 操作 | Hybrid Adjacency | Path Array | Temporal | Nested Intervals |
|------|-----------------|------------|----------|------------------|
| 讀取子樹 | O(1) | O(1) | O(log n) | O(1) |
| 寫入節點 | O(1) | O(n) | O(1) | O(1) |
| 移動子樹 | O(n) | O(n) | O(1) | O(1) |
| 查詢祖先 | O(log n) | O(1) | O(log n) | O(1) |
| 歷史查詢 | N/A | N/A | O(1) | N/A |

## 應用場景建議

在選擇適合的延伸樹狀結構時，需要考慮以下因素：

1. **資料特性**
   - 樹的深度和寬度
   - 節點數量級別
   - 資料變動頻率
   - 查詢模式分析

2. **功能需求**
   - 歷史記錄追蹤
   - 即時性要求
   - 查詢複雜度
   - 維護便利性

3. **技術限制**
   - 資料庫特性
   - 硬體資源
   - 維護能力
   - 擴展需求

基於上述因素，以下是幾種常見場景的建議：

1. **大型論壇系統**
   - 建議使用 Hybrid Adjacency List
   - 原因：支援快速讀取和適度的修改操作

2. **文件版本控制**
   - 建議使用 Temporal Adjacency List
   - 原因：完整的歷史追蹤能力

3. **組織架構管理**
   - 建議使用 Nested Intervals
   - 原因：高效的層級關係查詢

4. **分散式系統**
   - 建議使用 Path Array
   - 原因：容易分片且維護成本較低

## 總結

延伸樹狀結構提供了多種解決特定問題的方案，每種方法都有其特定的優勢和限制：

1. **Hybrid Adjacency List**
   - 適合需要平衡讀寫效能的場景
   - 提供了額外的優化空間
   - 實作相對簡單

2. **Materialized Path with Array**
   - 特別適合支援陣列型態的資料庫
   - 提供直覺的路徑操作
   - 適合靜態結構

3. **Temporal Adjacency List**
   - 完整的歷史追蹤能力
   - 適合需要版本控制的場景
   - 資料完整性好

4. **Nested Intervals**
   - 數學基礎紮實
   - 高效的樹操作
   - 適合靜態或少量修改的場景

在實際應用中，可能需要根據具體需求組合使用多種方法，或者對某種方法進行客製化改良。最重要的是要根據實際需求和限制來選擇合適的實作方式，並在實作過程中持續優化和調整。