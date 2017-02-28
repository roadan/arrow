#Apache Arrow Java API – Getting StartedApache Arrow is an in-memory data structures [specification](https://github.com/apache/arrow/tree/master/format) that can be used by multiple systems to provide data interoperability without the need to perform steps such as serialization and data movement.The Arrow format specify a flexible data model that supports both primitive and complex types that handles flat tables as well as real-world hierarchical data. ##Basics - Working with ArrowRecordBatchArrow structures are defined in a table-like structure. In the Java API, these are represented by the `ArrowRecordBatch` class. `ArrowRecordBatch` are composed of [value vectors](https://drill.apache.org/docs/value-vectors/) which represent Arrow arrays in the [Arrow format](https://github.com/apache/arrow/blob/master/format/Layout.md). 
**Note:** value vectors come out of the Apache Drill implementation, hence the link to the drill documentation.###Value Vectors Before we create an `ArrowRecordBatch`, we need to create our value vectors. The following code creates and allocates two vectors, one is an `IntVector` that will store an id for whatever it is we intend to store in our row batch and the other is a ` VarBinaryVector` to store it’s name:```java
RootAllocator allocator = new RootAllocator(Long.MAX_VALUE);

IntVector idVector = new IntVector("id", allocator);
VarCharVector nameVector = new VarCharVector("name", allocator);
        
idVector.allocateNew(1000);
nameVector.allocateNew(1000 * 8, 1000);
```
In the above code there are a couple of things to note: - The root allocator will be used by both of the vectors to allocate the underlying ArrowBufs. ArrowBufs are contiguous memory regions, allocated for storing the Arrow arrays. - The `allocateNew` method allocates a new or reused underlying buffer for the vector. The buffer size can be larger that the requested size, depending on underlying buffer rounding size, or the size of the recycled buffer, however, buffer's capacity will be set to the limit passed to the `allocateNew` method (unless values are set using the [`setSafe`](#setsetsafe) method). - Arrow allocates the vectors as a Contiguously directly in unmanaged memory; this is the role of the `RootAllocator`. Because vectors are allocated Contiguously in memory, the allocator needs to know the exact size of buffer to allocate. This is why value vectors define an `allocateNew` method that defines the count of items, and the total bytes to allocate.  - Neither `IntVector` nor `VarBinaryVector` can store null values in thier underlaying buffer. If you need to store null values you should use the `NullableIntVector` and `NullableVarBinaryVector` vector types that support null values. The reason behind that, that arrow arrays (which are represented by vectors in the Java API) that contain null values, have to maintain null bitmaps (also called validity bitmaps) as part of the Arrow specification. Using the non-nullable versions can save to (arguably minor) overhead of managing null bitmaps. For more information about null representation in Arrow,  refer to the [Physical memory layout specifications]( https://github.com/apache/arrow/blob/master/format/Layout.md)### Writing Data to Value Vectors
The next step for us, will be writing to our vectors. To do so, we first need to create mutetors which can than be used to set entries in the vectors by using an index:

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
    nameMutetor.setSafe(i, String.format("name %d", i).getBytes());
}
```

Most value vector types also have a public `set` function (`VarCharVector` is a good example of a value vector that has set as a protected method), which will not allocate a new buffer, but rather throw an error when going over the allocated buffer size, so in contradiction to the above code, the following code will fail, throwing `java.lang.IndexOutOfBoundsException`

```java
IntVector idVector = new IntVector("id", allocator);
idVector.allocateNew(500);
IntVector.Mutator idMutetor = idVector.getMutator();

for (int i = 0; i < 1000; i++) {
    idMutetor.set(i, i);
    nameMutetor.setSafe(i, String.format("name %d", i).getBytes());
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