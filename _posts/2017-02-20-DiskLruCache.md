---
layout: post
title: "DiskLruCache"
description: "About Jake Wharton's DiskLruCache"
category: android
tags: [Cache, LRU, Android]
subheading: About Jake Wharton's DiskLruCache
permalink: android/disklrucache
---

This post is about the implementation details of Jake Wharton's famous [DiskLruCache](https://github.com/JakeWharton/DiskLruCache). The reason that this cache version is so popular
is what prompted me to look inside it. Although it was developed 5 years ago, but i found it out now and it is
a very efficient and safe implementation for a Disk Cache. Its also used in OkHttp and it forms the basis of
[OkHttp's disk cache.](https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/internal/cache/DiskLruCache.java)

&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;![alt text][lru_cache]

The areas where this `DiskLruCache` really packs a punch is

1.It handles the case very efficiently where the cache or its files could corrupt itself . The obvious thing here is to delete the cache, which is not present in most of the cache implementations . We usually go on by the idea that such a case will rarely happen in our projects.

2.It maintains an in-memory LRU cache of the saved entries so we don’t have to check in the disk or DB to check for a particular cache entry every time it is accessed.

3.It can determine internally whether the value at a particular location at cache is the most recent or not. In other words, it does a versioning of the value every time an edit is made in the cache for a particular key.

4.If you start writing to a `key` in cache and something goes wrong in between, you can abort this edit operation. All the temporary values created/stored for this edit will be deleted. Even when there is an ongoing operation and and the app is killed for any reason, we can call `cache.close()` and the cache will still be in a consistent state by deleting all the temporary created values/files.

5.The cache is also safe safe for reading and writing to the same key simultaneously from two different threads.This is because the functions `get(key)` and `edit(key)` are synchronized at the function level and one thread won’t enter another function if another thread is already there in any other synchronized function of the same `DiskLruCache` object.

There are some classes/objects and other concepts that form the base for this cache. Lets go through these classes and the concepts and then we will discuss the cache operations.

## Entry object
Every entry in the cache is an `Entry` object. The `Entry` object has the following structure
{% highlight java %}
class Entry{
  String key;
  long[] lengths;
  boolean readable;
  Editor currentEditor;
  long sequenceNumber;

  File getCleanFile(index);
  File getDirtyFile(index);
}
{% endhighlight %}

-> **key** is the unique identifier for an `Entry`
-> **currentEditor**  is an `Editor` object(which is discussed below)which is not NULL if this `Entry` is being edited currently.
-> **lengths** is the number of values that are present in the cache for this particular `key`. So,we can save multiple values in the cache for a particular `key`.   
-> **readable** This flag denotes that this `Entry` has been successfully published and is now readable.This flag will be false for an `Entry` only if its being created for the first time.    
-> **sequenceNumber** This denotes the number of the most recently committed edit to this `Entry`.It is incremented every time a successful edit is made to this `Entry`. For its proper use-case please see the description of the `sequenceNumber` of `Snapshot` object.  
-> **getDirtyFile(index)** returns a temporary file(created during an edit operation,before it is renamed to the clean file when commit succeeds) at the `index` given for this `Entry`.   
-> **getCleanFile(index)** is the file at index supplied for this `Entry`,which is available for reading now after an edit has been committed successfully.


## Snapshot object  
This class is a snapshot of the `Entry` object in the cache. When the clients call `cache.get(key)`, they get a `Snapshot` object, which contains all the information needed to retrieve the cached data at `key`.
{% highlight java %}
class Snapshot  {
    String key;
    long sequenceNumber;
    InputStream[] ins;
    long[] lengths;
  }
  {% endhighlight %}

-> **key** is the key of the `Entry` which this object is a snapshot of.  
-> **ins** is the array of InputStream to our values in the cache. So ins[0] is the stream to the value present at
index zero of the `Entry` at `key`. DiskLruCache allows accessing the values in the cache only through stream.  
-> **lengths** is the same `lengths` as in the `Entry` object. It's a length of the values present in the cache for
'key'.  
-> **sequenceNumber**  As discussed in the `Entry` object, this number can be  very useful because its shared by the  `Entry` object also. So `cache.get(key)` returns a `Snapshot` object and when a `Snapshot` object is created internally,the same `sequenceNumber` of its corresponding `Entry` object to it also. So it can be determined that  if the `sequenceNumber` for a `Snapshot` doesn't match with the `sequenceNumber` for the `Entry` at a particular
`key`, then that `Snapshot` is stale. It means someone has edited the `Entry` in between for that `key` and incremented this number.  

## Editor object
Every `Entry` object wraps an `Editor` object in itself. If an `Entry` at a `key` is being actively edited, then
this `Editor` object is NOT NULL for it. So this object is the most useful thing in preventing multiple edits
simultaneously to a single `Entry`.
{% highlight java %}
class Editor {
    Entry entry;
    boolean[] written;
    boolean hasErrors;
    boolean committed;

    InputStream newInputStream(index);
    OutputStream newOutputStream(index);

  }
{% endhighlight %}

-> **entry** is the `Entry` object where this `Editor` object is working upon.    
-> **written** this flag takes care of the fact that if an `Entry` is being created for the first time, then there should be a value supplied for every `index` of that `Entry` . `DiskLruCache` insists that you must pass a value(even if its empty) for every `index` of an `Entry` when that `Entry` is created for the first time. Later any `index` can be edited with a new value.  
-> **hasErrors** determines whether any error occurred while writing to the `OutputStream`  
-> **committed** this flag is `true` if this `Editor` has committed its operation  
-> **newOutputStream(int index)** this returns a new unbuffered `OutputStream` to write the value at `index`. The file name at ith index is of the form key.i where `key` is the `Entry`'s key and i is the index where we want to place the file.   
-> **newInputStream(index)** This will return an unbuffered `InputStream` to read the last committed value at `index`,or null if no value has been committed.



## Journal file
One important feature of DiskLruCache is that it maintains a Journal file for recording various operations that the cache makes. A journal file is nothing but a simple `File` object. There are 4 kinds of entries present in the Journal file which correspond to various cache operations.

-> **READ key**-  It signifies that there has been a `READ` operation for the cache entry with its `key` as mentioned. I don’t understand why the `READ` entry is present. It just tracks the accesses that were done to the LRU. Nothing has been done in the `DiskLruCache` with this information.      
-> **REMOVE key** - This indicates that the cache entry with the specified `key` has been removed from cache(disk cache and LRU cache) and so we remove an entry is it exists in the memory cache while processing the journal file.      
-> **CLEAN key valuesLen** - This entry indicates that an entry with the mentioned `key` has been successfully inserted into the cache and now it may be accessed. `valuesLen` is the length of the length of values at every index for this `Entry` separated by spaces.    
-> **DIRTY key** - A dirty entry indicates that this entry is currently being edited or updated. Every `DIRTY` entry for a particular `key` should be followed by a `REMOVE` or `CLEAN` entry for the same `key`. If there is no matching `REMOVE` or `CLEAN` after `DIRTY`, it means that this entry was left in an inconsistent state during its edit.  

A question that can be thou upon is, why does the `DIRTY` entry exists anyway? Why not just write a `CLEAN` entry to the journal file every time an `Entry` is published? What is the use of this intermediate step?  

This is because suppose an edit or update to a file was in progress and the app got killed somehow.Now what will happen to that half-written file? This is where the `DIRTY` entry comes into picture. Using the `DIRTY` entry, we can at-least delete those files next time when the app starts and the journal is processed again. This entry is used to keep track of such scenarios.Now because we have a `DIRTY` entry, we have a dirty file too corresponding to that. This dirty file gets converted to a clean file when `editor.commit()` is completed.

The journal file is created when the cache is initialized the first time and if a journal file already exists from
the next time of app start, then the entries of the file are simply read into a `LinkedHashMap<Entry.key,Entry>`, which also serves as in the in-memory cache for the `Entry` objects.  
So if the journal file already contains entries from the previous app session, we can create a list of all the cache
entries and put it in the `LinkedHashMap` using the journal file.  
If there is a `REMOVE` entry in the journal file, the corresponding entry is also removed from
the `Map`. For a `CLEAN` entry in the file, a new `Entry` object is created and put it in the `Map`. For a `DIRTY` entry also, a new `Entry` object is put in the `Map`.While putting it in the `Map`, for a `DIRTY` entry, the `Editor` object(`currentEditor`) will be NOT NULL and a new `Editor` is created there. For a `READ` entry, nothing has to be done, because for this `READ` entry, there would already be an `Entry` object created for the corresponding `CLEAN` entry for the same `key`.  
But after reading the journal at the cache start-up time, if there are entries that are completely `DIRTY` and no `CLEAN` or `REMOVE` entry exist for that key, then those entries are inconsistent files/values, they are removed from the `LinkedHashMap`, although they still continue to be in the journal file.

While putting it in the `Map`, for a `DIRTY` entry, the `Editor` object will be NOT NULL and a new `Editor` is
created there. But after reading the journal at the cache start-up time, since `DIRTY` entries are inconsistent
files/values, they are removed from the `LinkedHashMap`, although they still continue to be in the journal file.
For `CLEAN` and `READ` entries, the `Entry` object is just created and inserted.

Also, if we get any `IOException` during reading and processing of journal file, we can be sure that the journal and consequently the cache directory is corrupt, and so we can delete the cache directory. This is how the journal file helps in guarding against the corrupt cache directory.

At the time of starting up the cache initially, since there is no previous journal file, the new journal file is
simply updated with its headers i.e the file name, app version and the `valueCount`.(the max entries a particular key
can store)

The journal file is rebuilt every time the number of operations in the file go beyond 2000. This will cut down unnecessary entries from the file and will keep the size of the journal optimal. Anytime when a journal rebuild is required, firstly the public writer is closed and the journal is updated with a function local `Writer` object. This prevents concurrent updation to the journal file.

So now we have covered all the things required for understanding it, we can go over its major operations.

## Initializing the cache
So before using the cache for storing anything, we need to initialize it.We can initialize or open the cache using
{% highlight java %}
 DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize)
{% endhighlight %}

