---
layout:     post   				    # 使用的布局（不需要改）
title:      Python 连接 Google BigQuery 进行SQl查询并下载数据 				# 标题 
subtitle:   Python 连接 Google BigQuery 进行SQl查询并下载数据         #副标题
date:       2019-12-15 				# 时间
author:     XF 						# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Python
    - BigQuery
---

## 1.官方API文档

直接查看Google BigQuery的官方API文档：(需fan墙)

https://googleapis.dev/python/bigquery/latest/index.html

该文档给出的例子很清楚

下面给出程序。

## 2.程序

先安装 BigQuery 的 Python 客户端

~~~python
pip install google-cloud-bigquery
~~~

**官方给出的例程：**

~~~python
from google.cloud import bigquery

client = bigquery.Client()

# Perform a query.
QUERY = (
    'SELECT name FROM `bigquery-public-data.usa_names.usa_1910_2013` '
    'WHERE state = "TX" '
    'LIMIT 100')
query_job = client.query(QUERY)  # API request
rows = query_job.result()  # Waits for query to finish

for row in rows:
    print(row.name)
~~~



**我的代码**：

```python
# -*- coding: utf-8 -*- 
# @Time : 2019/12/15 16:01 
# @Author : Xufeng 
# @File : Google_BigQuery_api.py
# @Desc : Google BigQuery API

from google.cloud import bigquery
import openpyxl
import datetime

BQ_TIMEOUT = 240
#max is 100 per project
BQ_THREAD_DRY_RUN_LIMIT = 20
BQ_QUERY_SLEEP_SECONDS = 1

# 认证文件，在BigQuery生成认证文件
AUTH_JSON_FILE_PATH = './1234567.json'

# Client 认证
def bq_InitConnection():
    return bigquery.Client.from_service_account_json(AUTH_JSON_FILE_PATH)

# 向BigQuery请求数据
def bq_query(SQL):
    client = bq_InitConnection()

    # print(bqSql)
    bqJob = client.query(SQL)

    bqList = list(bqJob.result(timeout=BQ_TIMEOUT))  # Waits for job to complete.
    return bqList

# 处理BigQuery的数据，返回处理好的数据
def query_Collect(SQL):
    # print("yesterday is:", getYesterdayFbStr())
    retList = []
    bqListRet = bq_query(SQL)
    i = 0
    for listItem in bqListRet:
        item = list(listItem)
        retList.append(item)
        i += 1
    return retList

# 传入SQL查询命令，返回Sheet  (ts,uid)，此函数要根据查询得到的数据格式进行修改
def query_SaveSheet_TsUid(SQL, SheetTitle, FilePath):
    # 创建Excel的Sheet
    wb = openpyxl.Workbook()
    ws = wb.active
    # 向BigQuery查询
    bqListRet = bq_query(SQL)
    # 向Sheet第一行添加
    ws.append(["ts", "uid"])
    # 查询到的数据存入Excel
    i = 0
    for listItem in bqListRet:
        item = list(listItem)
        # 格式进行转换
        item[0] = datetime.datetime.strptime(str(item[0]), '%Y-%m-%d %H:%M:%S')
        item[1] = int(item[1])
        ws.append(item)
        i += 1
    # Sheet 保存
    ws.title = SheetTitle
    wb.save(FilePath)

    print(FilePath + "  has been downloaded")

if __name__ == '__main__':
    SQL = """
		此处为SQl查询语句
		"""
    # 直接查询
    res = query_Collect(SQL)
    print(res)
    print(res[0][0])
    # 将查询保存为Excel
    query_SaveSheet_TsUid(SQL, SheetTitle="Online", FilePath=".\Online.xlsx")
```

### 2.1 函数说明

使用 query_Collect(SQL) 函数，可以返回SQL命令查询的结果。

使用 query_SaveSheet_TsUid(SQL, SheetTitle, FilePath) 函数，可将查询到的结果保存为Excel文件，但需要根据自己查询到的数据进行相应的修改。此处查询的数据 第一列为时间，第二列为 int 型数据。

### 2.2 认证

~~~python
# Client 认证
def bq_InitConnection():
    return bigquery.Client.from_service_account_json(AUTH_JSON_FILE_PATH)
~~~

此函数为Client认证，需要根据自己的项目生成认证文件，[具体见官方文档](https://cloud.google.com/bigquery/docs/reference/libraries?hl=zh-cn#client-libraries-install-python)。

## 3.注意

**最重要的，别忘了把梯子设置成全局。（一把辛酸泪）**







