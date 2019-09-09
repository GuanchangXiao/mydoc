# ElasticSearch（二）建立索引

**1、下载 [accounts.json](https://github.com/elastic/elasticsearch/blob/master/docs/src/test/resources/accounts.json?raw=true) 文件,json格式如下:**

```json
{
    "account_number": 0,
    "balance": 16623,
    "firstname": "Bradshaw",
    "lastname": "Mckenzie",
    "age": 29,
    "gender": "F",
    "address": "244 Columbus Place",
    "employer": "Euron",
    "email": "bradshawmckenzie@euron.com",
    "city": "Hobucken",
    "state": "CO"
}
```

**2、将json文件链接到索引bank**

```shell
curl -H "Content-Type: application/json" -XPOST "localhost:9200/bank/_bulk?pretty&refresh" --data-binary @accounts.json
curl "localhost:9200/_cat/indices?v"
```

**3、查看索引信息**

```shell
GET /_cat/indices?v
```