-> **appVersion** is the version number that will be written in the journal. This is to match this number with the appVersion that is actually present in the journal file, so that we can be sure that we are reading the correct file or the file is not corrupt.  
-> **valueCount** is the number of entries that each key in the cache can have.  
-> **maxSize** is the maximum size that we want the cache to have. If the total size of the files exceed `maxSize`, the entries from the LRU cache are and disk are deleted everytime the journal is rebuilt. But this can pose a problem because suppose we are playing a video from a disk location and it gets deleted due to cache overflow.  

Opening of the cache checks for a journal file. If it already exists, we just create `Entry` objects from it and store in the `Map` as described above in the `journal` section. If it doesn't exist, then a new journal file is created and a `BufferedStream` is opened for writing into that file,which is the journal writer( a `Writer` object).

## Writing into the cache
For writing into that cache, `DiskLruCache` provides the `edit(key)` API and it returns an `Editor`.
{% highlight java %}
DiskLruCache.Editor editor = null;
editor = mDiskCache.edit(key);
{% endhighlight %}


`edit(key)` will return NULL if for the `Entry` with that `key`, the `Editor` is not NULL, means the current `Entry`
is being actively edited. Otherwise,  `Entry` for the given `key` is created or update and a corresponding entry
is made in  the journal file.  
{% highlight java %}
journalWriter.write(DIRTY + ' ' + key + '\n');
{% endhighlight %}

