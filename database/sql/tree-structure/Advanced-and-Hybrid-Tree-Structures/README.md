# Advanced and Hybrid Tree Structures in SQL

在了解了基本的樹狀結構實作方式（Adjacency List、Path Enumeration、Nested Sets、Closure Table）以及進階方法（Hybrid Adjacency List、Materialized Path with Array、Temporal Adjacency List、Nested Intervals）後，我們可以進一步探討這些方法的進階應用和混合使用方式。

## 目錄
1. [Closure Table with Path Enumeration](#closure-table-with-path-enumeration)
2. [Nested Sets with Closure Table](#nested-sets-with-closure-table)
3. [Temporal Closure Table](#temporal-closure-table)
4. [Materialized Path with Nested Sets](#materialized-path-with-nested-sets)
5. [分散式樹狀結構](#distributed-tree-structure)

## Closure Table with Path Enumeration

這種混合方式結合了 Closure Table 的查詢效率和 Path Enumeration 的路徑資訊。

```sql
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    bug_id        BIGINT UNSIGNED NOT NULL,
    author        TEXT NOT NULL,
    comment_text  TEXT NOT NULL
);

CREATE TABLE comment_paths (
    ancestor      BIGINT UNSIGNED NOT NULL,
    descendant    BIGINT UNSIGNED NOT NULL,
    path          VARCHAR(1000),                -- Path Enumeration
    path_length   INT NOT NULL,
    
    PRIMARY KEY(ancestor, descendant),
    INDEX(path),
    FOREIGN KEY (ancestor) REFERENCES comments(comment_id),
    FOREIGN KEY (descendant) REFERENCES comments(comment_id)
);
```

資料範例：
```
-- comments 表
comment_id  bug_id  author  comment_text
--------------------------------------------------
1           1234    Fran    為什麼會有這個 bug ?
2           1234    Ollie   我想應該是 null pointer。
3           1234    Fran    不是，我確認過了。

-- comment_paths 表
ancestor  descendant  path        path_length
--------------------------------------------------
1         1          1/          0
1         2          1/2/        1
1         3          1/2/3/      2
2         2          2/          0
2         3          2/3/        1
3         3          3/          0
```

### 優勢
1. 結合兩種方法的查詢優勢
2. 支援複雜的路徑操作
3. 維護成本分攤在兩個特性上

### 查詢示例
```sql
-- 使用 Closure Table 特性查詢子樹
SELECT c.* 
FROM comments c
JOIN comment_paths p ON c.comment_id = p.descendant
WHERE p.ancestor = 1;

-- 使用 Path Enumeration 特性查詢特定路徑
SELECT c.* 
FROM comments c
JOIN comment_paths p ON c.comment_id = p.descendant
WHERE p.path LIKE '1/2/%';
```

## Nested Sets with Closure Table

這種混合方式適合需要頻繁查詢但較少更新的場景。

```sql
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    nsleft        INTEGER NOT NULL,
    nsright       INTEGER NOT NULL,
    bug_id        BIGINT UNSIGNED NOT NULL,
    author        TEXT NOT NULL,
    comment_text  TEXT NOT NULL
);

CREATE TABLE comment_paths (
    ancestor      BIGINT UNSIGNED NOT NULL,
    descendant    BIGINT UNSIGNED NOT NULL,
    distance      INTEGER NOT NULL,
    
    PRIMARY KEY(ancestor, descendant),
    FOREIGN KEY (ancestor) REFERENCES comments(comment_id),
    FOREIGN KEY (descendant) REFERENCES comments(comment_id)
);
```

### 資料同步方法
```sql
-- 同步 Nested Sets 到 Closure Table
INSERT INTO comment_paths (ancestor, descendant, distance)
SELECT p1.comment_id, p2.comment_id,
    (COUNT(*) OVER (PARTITION BY p1.comment_id)) - 1 as distance
FROM comments p1, comments p2
WHERE p2.nsleft BETWEEN p1.nsleft AND p1.nsright
ORDER BY p1.nsleft;
```

### 高效能查詢示例
```sql
-- 使用 Nested Sets 快速找出子樹
SELECT * FROM comments 
WHERE nsleft BETWEEN 
    (SELECT nsleft FROM comments WHERE comment_id = 1) 
    AND 
    (SELECT nsright FROM comments WHERE comment_id = 1);

-- 使用 Closure Table 快速找出直接子節點
SELECT c.* FROM comments c
JOIN comment_paths p ON c.comment_id = p.descendant
WHERE p.ancestor = 1 AND p.distance = 1;
```

## Temporal Closure Table

這種方法擴展了 Closure Table，加入了時間維度的支援。

```sql
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    bug_id        BIGINT UNSIGNED NOT NULL,
    author        TEXT NOT NULL,
    comment_text  TEXT NOT NULL,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE comment_paths (
    ancestor      BIGINT UNSIGNED NOT NULL,
    descendant    BIGINT UNSIGNED NOT NULL,
    valid_from    TIMESTAMP NOT NULL,
    valid_to      TIMESTAMP,
    path_length   INT NOT NULL,
    
    PRIMARY KEY(ancestor, descendant, valid_from),
    FOREIGN KEY (ancestor) REFERENCES comments(comment_id),
    FOREIGN KEY (descendant) REFERENCES comments(comment_id)
);
```

### 處理節點移動
```sql
-- 當節點移動時，建立新的有效期間記錄
INSERT INTO comment_paths (ancestor, descendant, valid_from, path_length)
SELECT t.ancestor, t.descendant, CURRENT_TIMESTAMP, t.path_length
FROM (
    -- 計算新的路徑
    SELECT a.ancestor, d.descendant, 
           a.path_length + d.path_length + 1 as path_length
    FROM comment_paths a
    CROSS JOIN comment_paths d
    WHERE a.descendant = new_parent_id
    AND d.ancestor = moving_node_id
) t;

-- 將舊記錄標記為結束
UPDATE comment_paths 
SET valid_to = CURRENT_TIMESTAMP
WHERE descendant IN (
    SELECT descendant 
    FROM comment_paths 
    WHERE ancestor = moving_node_id
)
AND valid_to IS NULL;
```

## Materialized Path with Nested Sets

這種混合方式結合了路徑的易讀性和 Nested Sets 的快速查詢能力。

```sql
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    path          VARCHAR(1000),
    nsleft        INTEGER NOT NULL,
    nsright       INTEGER NOT NULL,
    bug_id        BIGINT UNSIGNED NOT NULL,
    author        TEXT NOT NULL,
    comment_text  TEXT NOT NULL,
    
    INDEX(path),
    INDEX(nsleft),
    INDEX(nsright)
);
```

### 維護策略
```sql
-- 在新增節點時同時更新兩種結構
DELIMITER //

CREATE PROCEDURE add_comment(
    IN p_parent_id BIGINT,
    IN p_bug_id BIGINT,
    IN p_author TEXT,
    IN p_comment_text TEXT
)
BEGIN
    DECLARE v_nsleft INT;
    DECLARE v_path VARCHAR(1000);
    
    -- 獲取插入位置
    SELECT nsright, path 
    INTO v_nsleft, v_path
    FROM comments 
    WHERE comment_id = p_parent_id;
    
    -- 更新 Nested Sets
    UPDATE comments 
    SET nsright = nsright + 2,
        nsleft = CASE 
            WHEN nsleft > v_nsleft 
            THEN nsleft + 2 
            ELSE nsleft 
        END
    WHERE nsright >= v_nsleft;
    
    -- 插入新節點
    INSERT INTO comments (
        path, nsleft, nsright, bug_id, author, comment_text
    ) VALUES (
        CONCAT(v_path, LAST_INSERT_ID(), '/'),
        v_nsleft,
        v_nsleft + 1,
        p_bug_id,
        p_author,
        p_comment_text
    );
END //

DELIMITER ;
```

## 分散式樹狀結構

在大規模應用中，我們可能需要將樹狀結構分散到多個資料庫中。這裡提供一種混合方案。

```sql
-- 主節點表（分區表）
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    shard_key     INT NOT NULL,                -- 分區鍵
    bug_id        BIGINT UNSIGNED NOT NULL,
    author        TEXT NOT NULL,
    comment_text  TEXT NOT NULL,
    
    INDEX(shard_key)
) PARTITION BY HASH(shard_key) PARTITIONS 4;

-- 路徑關係表
CREATE TABLE comment_paths (
    ancestor_shard    INT NOT NULL,
    ancestor_id       BIGINT UNSIGNED NOT NULL,
    descendant_shard  INT NOT NULL,
    descendant_id     BIGINT UNSIGNED NOT NULL,
    path              VARCHAR(1000),
    
    PRIMARY KEY(ancestor_shard, ancestor_id, descendant_shard, descendant_id)
);

-- 快取表
CREATE TABLE comment_tree_cache (
    root_id       BIGINT UNSIGNED NOT NULL,
    tree_data     JSON NOT NULL,
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    PRIMARY KEY(root_id)
);
```

### 分散式查詢策略
```sql
-- 跨分區查詢子樹
SELECT c.* 
FROM comments c
JOIN comment_paths p ON 
    c.comment_id = p.descendant_id AND 
    c.shard_key = p.descendant_shard
WHERE p.ancestor_id = ? 
AND p.ancestor_shard = ?;

-- 使用快取加速查詢
SELECT tree_data 
FROM comment_tree_cache 
WHERE root_id = ? 
AND updated_at > DATE_SUB(NOW(), INTERVAL 5 MINUTE);
```

## 效能比較

| 操作類型 | Closure+Path | Nested+Closure | Temporal | MP+NS | 分散式 |
|---------|-------------|----------------|----------|-------|--------|
| 讀取子樹 | O(1) | O(1) | O(1) | O(1) | O(1)* |
| 寫入節點 | O(1) | O(n) | O(1) | O(n) | O(1)* |
| 移動子樹 | O(n) | O(n) | O(1) | O(n) | O(n) |
| 時間查詢 | O(n) | O(n) | O(1) | O(n) | O(n) |

* 假設有適當的分區策略和快取機制

## 使用建議

1. **Closure Table with Path Enumeration**
   - 適合需要靈活路徑操作的場景
   - 較高的儲存成本換取查詢效能
   - 適合中小型系統

2. **Nested Sets with Closure Table**
   - 適合讀多寫少的場景
   - 需要複雜統計查詢
   - 適合較穩定的結構

3. **Temporal Closure Table**
   - 需要歷史追蹤功能
   - 需要時間維度的查詢
   - 適合審計系統

4. **Materialized Path with Nested Sets**
   - 需要人類可讀的路徑
   - 需要高效能的樹狀查詢
   - 適合導航類系統

5. **分散式樹狀結構**
   - 大規模系統
   - 需要水平擴展
   - 可以容忍最終一致性

## 實作考量

1. **資料一致性**
   - 使用交易（Transaction）確保多表操作的一致性
   - 考慮使用樂觀鎖定（Optimistic Locking）
   - 實作補償機制處理失敗情況

2. **效能優化**
   - 適當的索引策略
   - 資料分區
   - 快取機制
   - 非正規化策略

3. **維護策略**
   - 定期資料檢查和修復
   - 效能監控
   - 備份策略

## 結論

選擇合適的樹狀結構實作方式時，需要考慮：

1. **應用場景需求**
   - 讀寫比例
   - 資料量大小
   - 樹的深度和寬度
   - 變動頻率

2. **技術限制**
   - 資料庫特性
   - 硬體資源
   - 維護能力

3. **擴展需求**
   - 未來成長
   - 新功能整合
   - 效能需求變化

沒有一種方法是完美的，關鍵是根據實際需求選擇合適的方法或混合方式。在實作時，建議先用簡單的方式開始，然後根據需求逐步改進和優化。