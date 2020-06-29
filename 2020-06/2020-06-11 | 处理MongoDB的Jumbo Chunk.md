- [原文链接](https://www.percona.com/blog/2020/06/11/dealing-with-jumbo-chunks-in-mongodb/)


# 处理MongoDB的Jumbo Chunk
在这篇博客中，我们将讨论如何处理MongoDB中的jumbo chunks。

场景是：您是一个MongoDB DBA，每天的第一个任务是从集群中迁移一个分片。一开始听起来很可怕，但其实很简单。您可以使用如下简单的命令：
```
db.runCommand( { removeShard: "server1_set6" } )
```

MongoDB会自动找到数据块和数据库，并在所有服务器上对它们进行平衡。您可以放心地去睡觉。

第二天早上当您醒来，检查这个特定的分片的状态时，发现进程卡住了：
```
"msg" : "draining ongoing",
"state" : "ongoing",
"remaining" : {
"chunks" : NumberLong(3),
"dbs" : NumberLong(0)
```
由于某些原因，有三个块没有被迁移，所以`removeShard`命令被阻塞了！现在，您该如何做？

## 找到未被迁移的块
需要连接到`mongos`，检查目录
```
mongos> use config
switched to db config
mongos> db.chunks.find({shard:"server1_set6"})
```
输出显示有三个块，并有`_id`的最小值和最大值，以及它们所属的namespace。输出的最后一部分是我们真正需要检查的内容：
```
{
[...]
"min" : {
"_id" : "17zx3j9i60180"
},
"max" : {
"_id" : "30td24p9sx9j0"
},
"shard" : "server1_set6",
"jumbo" : true
}
```

因此，块被标记为"jumbo"。我们得到的原因是平衡器无法迁移这些chunk。

## Jumbo Chunks & 如何处理Jumbo Chunks
什么是"jumbo chunk"？它是一个大小超过配置参数中指定`chunk`最大大小的块，默认值为64MB。当值超出限制时，平衡器不会移动它。

这仅是概念而已。作为一个具体的实现，在`splitChunk`命令发现它不能将一个范围文档切割成小于设置定义的块大小时，它会被标记为**jumbo**。`splitChunk`命令通常由平衡器的`moveChunk`或后台自动切分进程执行。

假设您有一个分片键
**{“surname”: 1, “given_name”: 1}**，
这是一个非唯一的元组，因为没有主键。如果您在
**{“surname”: “Smith”, “given_name”: “John”, …}**
上有100000个文档，那么没有办法将它们分开。因此，块将和那100000个文档一样大。

## 如何清除"Jumbo"标记
在4.0.15之后的版本，可以使用`clearJumboFlags`命令。

在之前的版本中，您需要在`config`库中手工删除文档的"jumbo"字段，`congfig`库中定义了mongos节点和nodes节点指向的chunk范围。
```
db.getSiblingDB("config").chunks.update(
  {"ns": <your_sharded_user_db.coll>, "jumbo": true}, 
  {$unset: { "jumbo": "" }}
);
```

## 处理不可分割的问题
从MongoDB4.4版本开始，您可以使用`refineCollectionShardKey`命令将另一个字段作为后缀字段添加到分片键。如将
**{“surname”: 1, “given_name”: 1}**
改为
**{“surname”: 1, “given_name”: 1, “date_of_birth”: 1}**。

如果您使用的是MongoDB4.2或者之前的版本，块如果超出块大小设置，将无法移动。在上文描述的无法解决的问题中，您需要执行以下操作之一来完成分片迁移，这样您就可以最终完成`removeShard`。
- 增大块大小设置之后，迁移jumbo块。
	- 使用`dataSize`命令遍历所有的jumbo chunk，以找出最大大小。
	- 将`chunksize`的大小设置超出这个大小。
	- 清除**jumbo**标记（参见上面的小节）
	- 再次开始迁移，等待它移动jumbo chunk。如果您希望尽早完成，可以使用`sh.moveChunk()`命令。
	- 不要忘记之后将`chunksize`更改回来。
- 暂时删除该数据。在分片迁移完成后重新插入副本。
	- 在将空chunk迁移到另一个分片之前，您仍然需要清除jumbo标记（参见上文）

## 当MongoDB中Jumbo Chunk变小时
即使文档被删除或者收缩，**jumbo**标记一旦被搭上也不会被删除，因此块大小默认为64MB。参照上文您可以手工清除**jumbo**标记并重试。

理论上，您不需要手动分割，但是如果您想要加快速度并确认块可以切分成足够小的大小，请参阅如何使用`sh.splitAt()`命令的官档。