Unless we issue the command `editor.commit()`, the `Entry` will be considered as being edited. This is a quite good
design principle that we can edit an entry/key as long as we want to and the cache will still be consistent. For the
same key, if we are not saving multiple entries concurrently, this will be very safe and sound.

So a journal and a `Map` entry is created, but how do we save the actual data into the disk/cache?
For writing into the cache, we use an `OutputStream`. The `DiskLruCache` opens a stream to a particular location
in the directory. For example, for getting a stream at index 0( remember we can save multiple values for a particular key), and converting it to `BufferedStream` for writing data, we use  
{% highlight java %}
OutputStream out = new BufferedOutputStream(editor.newOutputStream(0), bufferSize);
{% endhighlight %}


`Editor.newOutputStream(index)` will create a dirty file(temporary file with temp extension in the form in the form directory/key/.index.tmp) and returns an `OutputStream` to that location. The dirty file is later changed to a clean file with proper operation once successful edit to this `Entry` has been done.

Now we have to commit this edit so that other readers can use it and another edit can be possible over this key.
So `editor.commit()` command makes this possible. In `commit()`, the dirty file which was created earlier is renamed to a clean file(in the form directory/key/.index) and if any error occurs in this process, the edit is aborted       (`editor.abort()`) and the dirty file is deleted. After the commit is done, a journal entry for `CLEAN` is made and that `Entry` is marked as `readable`(any `Entry` which has been created will be `readable` once its published).    

{% highlight java %}
journalWriter.write(CLEAN + ' ' + entry.key + entry.getLengths() + '\n');
{% endhighlight %}

While writing into the cache, if something goes wrong in the `edit()` or `commit()` process, or any of the steps throw an `IOException`, then we can abort the edit using `editor.abort()`. This will any dirty file created in the process and will also append a `REMOVE` operation for it, since we ought to have a `REMOVE` or `CLEAN` entry in the journal file for a corresponding `DIRTY` entry.

## Getting data from cache  
We get the data from cache in the form of a `Snapshot` object. `DiskLruCache` offers a `get(key)` method for this.
So if we want to get any of the data stored at `key`, we do  

{% highlight java %}
DiskLruCache.Snapshot snapshot = snapshot = mDiskCache.get(key);
{% endhighlight %}

As discussed above, the `Snapshot` object contains an array of `InputStream`s and these are streams to the files at the respective indices. So we will get the stream and convert it into a `BufferedStream` in the following way. Here,
i am getting the `File` at index zero of this `Entry`.
{% highlight java %}
InputStream in = snapshot.getInputStream(0);
BufferedInputStream buffIn = new BufferedInputStream(in, bufferSize);
{% endhighlight %}

Once the data has been read, all the streams can be closed using `snapshot.close()` and a entry is made in the journal file for a successful read.  
{% highlight java %}
journalWriter.append(READ + ' ' + key + '\n');
{% endhighlight %}


## Closing the cache
After everything has been done, the cache can be closed using `cache.close()`. This operation will check that if there are any ongoing edits in progress( `currentEdtior`!=null for an `Entry`), and if there are, it will abort them(`edior.abort()`). Also, the journalWriter is closed, so that nothing more can now be written onto the journal file.  

There are many implementations of `DiskLruCache` available online. One of those implementations is in Google samples itself [here.](https://developer.android.com/samples/DisplayingBitmaps/src/com.example.android.displayingbitmaps/util/ImageCache.html)

[lru_cache]: /images/lru_cache.png "DiskLruCache"
