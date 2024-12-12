# Advanced and Hybrid Tree Structures in SQL

�b�F�ѤF�򥻪��𪬵��c��@�覡�]Adjacency List�BPath Enumeration�BNested Sets�BClosure Table�^�H�ζi����k�]Hybrid Adjacency List�BMaterialized Path with Array�BTemporal Adjacency List�BNested Intervals�^��A�ڭ̥i�H�i�@�B���Q�o�Ǥ�k���i�����ΩM�V�X�ϥΤ覡�C

## �ؿ�
1. [Closure Table with Path Enumeration](#closure-table-with-path-enumeration)
2. [Nested Sets with Closure Table](#nested-sets-with-closure-table)
3. [Temporal Closure Table](#temporal-closure-table)
4. [Materialized Path with Nested Sets](#materialized-path-with-nested-sets)
5. [�������𪬵��c](#distributed-tree-structure)

## Closure Table with Path Enumeration

�o�زV�X�覡���X�F Closure Table ���d�߮Ĳv�M Path Enumeration �����|��T�C

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

��ƽd�ҡG
```
-- comments ��
comment_id  bug_id  author  comment_text
--------------------------------------------------
1           1234    Fran    ������|���o�� bug ?
2           1234    Ollie   �ڷQ���ӬO null pointer�C
3           1234    Fran    ���O�A�ڽT�{�L�F�C

-- comment_paths ��
ancestor  descendant  path        path_length
--------------------------------------------------
1         1          1/          0
1         2          1/2/        1
1         3          1/2/3/      2
2         2          2/          0
2         3          2/3/        1
3         3          3/          0
```

### �u��
1. ���X��ؤ�k���d���u��
2. �䴩���������|�ާ@
3. ���@�������u�b��ӯS�ʤW

### �d�ߥܨ�
```sql
-- �ϥ� Closure Table �S�ʬd�ߤl��
SELECT c.* 
FROM comments c
JOIN comment_paths p ON c.comment_id = p.descendant
WHERE p.ancestor = 1;

-- �ϥ� Path Enumeration �S�ʬd�߯S�w���|
SELECT c.* 
FROM comments c
JOIN comment_paths p ON c.comment_id = p.descendant
WHERE p.path LIKE '1/2/%';
```

## Nested Sets with Closure Table

�o�زV�X�覡�A�X�ݭn�W�c�d�ߦ����֧�s�������C

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

### ��ƦP�B��k
```sql
-- �P�B Nested Sets �� Closure Table
INSERT INTO comment_paths (ancestor, descendant, distance)
SELECT p1.comment_id, p2.comment_id,
    (COUNT(*) OVER (PARTITION BY p1.comment_id)) - 1 as distance
FROM comments p1, comments p2
WHERE p2.nsleft BETWEEN p1.nsleft AND p1.nsright
ORDER BY p1.nsleft;
```

### ���į�d�ߥܨ�
```sql
-- �ϥ� Nested Sets �ֳt��X�l��
SELECT * FROM comments 
WHERE nsleft BETWEEN 
    (SELECT nsleft FROM comments WHERE comment_id = 1) 
    AND 
    (SELECT nsright FROM comments WHERE comment_id = 1);

-- �ϥ� Closure Table �ֳt��X�����l�`�I
SELECT c.* FROM comments c
JOIN comment_paths p ON c.comment_id = p.descendant
WHERE p.ancestor = 1 AND p.distance = 1;
```

## Temporal Closure Table

�o�ؤ�k�X�i�F Closure Table�A�[�J�F�ɶ����ת��䴩�C

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

### �B�z�`�I����
```sql
-- ��`�I���ʮɡA�إ߷s�����Ĵ����O��
INSERT INTO comment_paths (ancestor, descendant, valid_from, path_length)
SELECT t.ancestor, t.descendant, CURRENT_TIMESTAMP, t.path_length
FROM (
    -- �p��s�����|
    SELECT a.ancestor, d.descendant, 
           a.path_length + d.path_length + 1 as path_length
    FROM comment_paths a
    CROSS JOIN comment_paths d
    WHERE a.descendant = new_parent_id
    AND d.ancestor = moving_node_id
) t;

-- �N�°O���аO������
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

�o�زV�X�覡���X�F���|����Ū�ʩM Nested Sets ���ֳt�d�߯�O�C

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

### ���@����
```sql
-- �b�s�W�`�I�ɦP�ɧ�s��ص��c
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
    
    -- ������J��m
    SELECT nsright, path 
    INTO v_nsleft, v_path
    FROM comments 
    WHERE comment_id = p_parent_id;
    
    -- ��s Nested Sets
    UPDATE comments 
    SET nsright = nsright + 2,
        nsleft = CASE 
            WHEN nsleft > v_nsleft 
            THEN nsleft + 2 
            ELSE nsleft 
        END
    WHERE nsright >= v_nsleft;
    
    -- ���J�s�`�I
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

## �������𪬵��c

�b�j�W�����Τ��A�ڭ̥i��ݭn�N�𪬵��c������h�Ӹ�Ʈw���C�o�̴��Ѥ@�زV�X��סC

```sql
-- �D�`�I��]���Ϫ�^
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    shard_key     INT NOT NULL,                -- ������
    bug_id        BIGINT UNSIGNED NOT NULL,
    author        TEXT NOT NULL,
    comment_text  TEXT NOT NULL,
    
    INDEX(shard_key)
) PARTITION BY HASH(shard_key) PARTITIONS 4;

-- ���|���Y��
CREATE TABLE comment_paths (
    ancestor_shard    INT NOT NULL,
    ancestor_id       BIGINT UNSIGNED NOT NULL,
    descendant_shard  INT NOT NULL,
    descendant_id     BIGINT UNSIGNED NOT NULL,
    path              VARCHAR(1000),
    
    PRIMARY KEY(ancestor_shard, ancestor_id, descendant_shard, descendant_id)
);

-- �֨���
CREATE TABLE comment_tree_cache (
    root_id       BIGINT UNSIGNED NOT NULL,
    tree_data     JSON NOT NULL,
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    PRIMARY KEY(root_id)
);
```

### �������d�ߵ���
```sql
-- ����Ϭd�ߤl��
SELECT c.* 
FROM comments c
JOIN comment_paths p ON 
    c.comment_id = p.descendant_id AND 
    c.shard_key = p.descendant_shard
WHERE p.ancestor_id = ? 
AND p.ancestor_shard = ?;

-- �ϥΧ֨��[�t�d��
SELECT tree_data 
FROM comment_tree_cache 
WHERE root_id = ? 
AND updated_at > DATE_SUB(NOW(), INTERVAL 5 MINUTE);
```

## �į���

| �ާ@���� | Closure+Path | Nested+Closure | Temporal | MP+NS | ������ |
|---------|-------------|----------------|----------|-------|--------|
| Ū���l�� | O(1) | O(1) | O(1) | O(1) | O(1)* |
| �g�J�`�I | O(1) | O(n) | O(1) | O(n) | O(1)* |
| ���ʤl�� | O(n) | O(n) | O(1) | O(n) | O(n) |
| �ɶ��d�� | O(n) | O(n) | O(1) | O(n) | O(n) |

* ���]���A�����ϵ����M�֨�����

## �ϥΫ�ĳ

1. **Closure Table with Path Enumeration**
   - �A�X�ݭn�F�����|�ާ@������
   - �������x�s���������d�߮į�
   - �A�X���p���t��

2. **Nested Sets with Closure Table**
   - �A�XŪ�h�g�֪�����
   - �ݭn�����έp�d��
   - �A�X��í�w�����c

3. **Temporal Closure Table**
   - �ݭn���v�l�ܥ\��
   - �ݭn�ɶ����ת��d��
   - �A�X�f�p�t��

4. **Materialized Path with Nested Sets**
   - �ݭn�H���iŪ�����|
   - �ݭn���į઺�𪬬d��
   - �A�X�ɯ����t��

5. **�������𪬵��c**
   - �j�W�Ҩt��
   - �ݭn�����X�i
   - �i�H�e�Գ̲פ@�P��

## ��@�Ҷq

1. **��Ƥ@�P��**
   - �ϥΥ���]Transaction�^�T�O�h��ާ@���@�P��
   - �Ҽ{�ϥμ��[��w�]Optimistic Locking�^
   - ��@���v����B�z���ѱ��p

2. **�į��u��**
   - �A�����޵���
   - ��Ƥ���
   - �֨�����
   - �D���W�Ƶ���

3. **���@����**
   - �w������ˬd�M�״_
   - �į�ʱ�
   - �ƥ�����

## ����

��ܦX�A���𪬵��c��@�覡�ɡA�ݭn�Ҽ{�G

1. **���γ����ݨD**
   - Ū�g���
   - ��ƶq�j�p
   - �𪺲`�שM�e��
   - �ܰ��W�v

2. **�޳N����**
   - ��Ʈw�S��
   - �w��귽
   - ���@��O

3. **�X�i�ݨD**
   - ���Ӧ���
   - �s�\���X
   - �į�ݨD�ܤ�

�S���@�ؤ�k�O�������A����O�ھڹ�ڻݨD��ܦX�A����k�βV�X�覡�C�b��@�ɡA��ĳ����²�檺�覡�}�l�A�M��ھڻݨD�v�B��i�M�u�ơC