# SQL完全攻略：大学履修登録システム編

この資料は、大学の履修システムをモデルに、データベース定義(DDL)、操作(DML)、検索(DQL)の基本から応用までを網羅した学習ノートです。

---

## 0. データベースの設計図 (スキーマ定義)

今回扱うデータは、以下の3つのテーブルで構成されています。

### ① 学生テーブル (`student`)
学生の基本情報を管理します。
| 列名 | データ型 | 制約 | 説明 |
| :--- | :--- | :--- | :--- |
| **student_id** | `VARCHAR(10)` | **PK**, NOT NULL | 学籍番号（主キー・必ず入力） |
| **name** | `VARCHAR(30)` | | 名前 |
| **city** | `VARCHAR(30)` | | 住んでいる都市 |
| **age** | `INTEGER` | | 年齢 |

### ② 科目テーブル (`course`)
大学で開講されている授業の情報です。
| 列名 | データ型 | 制約 | 説明 |
| :--- | :--- | :--- | :--- |
| **course_id** | `VARCHAR(10)` | **PK**, NOT NULL | 科目ID（主キー・必ず入力） |
| **course_name** | `VARCHAR(50)` | NOT NULL | 授業名（必ず入力） |
| **lecturer** | `VARCHAR(30)` | | 担当教員 |
| **credit** | `INTEGER` | | 単位数 |

### ③ 成績テーブル (`grade`)
「誰が」「どの科目を」履修し、「何点取ったか」を記録する、学生と科目の**つなぎ役**です。
| 列名 | データ型 | 制約 | 説明 |
| :--- | :--- | :--- | :--- |
| **student_id** | `VARCHAR(10)` | **FK**, **PK** | 学籍番号（外部キー） |
| **course_id** | `VARCHAR(10)` | **FK**, **PK** | 科目ID（外部キー） |
| **score** | `INTEGER` | | 点数 |

---

## 1. DDL: テーブルを作成する (CREATE)

データを格納するための「箱」を作ります。データの整合性を保つための「制約（ルール）」が重要です。

### 基本的な作成と制約 (student)
`student` テーブルを作成します。学籍番号は重複してはいけないため `PRIMARY KEY` を設定します。

```sql
CREATE TABLE student(
  student_id VARCHAR(10) NOT NULL,
  name VARCHAR(30),
  city VARCHAR(30),
  age INTEGER,
  PRIMARY KEY (student_id)
);
```


### 制約の強化 (UNIQUE / NOT NULL)

名前の入力を必須にし、さらに重複を禁止する場合の例です [cite: 19, 28-29]。

```sql
CREATE TABLE student(
  student_id VARCHAR(10) NOT NULL,
  name VARCHAR(30) NOT NULL UNIQUE, /* NULL禁止 かつ 重複禁止 */
  city VARCHAR(30),
  age INTEGER,
  PRIMARY KEY (student_id)
);
```

### 科目テーブルの作成 (course)

科目テーブルも同様に主キーと必須項目を定義します [cite: 48-53]。

```sql
CREATE TABLE course(
  course_id VARCHAR(10) NOT NULL,
  course_name VARCHAR(50) NOT NULL,
  lecturer VARCHAR(30),
  credit INTEGER,
  PRIMARY KEY (course_id)
);
```

### 外部キーによるリレーション (FOREIGN KEY)

`grade` テーブルは、実在する学生と科目しか登録できないように、他のテーブルを参照します。
[cite\_start]また、`ON DELETE CASCADE` を指定することで、学生が削除されたらその成績データも自動で消えるようにします [cite: 57-81]。

```sql
CREATE TABLE grade(
  student_id VARCHAR(10) NOT NULL,
  course_id VARCHAR(10) NOT NULL,
  score INTEGER,
  PRIMARY KEY (student_id, course_id),
  FOREIGN KEY (student_id) REFERENCES student
    ON DELETE CASCADE   /* 親テーブルから行が削除された場合、その削除された行を参照している子テーブル内の対応するすべての行も自動的に削除されます。 */
    ON UPDATE NO ACTION, /*親テーブルのキーは、子テーブルが参照している間は絶対に変えないでください  */
  FOREIGN KEY (course_id) REFERENCES course
    ON DELETE CASCADE
    ON UPDATE NO ACTION
);
```

-----

## 2\. DML: データを操作する (INSERT / UPDATE / DELETE)

### データの挿入 (INSERT)

新しいデータを追加します。

```sql
/* 全列に値を指定 [cite: 99] */
INSERT INTO student VALUES ('S6', 'Kondo', 'Otsu', 21);

/* 列を明示して指定 [cite: 105] */
INSERT INTO student (student_id, name, city, age)
VALUES ('S6', 'Kondo', 'Otsu', 21);

/* 複数行を一括挿入 [cite: 108-109] */
INSERT INTO student
VALUES ('S7', 'Sato', 'Nara', 22), ('S8', 'Yamamoto', 'Osaka', 20);

/* 一部の列のみ挿入 (未指定はNULLになる) [cite: 112-113] */
INSERT INTO student (student_id, name, city)
VALUES ('S9', 'Terao', 'Otsu');

/* 検索されたものをgradeというテーブルに挿入する*/

INSERT INTO grade (student_id, course_id)
SELECT 'S5', course_id
FROM student NATURAL JOIN grade
WHERE name = 'Takeda';
```

### データの更新 (UPDATE)

既存のデータを書き換えます。

