#Apache Arrow Java API – Getting Started

RootAllocator allocator = new RootAllocator(Long.MAX_VALUE);

IntVector idVector = new IntVector("id", allocator);
VarCharVector nameVector = new VarCharVector("name", allocator);
        
idVector.allocateNew(1000);
nameVector.allocateNew(1000 * 8, 1000);
```
In the above code there are a couple of things to note:


```java
IntVector.Mutator idMutetor = idVector.getMutator();
VarCharVector.Mutator nameMutetor = nameVector.getMutator();

for (int i = 0; i < 1000; i++) {
    idMutetor.setSafe(i, i);
    nameMutetor.setSafe(i, String.format("name %d", i).getBytes());
}
```

### <a name="setsetsafe"></a>setSafe vs. set

In the above example we have used the `setSafe` method to set a value of specific indices in our vector. As it's name implies, `setSafe` is a safe way to set the value of a specific entry in a vector, and buy safe, we really mean that if there is no more space in the underlying `ArrowBuf`, a new buffer will be allocated by the allocator, twice the size of the original buffer and all data will be copied to the new buffer. This is done as long as the new buffer doesn't surpass the maximum limitation of the `RootAllocator` which is `Long.MAX_VALUE` Bytes. For this reason the following code will execute successfully, but will be wasteful of memory and create two buffers and copy data between them:

```java
IntVector idVector = new IntVector("id", allocator);
idVector.allocateNew(500);
IntVector.Mutator idMutetor = idVector.getMutator();

for (int i = 0; i < 1000; i++) {
    idMutetor.setSafe(i, i);
}
```

Most value vector types also have a public `set` function (`VarCharVector` is a good example of a value vector that has set as a protected method), which will not allocate a new buffer, but rather throw an error when going over the allocated buffer size, so in contradiction to the above code, the following code will fail, throwing `java.lang.IndexOutOfBoundsException`

```java
IntVector idVector = new IntVector("id", allocator);
idVector.allocateNew(500);
IntVector.Mutator idMutetor = idVector.getMutator();

for (int i = 0; i < 1000; i++) {
    idMutetor.set(i, i);
}
```

###Creating an ArrowRecordBatch

Our last step towards having a columnar, table-like representation of our data, is adding the value vectors to an `ArrowRecordBatch`. To do so, we need to provide the `ArrowRecordBatch` constructor, a list of both underlying Arrow buffers, a list of `ArrowFieldNode` representing each of the fields, and the number of items in the batch:

```java
// creating a list of the underlying ArrowBufs from both vectors
List<ArrowBuf> bufs = new ArrayList<>(1000);
bufs.add(idVector.getBuffer());
bufs.add(nameVector.getBuffer());

// each ArrowFieldNode represent a filed in the ArrowRecordBatch, where the first
// parameter is the number of values and the second one is the number of nulls
List<ArrowFieldNode> nodes = new ArrayList<>(1000);
nodes.add(new ArrowFieldNode(1000, 0));
nodes.add(new ArrowFieldNode(1000, 0));

ArrowRecordBatch recordBatch = new ArrowRecordBatch(1000, nodes, bufs);
```

### Reding from ArrowRecordBatch 	