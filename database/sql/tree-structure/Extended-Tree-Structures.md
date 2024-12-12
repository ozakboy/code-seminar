# Extended Tree Structures in SQL

## �e��

�b�z�ѤF�򥻪��𪬵��c��{��k��A�ڭ̥i�H�i�@�B���Q�@�ǧ�i������@�覡�C�o�Ǥ�k�q�`���X�F�h�ذ�¦��k���u�I�A�Ϊ̰w��S�w�ϥγ����i��F�u�ơC����N�ԲӤ��дX�ضi�����𪬵��c��{��k�A�ä��R���̪��u���I�M�A�γ����C

## ���һ���

���ڭ̩��򤧫e�� bug �l�ܨt�νd�ҡA���[�J�@�ǧ�i�����ݨD�G

1. �ݭn�ֳt�s�����N�`�I��������|
2. �䴩�j�q�����׸��
3. �ݭn�O�d���ת��ק���v
4. �䴩���Ʈw���������[�c
5. �ݭn���Ī��𪬵��c�ާ@

### �d�Ҹ�Ƶ��c

```
Fran: ������|���o�� bug�H
�u�w Ollie: �ڷQ���ӬO null pointer�C
�x  �|�w Fran: ���O�A�ڽT�{�L�F�C
�x     �|�w Ollie: [�w�s��] �T�{�O memory leak�C
�|�w Kukla: ���ӭn�h�ˬd���~��ơC
   �u�w Fran: �д��ѧ�h�Ӹ`�C
   �|�w Ollie: �ڨӸɥR�����C
      �|�w Kukla: �w��s�������C
```

## 1. Hybrid Adjacency List (�V�X�F����)�ҫ�

### ����
�o�O Adjacency List ����}�����A�z�L�W�[�B�~�������u�Ƭd�߮į�C

### ��Ƶ��c
```sql
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    parent_id     BIGINT UNSIGNED,
    root_id       BIGINT UNSIGNED,           -- ���V�ڸ`�I
    depth         INT NOT NULL DEFAULT 0,    -- �x�s�`��
    path_order    VARCHAR(1000),             -- �Ƨǥθ��|
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

### �w����ƽd��
```
comment_id  parent_id   root_id  depth  path_order   author  comment_text
--------------------------------------------------------------------------------
1           NULL        1        0      0001         Fran    ������|���o�� bug�H
2           1           1        1      0001.0001    Ollie   �ڷQ���ӬO null pointer�C
3           2           1        2      0001.0001.0001 Fran  ���O�A�ڽT�{�L�F�C
4           1           1        1      0001.0002    Kukla   ���ӭn�h�ˬd���~��ơC
5           4           1        2      0001.0002.0001 Fran  �д��ѧ�h�Ӹ`�C
```

### �d�߾ާ@

1. �d�߯S�w���ת��Ҧ��^�СG
```sql
SELECT * FROM comments 
WHERE root_id = 1 
AND depth > (SELECT depth FROM comments WHERE comment_id = 1)
ORDER BY path_order;
```

2. �d�ߪ����^�СG
```sql
SELECT * FROM comments 
WHERE parent_id = 1 
ORDER BY path_order;
```

3. �d�߯S�w�h�Ū����סG
```sql
SELECT * FROM comments 
WHERE root_id = 1 
AND depth = 2;
```

### �ק�ާ@

1. �s�W���סG
```sql
INSERT INTO comments (
    parent_id, root_id, depth, path_order, 
    bug_id, author, comment_text
) 
SELECT 
    4,                              -- parent_id
    root_id,                        -- �~�Ӯڸ`�I
    depth + 1,                      -- �`�ץ[�@
    CONCAT(path_order, '.', LPAD(LAST_INSERT_ID(), 4, '0')), -- �c�رƧǸ��|
    bug_id, 'Kukla', '�w��s�������C'
FROM comments 
WHERE comment_id = 4;
```

2. ���ʵ��סG
```sql
UPDATE comments c1, comments c2
SET 
    c1.parent_id = 2,
    c1.depth = c2.depth + 1,
    c1.path_order = CONCAT(c2.path_order, '.', LPAD(c1.comment_id, 4, '0'))
WHERE c1.comment_id = 4
AND c2.comment_id = 2;
```

### �D�n�x���I

1. **���|���@**
   - path_order �ݭn�w������z
   - ���ʤj�q�`�I�ɮį���t
   - �ݭn�B�z���|���׭���

2. **�@�P�ʺ��@**
   - �h�����ݭn�P�B��s
   - �ݭn�T�O root_id ���T��
   - depth �p��ݭn�B�~�B�z

3. **�į�~�V**
   - �j�q���ʾާ@�ɮį୰�C
   - �`�h�`�I�����|�i��L��
   - �ݭn�B�~�����޺��@����

### �p��

�u�I�G
1. �d�߮į��u��
2. �䴩�h�جd�߼Ҧ�
3. �i�H�ֳt�o��`�I�`��
4. �e����{�Ƨǥ\��

���I�G
1. �ݭn���@�h�����
2. ���ʾާ@������
3. �s�x�Ŷ����j
4. �ݭn�w�����@���|���

## 2. Materialized Path with Array (�}�C�Ƹ��|)�ҫ�

### ����
�o�ؤ�k�ϥΰ}�C���A���x�s���|��T�A�S�O�A�X�b PostgreSQL �o���䴩�}�C���A����Ʈw���ϥΡC

### ��Ƶ��c
```sql
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    path_array    INTEGER[],                  -- �ϥΰ}�C�x�s���|
    bug_id        BIGINT UNSIGNED NOT NULL,
    author        TEXT NOT NULL,
    comment_text  TEXT NOT NULL,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX(path_array)
);
```

### �w����ƽd��
```
comment_id  path_array    author  comment_text
------------------------------------------------------------
1           {1}           Fran    ������|���o�� bug�H
2           {1,2}         Ollie   �ڷQ���ӬO null pointer�C
3           {1,2,3}       Fran    ���O�A�ڽT�{�L�F�C
4           {1,4}         Kukla   ���ӭn�h�ˬd���~��ơC
5           {1,4,5}       Ollie   ��A�o�O�� bug�C
```

### �d�߾ާ@

1. �d�ߤl��G
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

2. �d�߯S�w�h�šG
```sql
SELECT * FROM comments 
WHERE array_length(path_array, 1) = 2;
```

3. �d�߯������|�G
```sql
SELECT * FROM comments 
WHERE comment_id = ANY(
    SELECT unnest(path_array) 
    FROM comments 
    WHERE comment_id = 3
);
```

### �ק�ާ@

1. �s�W�`�I�G
```sql
INSERT INTO comments (path_array, bug_id, author, comment_text)
SELECT 
    array_append(path_array, nextval('comment_id_seq')),
    bug_id,
    'Kukla',
    '�s���^��'
FROM comments 
WHERE comment_id = 4;
```

2. ���ʸ`�I�G
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

### �D�n�x���I

1. **��Ʈw�ۮe��**
   - �ݭn��Ʈw�䴩�}�C���A
   - ���P��Ʈw��{�覡���P
   - �i��ݭn�ۭq��Ƥ䴩

2. **�}�C�ާ@������**
   - �ݭn���x�}�C�ާ@�y�k
   - �����d�߮į���t
   - ���޵����ݭn�S�O�]�p