```sql
/* J3という科目の情報を変更 [cite: 124-130] */
UPDATE course
SET course_name = 'OS',
    lecturer = 'Tomita',
    credit = credit - 2
WHERE course_id = 'J3';

/* 計算式を用いた更新 (Tanaka先生の単位を2倍に) [cite: 132-134] */
UPDATE course
SET credit = credit * 2
WHERE lecturer = 'Tanaka';
```

### データの削除 (DELETE)

データを削除します。外部キー設定により、関連するgradeデータも消える点に注意してください [cite: 137-140]。

```sql
DELETE FROM course
WHERE course_id = '1';
```

-----

## 3\. DQL: データを検索する (SELECT)

### 基本的な検索

| 目的 | SQL例 | 解説 |
| :--- | :--- | :--- |
| **選択** (Selection) | `SELECT student_id FROM grade WHERE course_id = '2' and score >= 80;` | [cite\_start]条件に合う行を取得 [cite: 143] |
| **射影** (Projection) | `SELECT DISTINCT course_id FROM grade;` | [cite\_start]重複を除いて科目IDを取得 [cite: 143] |
| **全列選択** | `SELECT * FROM student;` | [cite\_start]全ての列を取得 [cite: 143] |
| **論理演算子** | `SELECT * FROM student WHERE age >= 20 AND NOT city = 'Nara';` | [cite\_start]20歳以上 かつ 奈良以外 [cite: 143] |
| **列名変更** | `SELECT student_id AS id ...` | [cite\_start]表示名を `id` に変更 [cite: 143] |
| **四則演算** | `SELECT student_id, 100 - score FROM grade;` | [cite\_start]100点満点との差を計算 [cite: 143] |

### あいまい検索・範囲指定

| 述語 | SQL例 | 解説 |
| :--- | :--- | :--- |
| **BETWEEN** | `WHERE age BETWEEN 19 AND 21` | [cite\_start]19歳以上21歳以下 [cite: 151] |
| **IN** | `WHERE course_id IN ('11', '33', '05')` | [cite\_start]いずれかに一致 [cite: 151] |
| **NOT IN** | `WHERE student_id NOT IN ('S2', 'S3')` | [cite\_start]除外検索 [cite: 151] |
| **LIKE** | `WHERE lecturer LIKE '%to'` | [cite\_start]末尾が "to" で終わる文字列 [cite: 151] |

-----

## 4\. テーブルの結合 (JOIN)

複数のテーブルをつなぎ合わせて情報を取得します。

### 内部結合 (JOIN / NATURAL JOIN)

「単位数が5の科目」を履修している学生を探します。

```sql
/* 通常のJOIN [cite: 153] */
SELECT DISTINCT student_id FROM grade
JOIN course ON grade.course_id = course.course_id
WHERE credit = 5;

/* 自然結合 (同じ列名を自動で結合) [cite: 153] */
SELECT DISTINCT student_id FROM grade
NATURAL JOIN course
WHERE credit = 5;
```

### 高度な結合

  * [cite\_start]**交差結合 (CROSS JOIN):** 全ての組み合わせを作成（総当たり）[cite: 153]。
  * [cite\_start]**自己結合:** 同じテーブルを別名 (`AS ST1`, `AS ST2`) で結合し、同じ都市に住む学生ペアなどを探す [cite: 153]。
  * [cite\_start]**複雑な結合:** 3つ以上のテーブルを結合し、京都と奈良の学生が共通して履修している科目を特定するなどの応用 [cite: 160-161]。

-----

## 5\. 集合演算 (Set Operators)

[cite\_start]検索結果同士を足したり引いたりします。属性と定義域が同じである必要があります [cite: 168]。

  * **UNION (和集合):** A または B
  * **INTERSECT (共通集合):** A かつ B
  * **EXCEPT (差集合):** A から B を除いたもの

<!-- end list -->

```sql
/* 単位数4の科目 から Yamadaが履修している科目 を除く */
SELECT course_id FROM course WHERE credit = 4
EXCEPT
SELECT course_id FROM grade NATURAL JOIN student WHERE name = 'Yamada';
```

-----

## 6\. 副問合せ (Subqueries)

SQLの中に別のSQLを入れ子にします。

### 基本的な副問合せ

  * [cite\_start]**FROM句:** `(SELECT ... ) AS GS` のように、検索結果を一時的なテーブルとして扱う [cite: 171]。
  * [cite\_start]**WHERE句 (IN):** `IN (SELECT ...)` でリストの中にいる人を探す [cite: 171]。
  * **WHERE句 (比較):** `city = (SELECT ...)` で特定の値（S1の都市など）と比較する [cite: 171]。

### 存在確認と全称記号 (EXISTS / ALL / ANY)

より複雑な条件判定を行います。

```sql
/* NOT EXISTS: 科目J2を履修していない学生 [cite: 174] */
SELECT name FROM student
WHERE NOT EXISTS (
    SELECT * FROM grade
    WHERE student_id = student.student_id AND course_id = 'J2'
);

/* ANY: どれか一つと一致 (INと同じ) [cite: 174] */
SELECT name FROM student
WHERE student_id = ANY (
    SELECT student_id FROM grade WHERE course_id = '32'
);

/* ALL: 全てと比較して真 (履修科目が全てJ2以外 = J2未履修)  */
SELECT name FROM student
WHERE 'J2' <> ALL (
    SELECT course_id FROM grade
    WHERE student_id = student.student_id
);
```

```
```