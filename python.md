# 记录python一些错误

## TypeError: datetime.datetime(2016, 12, 14, 11, 11, 12, 702000) is not JSON serializable
> mongodb的serverStatus信息里有datetime,json.dumps时出的错误.

解决:
```
class DatetimeEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime.datetime):
            return obj.strftime('%Y-%m-%d %H:%M:%S')
        elif isinstance(obj, datetime.date):
            return obj.strftime('%Y-%m-%d')
        return json.JSONEncoder.default(self, obj)
json.dumps({
            "server_status": server_status
        }, cls=DatetimeEncoder)

```

参考:
- [-json-serialization-of-datetime](http://stackoverflow.com/questions/10721409/why-does-json-serialization-of-datetime-objects-in-python-not-work-out-of-the-bo)