3. **��ƺ��@**
   - �}�C��s��������
   - �ݭn�B�z��Ƥ@�P��
   - �i��ݭn�w�����հ}�C

### �p��

�u�I�G
1. ��Ƶ��c���[
2. �䴩�״I���}�C�ާ@
3. �e����{��M��
4. �A�X�R�A�𵲺c

���I�G
1. ��Ʈw�䴩����
2. �����d�߮į���D
3. ���@��������
4. ���A�X�W�c�ק�

## 3. Temporal Adjacency List (�ɧǾF����)�ҫ�

### ����
�o�ؤ�k�X�i�F�򥻪��F����ҫ��A�[�J�F�ɶ����סA�S�O�A�X�ݭn�l�ܾ��v�ܧ󪺳����C

### ��Ƶ��c
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

### �w����ƽd��
```
comment_id  parent_id   valid_from    valid_to      version  author  comment_text
------------------------------------------------------------------------------------
1           NULL        2024-01-01    NULL          1        Fran    ������|���o�� bug�H
2           1           2024-01-01    2024-01-02    1        Ollie   �ڷQ���ӬO null pointer�C
2           1           2024-01-02    NULL          2        Ollie   [�ק�] �T�{�O memory leak�C
3           2           2024-01-02    NULL          1        Fran    ���O�A�ڽT�{�L�F�C
```

### �d�߾ާ@

1. �d�߷�e���Ī��𪬵��c�G
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

2. �d�߯S�w�ɶ��I�����A�G
```sql
SELECT * FROM comments 
WHERE '2024-01-01 12:00:00' BETWEEN valid_from 
AND COALESCE(valid_to, '9999-12-31');
```

3. �d�߭ק���v�G
```sql
SELECT * FROM comments 
WHERE comment_id = 2 
ORDER BY version;
```

### �ק�ާ@

1. ��s���סG
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
    bug_id, author, '�ק�᪺���e',
    version + 1
FROM comments 
WHERE comment_id = 2 
AND valid_to = CURRENT_TIMESTAMP;
```

2. ���ʸ`�I�G
```sql
-- ������e����
UPDATE comments 
SET valid_to = CURRENT_TIMESTAMP 
WHERE comment_id = 4 
AND valid_to IS NULL;

-- �إ߷s����
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

### �D�n�x���I

1. **���v��ƺ޲z**
   - ��ƶq�ֳt�W��
   - �ݭn���v��ƲM�z����
   - �d�߮į��H�ɶ����C
   
2. **�ɶ��@�P��**
   - �ݭn�B�z�ɰϰ��D
   - �T�O�����ഫ���s���
   - �B�z�õo�ק�Ĭ�

3. **�d�߽�����**
   - �ݭn�Ҽ{�ɶ�����
   - �į��u�Ƹ����x��
   - ���޳]�p�󬰽���

### �p��

�u�I�G
1. ���㪺���v�l�ܯ�O
2. �䴩�ɶ��I�d��
3. ��Ƨ�s�z����
4. �A�X�]�ְl�ܻݨD

���I�G
1. �s�x�Ŷ��ݨD�j
2. �d�߸�������
3. �į��H��ƶq�W�[�ӭ��C
4. ���@��������

## 4. Nested Intervals (�O�M�϶�)�ҫ�

### ����
�o�O�@�بϥμƾǭ�z�Ӻ޲z�𪬵��c����k�A�q�L���ƭȨӪ�ܸ`�I���������Y�C�C�Ӹ`�I���Τ@�Ӥ��ƭȨӪ�ܨ�b�𤤪���m�A�o�Ӥ��ƭȥi�H�ߤ@�T�w�`�I���h�����Y�C

### ��Ƶ��c
```sql
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    numerator     BIGINT NOT NULL,           -- ���l
    denominator   BIGINT NOT NULL,           -- ����
    bug_id        BIGINT UNSIGNED NOT NULL,
    author        TEXT NOT NULL,
    comment_text  TEXT NOT NULL,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE(numerator, denominator)
);
```

### �w����ƽd��
```
comment_id  numerator  denominator  author  comment_text
----------------------------------------------------------------
1           1          1            Fran    ������|���o�� bug�H
2           2          3            Ollie   �ڷQ���ӬO null pointer�C
3           3          5            Fran    ���O�A�ڽT�{�L�F�C
4           3          2            Kukla   ���ӭn�h�ˬd���~��ơC
5           5          7            Ollie   ��A�o�O�� bug�C
```

### �ƾǭ�z����

�b�o�Ӽҫ����A�C�Ӹ`�I���Τ@�Ӥ��ơ]numerator/denominator�^�Ӫ�ܡC�l�`�I�����ƭȥ����b����`�I�����ƭȩM�U�@�ӦP�Ÿ`�I�����ƭȤ����C�Ҧp�G

- �ڸ`�I�G1/1
- �Ĥ@�Ӥl�`�I�G2/3�]���� 1/1 �M 1/2 �����^
- �ĤG�Ӥl�`�I�G3/5�]���� 2/3 �M 3/4 �����^

�o�ت�ܤ�k�O�ҤF�G
1. �Ҧ����ƭȳ��O�ߤ@��
2. �i�H�b���N��Ӥ��Ƥ������J�s������
3. �`�I���������Y�i�H�q�L���Ƥj�p�����P�_

### �d�߾ާ@

1. �d�ߤl��G
```sql
SELECT * FROM comments 
WHERE numerator/denominator > 2/3     -- ���`�I������
AND numerator/denominator < 3/4       -- �U�@�ӦP�Ÿ`�I������
ORDER BY numerator/denominator;
```

2. �d�߯����`�I�G
```sql
SELECT * FROM comments 
WHERE numerator/denominator < 3/5     -- �ؼи`�I������
AND denominator < 5                   -- �ؼи`�I������
ORDER BY numerator/denominator DESC;
```

3. �d�ߦP�Ÿ`�I�G
```sql
SELECT * FROM comments 
WHERE numerator/denominator > 2/3     -- �W��
AND numerator/denominator < 4/5       -- �U��
AND denominator = 3                   -- �ۦP�h�Ū�����
ORDER BY numerator/denominator;
```

### �ק�ާ@

1. ���J�s�`�I�G
```sql
-- �b 2/3 �M 3/4 �������J�s�`�I
INSERT INTO comments (
    numerator, denominator,
    bug_id, author, comment_text
) VALUES (
    5, 7,                            -- �s�����ƭ�
    1234, 'Ollie', '�s������'
);
```

2. ���ʤl��G
```sql
-- �p��s�����ư϶�
WITH new_interval AS (
    SELECT 
        a.numerator as start_num,
        a.denominator as start_den,
        b.numerator as end_num,
        b.denominator as end_den
    FROM comments a, comments b
    WHERE a.comment_id = 2           -- �ؼФ��`�I
    AND b.comment_id = 3             -- �U�@�ӦP�Ÿ`�I
)
UPDATE comments
SET 
    numerator = ...,                 -- �p��s�����l
    denominator = ...                -- �p��s������
WHERE numerator/denominator > 3/5;   -- ���ʾ�Ӥl��
```

### �D�n�x���I

1. **�ƭȷ��X���D**
   - �ݭn�B�z�j�ƹB��
   - ���ƺ�׭���
   - �i��ݭn���s���ž�

2. **���ƭp�����**
   - ���J�`�I�ɪ����ƭp��
   - ���ʤl��ɪ��϶�����
   - �ݭn�קK���ƽĬ�

3. **�į�Ҷq**
   - ���Ƥ�����į�
   - �����u�Ƨx��
   - �ݭn�w�����@����z

### �p��

�u�I�G
1. ���ݭn�B�~�����Y��
2. �䴩���Ī���ާ@
3. �`�I���Y�P�_²��
4. �A�X�R�A�𵲺c

���I�G
1. ��@�����װ�
2. �ƭȭp��}�P�j
3. �i��ݭn�w������z
4. ���A�X�W�c�ק�

