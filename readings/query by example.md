# Query By Example(QBE)

# 什么是`QBE`

> **Query By Example(QBE)** is a database query language for relational database. -Wiki

QBE由IBM开发

QBE Example:

```sql
.....Name: Bob
..Address:
.....City:
....State: TX
..Zipcode:
```
转为SQL

```sql
SELECT * FROM Contacts WHERE Name='Bob' AND State='TX';
```

## 资料
[QBE](http://pages.cs.wisc.edu/~dbbook/openAccess/thirdEdition/qbe.pdf)