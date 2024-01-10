# Example to Integrate Bytewax and Proton together
[proton.py](https://github.com/timeplus-io/proton-python-driver/blob/develop/example/bytewax/proton.py) is a Bytewax output/sink for [Timeplus Proton](https://github.com/timeplus-io/proton) streaming database.

Inspired by https://bytewax.io/blog/polling-hacker-news, you can call Hacker News HTTP API with Bytewax and send latest news to Proton for SQL-based analysis, such as

```sql
select raw:id as id, raw:by as by, to_time(raw:time) as time, raw:title as title from hn
```

## Run with Docker Compose
Simply run `docker compose up` in this folder and it will start both Proton and a custom image that leverages bytewax to call Hacker News API and send data to Proton.

## Run without Docker


```shell
python3.10 -m venv py310-env
source py310-env/bin/activate
#git clone and cd to this proton-python-driver/example/bytewax folder
pip install bytewax==0.18
pip install requests 
pip install proton-driver

python -m bytewax.run hackernews
```
It will load new items every 15 second and send the data to Proton.

## How it works

```python
flow.output("out", ProtonSink(HOST, "hn"))
```
`hn` is an example stream name. The `ProtonOutput` will create the stream if it doesn't exist
```python
class _ProtonSinkPartition(StatelessSinkPartition):
    def __init__(self, stream: str, host: str):
        self.client=client.Client(host=host, port=8463)
        self.stream=stream
        sql=f"CREATE STREAM IF NOT EXISTS `{stream}` (raw string)"
        logger.debug(sql)
        self.client.execute(sql)
```
and batch insert data
```python
def write_batch(self, items):
        rows=[]
        for item in items:
            rows.append([item]) # single column in each row
        sql = f"INSERT INTO `{self.stream}` (raw) VALUES"
        # logger.debug(f"inserting data {sql}")
        self.client.execute(sql,rows)
```

```python
class ProtonSink(DynamicSink):
    def __init__(self, stream: str, host: str):
        self.stream = stream
        self.host = host if host is not None and host != "" else "127.0.0.1"


    def build(self, worker_index, worker_count):
        """See ABC docstring."""
        return _ProtonSinkPartition(self.stream, self.host)
```