## �į���

�U�����F�U�ة����𪬵��c�b���P�ާ@�U���į��{�G

| �ާ@ | Hybrid Adjacency | Path Array | Temporal | Nested Intervals |
|------|-----------------|------------|----------|------------------|
| Ū���l�� | O(1) | O(1) | O(log n) | O(1) |
| �g�J�`�I | O(1) | O(n) | O(1) | O(1) |
| ���ʤl�� | O(n) | O(n) | O(1) | O(1) |
| �d�߯��� | O(log n) | O(1) | O(log n) | O(1) |
| ���v�d�� | N/A | N/A | O(1) | N/A |

## ���γ�����ĳ

�b��ܾA�X�������𪬵��c�ɡA�ݭn�Ҽ{�H�U�]���G

1. **��ƯS��**
   - �𪺲`�שM�e��
   - �`�I�ƶq�ŧO
   - ����ܰ��W�v
   - �d�߼Ҧ����R

2. **�\��ݨD**
   - ���v�O���l��
   - �Y�ɩʭn�D
   - �d�߽�����
   - ���@�K�Q��

3. **�޳N����**
   - ��Ʈw�S��
   - �w��귽
   - ���@��O
   - �X�i�ݨD

���W�z�]���A�H�U�O�X�ر`����������ĳ�G

1. **�j���׾¨t��**
   - ��ĳ�ϥ� Hybrid Adjacency List
   - ��]�G�䴩�ֳtŪ���M�A�ת��ק�ާ@

