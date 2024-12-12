# Extended Tree Structures in SQL

���F�򥻪��𪬵��c��@�覡�~�A�٦��@�Ǩ�L�`������@��k�ȱo���Q�C����N���дX�ضi�����𪬵��c��@�覡�A�û������̦U�۪��ϥγ����C

## Hybrid Adjacency List

�o�O Adjacency List ����}�����A�q�L�W�[�B�~�������u�Ƭd�߮į�C

```sql
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    parent_id     BIGINT UNSIGNED,
    root_id       BIGINT UNSIGNED,           -- �s�W�G�����ڸ`�I
    depth         INT NOT NULL DEFAULT 0,    -- �s�W�G�����`��
    path_order    VARCHAR(1000),             -- �s�W�G�Ƨǥθ��|
    bug_id        BIGINT UNSIGNED NOT NULL,
    author        TEXT NOT NULL,
    comment_text  TEXT NOT NULL,
    
    FOREIGN KEY (parent_id) REFERENCES comments(comment_id),
    FOREIGN KEY (root_id) REFERENCES comments(comment_id)
);
```

��ƽd�ҡG
```
comment_id  parent_id   root_id  depth  path_order   author  comment
--------------------------------------------------------------------------------
1           NULL        1        0      0001         Fran    ������|���o�� bug ?
2           1           1        1      0001.0001    Ollie   �ڷQ���ӬO null pointer�C
3           2           1        2      0001.0001.0001 Fran  ���O�A�ڽT�{�L�F�C
4           1           1        1      0001.0002    Kukla   ���ӭn�h�ˬd���~��ơC
```

### �u�I

1. �O���F Adjacency List ��²���
2. �q�L root_id �ֳt�d�߾�ӰQ�צ�
3. �q�L depth ������ܼh��
4. path_order �䴩�۵M�Ƨ�

### �d�߽d��

```sql
-- ���o�Y�ӰQ�צꪺ�Ҧ��^��
SELECT * FROM comments 
WHERE root_id = 1 
ORDER BY path_order;

-- ���o�S�w�h�Ū��^��
SELECT * FROM comments 
WHERE root_id = 1 
AND depth = 2;

-- ���o�����^��
SELECT * FROM comments 
WHERE parent_id = 1 
ORDER BY path_order;
```

## Materialized Path with Array

�o�O Path Enumeration ������A�ϥΰ}�C���A���x�s���|�C����k�b PostgreSQL �S�O���ΡA�]�����䴩�}�C���A�C

```sql
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    path_array    INTEGER[],                  -- �ϥΰ}�C�x�s���|
    bug_id        BIGINT UNSIGNED NOT NULL,
    author        TEXT NOT NULL,
    comment_text  TEXT NOT NULL
);
```

��ƽd�ҡ]PostgreSQL �y�k�^�G
```
comment_id  path_array    author  comment
------------------------------------------------------------
1           {1}           Fran    ������|���o�� bug ?
2           {1,2}         Ollie   �ڷQ���ӬO null pointer�C
3           {1,2,3}       Fran    ���O�A�ڽT�{�L�F�C
4           {1,4}         Kukla   ���ӭn�h�ˬd���~��ơC
```

### �u�I

1. �e���z�ѩM�ާ@�}�C���c
2. �䴩��Ͱ}�C�ާ@�M���
3. �i�H����������|�h��

### �d�߽d�ҡ]PostgreSQL�^

```sql
-- ���o�l��
SELECT * FROM comments 
WHERE path_array[1] = 1 
ORDER BY path_array;

-- ���o�S�w�`�ת��`�I
SELECT * FROM comments 
WHERE array_length(path_array, 1) = 2;

-- ���o�����|
SELECT * FROM comments 
WHERE path_array = (
    SELECT path_array[1:array_length(path_array,1)-1]
    FROM comments 
    WHERE comment_id = 3
);
```

## Temporal Adjacency List

�o�ؤ�k�S�O�A�X�ݭn�B�z�������v�ήɶ��Ǫ��𪬵��c�C

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

��ƽd�ҡG
```
comment_id  parent_id   valid_from    valid_to      version  author  comment
------------------------------------------------------------------------------------
1           NULL        2023-01-01    NULL          1        Fran    ������|���o�� bug ?
2           1           2023-01-01    2023-01-02    1        Ollie   �ڷQ���ӬO null pointer�C
2           1           2023-01-02    NULL          2        Ollie   [�s��] �T�{�O null pointer�C
3           2           2023-01-02    NULL          1        Fran    ���O�A�ڽT�{�L�F�C
```

### �u�I

1. �䴩���v�����l��
2. �i�H�d�߯S�w�ɶ��I���𪬵��c
3. �A�X�ݭn�]�ְl�ܪ��t��

### �d�߽d��

```sql
-- ���o��e���Ī��𪬵��c
SELECT * FROM comments 
WHERE valid_to IS NULL 
START WITH parent_id IS NULL 
CONNECT BY PRIOR comment_id = parent_id;

-- ���o�S�w�ɶ��I���𪬵��c
SELECT * FROM comments 
WHERE '2023-01-01' BETWEEN valid_from AND COALESCE(valid_to, '9999-12-31');

-- ���o���ת��ק���v
SELECT * FROM comments 
WHERE comment_id = 2 
ORDER BY version;
```

## Nested Intervals

�o�O�@�ظ��֨����ܦ��쪺��@�覡�A�ϥμƾǭ�z�Ӻ��@�𪬵��c�C

```sql
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    numerator     BIGINT NOT NULL,           -- ���l
    denominator   BIGINT NOT NULL,           -- ����
    bug_id        BIGINT UNSIGNED NOT NULL,
    author        TEXT NOT NULL,
    comment_text  TEXT NOT NULL,
    
    UNIQUE(numerator, denominator)
);
```

��ƽd�ҡG
```
comment_id  numerator  denominator  author  comment
----------------------------------------------------------------
1           1          1            Fran    ������|���o�� bug ?
2           2          3            Ollie   �ڷQ���ӬO null pointer�C
3           3          5            Fran    ���O�A�ڽT�{�L�F�C
4           3          2            Kukla   ���ӭn�h�ˬd���~��ơC
```

### ��z����

- �C�Ӹ`�I���Τ��ƪ�ܡ]numerator/denominator�^
- �l�`�I�����ƭȥ����b���`�I�����ư϶���
- �ϥγs���Ʋz�רӲ��ͷs�`�I����

### �u�I

1. ���ݭn��s�{���`�I
2. �䴩�L���h��
3. �ƾǹB��ֳt

### �d�߽d��

```sql
-- ���o�l��]�ƾǰ϶��d�ߡ^
SELECT * FROM comments c1
WHERE c1.numerator/c1.denominator > 1/1 
AND c1.numerator/c1.denominator < 2/1
ORDER BY c1.numerator/c1.denominator;

-- �P�_���l���Y
SELECT * FROM comments c1, comments c2
WHERE c2.numerator/c2.denominator > c1.numerator/c1.denominator
AND c2.numerator/c2.denominator < (c1.numerator+1)/(c1.denominator);
```

## �ϥΫ�ĳ

1. **Hybrid Adjacency List**
   - �A�X�@�몺�Q�צ�t��
   - �ݭn²�檺�h�ű���
   - �����d�߮į�����Q�ӽ���

2. **Materialized Path with Array**
   - �ϥΤ䴩�}�C���A����Ʈw
   - �ݭn�F�������|�ާ@
   - ���|�����ܤƤ��j

3. **Temporal Adjacency List**
   - �ݭn��������
   - �ݭn�]�ְl��
   - ���ɶ����ת�����

