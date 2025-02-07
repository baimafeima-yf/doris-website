---
{
    "title": "NULLIF",
    "language": "zh-CN"
}
---

<!-- 
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

## nullif
## 描述
## 语法

`nullif(expr1, expr2)`


如果两个参数相等，则返回NULL。否则返回第一个参数的值。它和以下的 `CASE WHEN` 效果一样

```
CASE
     WHEN expr1 = expr2 THEN NULL
     ELSE expr1
END
```

## 举例

```
mysql> select nullif(1,1);
+--------------+
| nullif(1, 1) |
+--------------+
|         NULL |
+--------------+

mysql> select nullif(1,0);
+--------------+
| nullif(1, 0) |
+--------------+
|            1 |
+--------------+
```
### keywords
NULLIF