2. **��󪩥�����**
   - ��ĳ�ϥ� Temporal Adjacency List
   - ��]�G���㪺���v�l�ܯ�O

3. **��´�[�c�޲z**
   - ��ĳ�ϥ� Nested Intervals
   - ��]�G���Ī��h�����Y�d��

4. **�������t��**
   - ��ĳ�ϥ� Path Array
   - ��]�G�e�������B���@�������C

## �`��

�����𪬵��c���ѤF�h�ظѨM�S�w���D����סA�C�ؤ�k������S�w���u�թM����G

1. **Hybrid Adjacency List**
   - �A�X�ݭn����Ū�g�į઺����
   - ���ѤF�B�~���u�ƪŶ�
   - ��@�۹�²��

2. **Materialized Path with Array**
   - �S�O�A�X�䴩�}�C���A����Ʈw
   - ���Ѫ�ı�����|�ާ@
   - �A�X�R�A���c

3. **Temporal Adjacency List**
   - ���㪺���v�l�ܯ�O
   - �A�X�ݭn�����������
   - ��Ƨ���ʦn

4. **Nested Intervals**
   - �ƾǰ�¦�Ϲ�
   - ���Ī���ާ@
   - �A�X�R�A�Τֶq�ק諸����

�b������Τ��A�i��ݭn�ھڨ���ݨD�զX�ϥΦh�ؤ�k�A�Ϊ̹�Y�ؤ�k�i��Ȼs�Ƨ�}�C�̭��n���O�n�ھڹ�ڻݨD�M����ӿ�ܦX�A����@�覡�A�æb��@�L�{�������u�ƩM�վ�C