4. **Nested Intervals**
   - ���֧�ʪ��𪬵��c
   - �ݭn�䴩�L���h��
   - �����ƾǹB��į�

## �į�Ҷq

�H�U�O�U�ؤ�k�b���P�ާ@�U���į����G

| �ާ@ | Hybrid Adjacency | Path Array | Temporal | Nested Intervals |
|------|-----------------|------------|----------|------------------|
| Ū���l�� | O(1) | O(1) | O(log n) | O(1) |
| ���J�`�I | O(1) | O(n) | O(1) | O(1) |
| ���ʤl�� | O(n) | O(n) | O(n) | O(1) |
| �R���`�I | O(n) | O(n) | O(1)* | O(1) |

*: �ϥγn�R���]soft delete�^��

## �`��

�C�ؤ�k������S�w���ϥγ����G

1. **�@������**�G�ϥ� Hybrid Adjacency List
2. **�ݭn��������**�G�ϥ� Temporal Adjacency List
3. **�ݭn�j�j���|�ާ@**�G�ϥ� Materialized Path with Array
4. **�S��ƾǻݨD**�G�ϥ� Nested Intervals

��ܾA����k�ɡA�ݭn�Ҽ{�G

- ��ƪ��ϥμҦ��]Ū�h�g�֡H�g�hŪ�֡H�^
- �𪺯S�ʡ]�`�סH�e�סH�ܰ��W�v�H�^
- �~�ȻݨD�]��������H�]�ְl�ܡH�^
- ���@������