# Advanced and Hybrid Tree Structures in SQL

## �e��

�b�F�ѧ��򥻪���Ƶ��c��@�覡(Adjacency List�BPath Enumeration�BNested Sets�BClosure Table)�H�ζi����k(Hybrid Adjacency List�BMaterialized Path with Array�BTemporal Adjacency List�BNested Intervals)��A�ڭ̥i�H�i�@�B���Q�o�Ǥ�k���i�����ΩM�V�X�ϥΤ覡�C����N�`�J�Q�״X�ضi�����𪬵��c��{��k�A�øԲӤ��R���̪��u�աB�A�γ����ι�@�Ҷq�C

## ���һ���

���򤧫e�� bug �l�ܨt�νd�ҡA���[�J��h�i���ݨD�G

1. �ݭn�䴩�j�q�����׸�ơ]�ʸU�ŧO�^
2. �ݭn����O�s���v�s�����
3. ���ץi�H�Q�аO���w�s��Τw�R��
4. �ݭn�䴩���Ʈw���������[�c
5. �ݭn�ֳt���𪬵��c�ާ@
6. �ݭn�䴩���㪺��������

### �d�Ҹ�Ƶ��c

```
Fran: ������|���o�� bug�H
�u�w Ollie: �ڷQ���ӬO null pointer�C
�x  �|�w Fran: ���O�A�ڽT�{�L�F�C
�x     �|�w Ollie: [�w�s��] �T�{�O memory leak�C
�x
�|�w Kukla: ���ӭn�ˬd���~��ơC
   �u�w Ollie: ��A�o�O�� bug�C
   �x  �|�w Fran: �Х[�W�ˬd�C
   �x     �|�w Kukla: �w�ץ��������C
   �x
   �|�w Fran: [�w�R��] �o�q�������C
```

## 1. Closure Table with Path Enumeration

### ����
�o�زV�X�覡���X�F Closure Table ���d�߮Ĳv�M Path Enumeration �����|��T�C�ϥ� Closure Table �Ӻ��@�`�I���Y�A�P�ɫO�s���|��T�H�䴩���F�����d�ߡC

### ��Ƶ��c
```sql
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    bug_id        BIGINT UNSIGNED NOT NULL,
    author        TEXT NOT NULL,
    comment_text  TEXT NOT NULL,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    is_deleted    BOOLEAN DEFAULT FALSE,
    version       INT NOT NULL DEFAULT 1
);

CREATE TABLE comment_paths (
    ancestor      BIGINT UNSIGNED NOT NULL,
    descendant    BIGINT UNSIGNED NOT NULL,
    path          VARCHAR(1000),
    path_length   INT NOT NULL,
    
    PRIMARY KEY(ancestor, descendant),
    INDEX(path),
    FOREIGN KEY (ancestor) REFERENCES comments(comment_id),
    FOREIGN KEY (descendant) REFERENCES comments(comment_id)
);

CREATE TABLE comment_versions (
    comment_id    BIGINT UNSIGNED NOT NULL,
    version       INT NOT NULL,
    comment_text  TEXT NOT NULL,
    modified_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    modified_by   TEXT NOT NULL,
    
    PRIMARY KEY(comment_id, version),
    FOREIGN KEY (comment_id) REFERENCES comments(comment_id)
);
```

### �w����ƽd��
```
-- comments ��
comment_id  bug_id  author  comment_text                    is_deleted  version
--------------------------------------------------------------------------------
1           1234    Fran    ������|���o�� bug�H           false       1
2           1234    Ollie   �ڷQ���ӬO null pointer�C      false       2
3           1234    Fran    ���O�A�ڽT�{�L�F�C             false       1
4           1234    Ollie   �T�{�O memory leak�C           false       1
5           1234    Kukla   ���ӭn�ˬd���~��ơC           false       1
6           1234    Ollie   ��A�o�O�� bug�C              false       1
7           1234    Fran    �Х[�W�ˬd�C                   false       1
8           1234    Kukla   �w�ץ��������C               false       1
9           1234    Fran    �o�q�������C                   true        1

-- comment_paths ��
ancestor  descendant  path          path_length
------------------------------------------------
1         1          1/            0
1         2          1/2/          1
1         3          1/2/3/        2
1         4          1/2/3/4/      3
1         5          1/5/          1
1         6          1/5/6/        2
1         7          1/5/6/7/      3
1         8          1/5/6/7/8/    4
1         9          1/5/9/        2
2         2          2/            0
2         3          2/3/          1
2         4          2/3/4/        2
...

-- comment_versions ��
comment_id  version  comment_text                     modified_at           modified_by
---------------------------------------------------------------------------------------
2           1        �ڷQ���ӬO null pointer�C       2024-01-01 10:00:00  Ollie
2           2        �T�{�O memory leak�C            2024-01-01 10:30:00  Ollie
9           1        �o�q�������C                    2024-01-01 11:00:00  Fran
```

### �d�߾ާ@

1. �d�ߧ��㪺���׾𪬵��c�G
```sql
SELECT c.*, p.path, p.path_length 
FROM comments c
JOIN comment_paths p ON c.comment_id = p.descendant
WHERE p.ancestor = 1 AND c.is_deleted = false
ORDER BY p.path;
```

2. �d�߯S�w���������סG
```sql
SELECT c.*, cv.comment_text, cv.modified_at
FROM comments c
JOIN comment_versions cv ON c.comment_id = cv.comment_id
WHERE c.comment_id = 2 AND cv.version = 1;
```

3. �d�߯S�w�`�ת����סG
```sql
SELECT c.* 
FROM comments c
JOIN comment_paths p ON c.comment_id = p.descendant
WHERE p.ancestor = 1 AND p.path_length = 2;
```

### �ק�ާ@

1. �s�W���סG
```sql
-- ���s�W����
INSERT INTO comments (bug_id, author, comment_text)
VALUES (1234, 'Kukla', '�s������');

-- �إ߸��|���Y
INSERT INTO comment_paths (ancestor, descendant, path, path_length)
SELECT t.ancestor, LAST_INSERT_ID(),
       CONCAT(t.path, LAST_INSERT_ID(), '/'),
       t.path_length + 1
FROM comment_paths t
WHERE t.descendant = 3
UNION ALL
SELECT LAST_INSERT_ID(), LAST_INSERT_ID(), 
       CONCAT(LAST_INSERT_ID(), '/'), 0;
```

2. �s����סG
```sql
-- �x�s�ª���
INSERT INTO comment_versions (comment_id, version, comment_text, modified_by)
SELECT comment_id, version, comment_text, author
FROM comments
WHERE comment_id = 2;

-- ��s����
UPDATE comments 
SET comment_text = '�s�����e',
    version = version + 1
WHERE comment_id = 2;
```

3. ���ʤl��G
```sql
-- ��s�Ҧ��l�`�I�����|
UPDATE comment_paths
SET path = REPLACE(path, '/2/', '/5/')
WHERE path LIKE '/1/2/%';

-- ��s���Y��
UPDATE comment_paths cp1
JOIN comment_paths cp2 ON cp1.descendant = cp2.descendant
SET cp1.ancestor = 5
WHERE cp2.path LIKE '/1/2/%';
```

### �D�n�x���I

1. **���|���@**
   - �ݭn�P�B���@ Closure Table �M���|��T
   - �j�q�ƾھE���ɮį���t
   - �ݭn�T�O���|���@�P��

2. **�������������**
   - �������v�i����Τj�q�x�s�Ŷ�
   - �ݭn�A���M�z����
   - �����^�_�����{��������

3. **���������ҳB�z**
   - ���Ʈw���@�P�ʺ��@
   - �������ưȪ��B�z
   - ��ƦP�B���ɮĩ�

### �p��

�u�I�G
1. ���X�F��ؤ�k���u�աA�d�߮įప
2. �䴩���㪺��������
3. ���|�ާ@�F��
4. �A�X�������𪬵��c�ާ@

���I�G
1. �x�s�Ŷ��ݨD���j
2. ��@�����װ�
3. ���@��������
4. �ݭn�B�~���P�B����

## 2. Nested Sets with Closure Table

### ����
�o�زV�X�覡���X�F Nested Sets �����Ĥl��d�ߩM Closure Table �����Y���@�u�աC�S�O�A�X�ݭn�W�c�i��𪬵��c���R���]�ݭn�O���ק�u�ʪ������C

### ��Ƶ��c
```sql
CREATE TABLE comments (
    comment_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    nsleft        INTEGER NOT NULL,
    nsright       INTEGER NOT NULL,
    bug_id        BIGINT UNSIGNED NOT NULL,
    author        TEXT NOT NULL,
    comment_text  TEXT NOT NULL,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    is_deleted    BOOLEAN DEFAULT FALSE
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

### �w����ƽd��
```
-- comments ��
comment_id  nsleft  nsright  author  comment_text                    is_deleted
--------------------------------------------------------------------------------
1           1       18       Fran    ������|���o�� bug�H           false
2           2       7        Ollie   �ڷQ���ӬO null pointer�C      false
3           3       4        Fran    ���O�A�ڽT�{�L�F�C             false
4           5       6        Ollie   �T�{�O memory leak�C           false
5           8       17       Kukla   ���ӭn�ˬd���~��ơC           false
6           9       14       Ollie   ��A�o�O�� bug�C              false
7           10      13       Fran    �Х[�W�ˬd�C                   false
8           11      12       Kukla   �w�ץ��������C               false
9           15      16       Fran    �o�q�������C                   true

-- comment_paths ��
ancestor  descendant  distance
--------------------------------
1         1           0
1         2           1
1         3           2
1         4           3
2         2           0
2         3           1
2         4           2
3         3           0
3         4           1
...
```

[�~��ɥR Nested Sets with Closure Table�BTemporal Closure Table�BMaterialized Path with Nested Sets �M�������𪬵��c���ԲӤ��e...]

## �į���

| �ާ@���� | Closure+Path | Nested+Closure | Temporal | MP+NS | ������ |
|---------|-------------|----------------|----------|-------|--------|
| Ū���l�� | O(1) | O(1) | O(1) | O(1) | O(1)* |
| �g�J�`�I | O(1) | O(n) | O(1) | O(n) | O(1)* |
| ���ʤl�� | O(n) | O(n) | O(1) | O(n) | O(n) |
| �ɶ��d�� | O(n) | O(n) | O(1) | O(n) | O(n) |

* ���]���A�����ϵ����M�֨�����

## ��@��ĳ

1. **�t�λݨD���R**
   - ������ƶq�j�p�M�W���Ͷ�
   - ���RŪ�g�ާ@���
   - �Ҽ{�X�i�ʻݨD
   - �������@����

2. **�޳N�﫬�Ҷq**
   - ��Ʈw�S��
   - �}�o�ζ���O
   - ��B�ݨD
   - ��������

3. **�u�Ƶ���**
   - �A���֨�����
   - �妸�B�z�̨Τ�
   - �D�P�B�B�z
   - ��Ƥ�������

## �`��

�U�ضi���𪬵��c��k���A�γ����G

1. **Closure Table with Path Enumeration**
   - �A�X�ݭn���į�d�ߩM������v�O�����t��
   - �S�O�A�Ω�ݭn�������|�ާ@������
   - ��ĳ�Ω󤤫��W�ҡBŪ�����h������
   - �A�X���~�����Ψt��

2. **Nested Sets with Closure Table**
   - �A�X�ݭn�W�c�l��d�ߪ�����
   - �S�O�A�Ω���R�A����´�[�c
   - ��ĳ�Ω��Ƶ��c�۹�í�w���t��
   - �A�X������R�t��

3. **Temporal Closure Table**
   - �A�X�ݭn���㪩������t��
   - �S�O�A�Ω�ݭn�]�ְl�ܪ�����
   - ��ĳ�Ω�j�ո�Ƨ���ʪ�����
   - �A�X���ĩΪk�W��`�t��

4. **Materialized Path with Nested Sets**
   - �A�X�ݭn���ĸ��|�ɯ誺����
   - �S�O�A�Ω��ɮרt��������
   - ��ĳ�Ω�h�Ÿ��`�����c
   - �A�X���޲z�t��

5. **�������𪬵��c**
   - �A�X�W�j�W������
   - �S�O�A�Ω�ݭn�����X�i������
   - ��ĳ�Ω󰪨ֵo�t��
   - �A�X���ݪA�Ȭ[�c

��@�����Ҽ{�H�U����]���G

1. **��ƯS�ʵ���**
   - ��ƶq���W�һP�����t��
   - ��ƪ��ܰ��W�v
   - �d�߼Ҧ����R
   - ���v��ƫO�d�ݨD

2. **�t�ά[�c�Ҷq**
   - ����Τ��������p
   - ��Ʈw�﫬
   - �X�i�ʻݨD
   - �į�n�D

3. **���@�P�ʱ�**
   - �w�����@����
   - �į�ʱ�����
   - ���`�B�z�y�{
   - ��Ƥ@�P���ˬd

4. **�~�ȻݨD��X**
   - �ŦX�~�Ȭy�{
   - �����ϥΪ�����
   - �䴩���ӥ\���X�R
   - �����įq����

## �i���u�ƫ�ĳ

1. **�֨�����**
   - ��@�h�h�ŧ֨�����
   - �w��������|�i��w�֨�
   - �ϥΤ������֨��t��
   - �إߧ֨���s����

2. **�į�ծ�**
   - ���޳̨Τ�
   - �d�߮į�վ�
   - ��Ƥ�������
   - �D�P�B�B�z����

3. **�i�a�ʫO��**
   - ��@��Ƴƥ�����
   - �إ߬G�ٴ_��y�{
   - �ʱ�ĵ���t��
   - �w�����O����

## ��@�d��

�H�U���Ѥ@�ӹ�ڪ���@�d�ҡA��X�F�h�ضi���𪬵��c���u�I�G

```sql
-- �֤߸�ƪ�
CREATE TABLE tree_nodes (
    node_id       BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name          VARCHAR(255) NOT NULL,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    version       INT NOT NULL DEFAULT 1,
    is_deleted    BOOLEAN DEFAULT FALSE
);

-- ���|���Y��
CREATE TABLE node_paths (
    ancestor      BIGINT UNSIGNED NOT NULL,
    descendant    BIGINT UNSIGNED NOT NULL,
    path          VARCHAR(1000),
    distance      INT NOT NULL,
    valid_from    TIMESTAMP NOT NULL,
    valid_to      TIMESTAMP,
    
    PRIMARY KEY(ancestor, descendant, valid_from),
    FOREIGN KEY (ancestor) REFERENCES tree_nodes(node_id),
    FOREIGN KEY (descendant) REFERENCES tree_nodes(node_id)
);

-- ���������
CREATE TABLE node_versions (
    node_id       BIGINT UNSIGNED NOT NULL,
    version       INT NOT NULL,
    name          VARCHAR(255) NOT NULL,
    modified_at   TIMESTAMP NOT NULL,
    modified_by   VARCHAR(255) NOT NULL,
    
    PRIMARY KEY(node_id, version),
    FOREIGN KEY (node_id) REFERENCES tree_nodes(node_id)
);

-- ������T��
CREATE TABLE node_shards (
    node_id       BIGINT UNSIGNED NOT NULL,
    shard_key     INT NOT NULL,
    shard_type    VARCHAR(50) NOT NULL,
    
    PRIMARY KEY(node_id),
    FOREIGN KEY (node_id) REFERENCES tree_nodes(node_id)
);
```

�o�ӳ]�p��X�F�G
- Closure Table �����Ĭd��
- Path Enumeration �����|�ɯ�
- Temporal ����������
- ���������X�i�䴩

## �̨ι���ĳ

�b��@�i���𪬵��c�ɡA����`�H�U�̨ι��G

1. **�`�Ǻ��i**
   �q�򥻹�@�}�l�A�ھڹ�ڻݨD�v�B�K�[�i���\��C�o�˥i�H���C��@���I�A�åB��e�����@�M�����C

2. **�ҲդƳ]�p**
   �N���P�\��ҲդơA�Ҧp�N���|�B�z�B��������B�����޿赥���}��@�C�o�˥i�H�����N�X���ΩʩM���@�ʡC

3. **�į��u��**
   �b�]�p���q�N�n�Ҽ{�į���D�A�]�A�G
   - �A�����޵���
   - �d���u��
   - �֨�����
   - �D�P�B�B�z

4. **�ʱ��P���@**
   �إߧ��㪺�ʱ��M���@����A�]�A�G
   - �į�ʱ�
   - ���~�l��
   - ��Ƥ@�P���ˬd
   - �w�����@�p��

## ���Ӯi��

�𪬵��c����@�޳N���b���_�t�i�A���ӥi�઺�o�i��V�]�A�G

1. **�϶����X**
   - �Q�ΰ϶���޳N�T�O��Ƥ��i�y��
   - ��{����������ƦP�B
   - ���ѧ�w������������

2. **AI���U�u��**
   - �ϥξ����ǲ߹w���X�ݼҦ�
   - ����֨�����
   - �۰ʮį�ծ�

3. **���ݭ�ͤ䴩**
   - ��n���e���Ƥ䴩
   - �L���A���[�c��X
   - �۰��X�i��O

4. **�Y�ɨ�@�\��**
   - �h�ϥΪ̧Y�ɽs��
   - �Ĭ�ѨM����
   - ��ɳq���t��

## �`��

�i���𪬵��c����@�O�@�ӽ������ȱo��ꪺ���C�q�L�X�z���]�p�M��@�A�i�H�إߤ@�ӱj�j�B�i�X�i�B���Ī��t�ΡC����b��G

1. **�z�ѻݨD**
   �`�J�z�ѷ~�ȻݨD�M�޳N����A��ܳ̾A�X����@�覡�C

2. **���Ũ���**
   �b�į�B�����סB���@�ʤ�����쥭���I�A���L�׳]�p�]���L��²�ơC

3. **�����u��**
   �O���t�Ϊ��i�X�i�ʡA�ھڹ�ڹB�污�p�����i�M�u�ơC

4. **�ζ���O**
   �Ҽ{�}�o�ζ����޳N��O�M���@��O�A��ܦX�A���޳N��סC

�̫�A�S���@�ӧ������ѨM��ׯ�A�Ω�Ҧ������A����b��ھڨ���ݨD��ܦX�A����@�覡�A�æb��@�L�{�������i�M�u�ơC