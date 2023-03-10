---
# Blog post title
title: 'Implementing (simple) MySQL storage engine with libpmemobj'

# Blog post creation date
date: 2015-06-02T19:55:17-07:00

# Change to 'false' when publishing the blog post
draft: false

# Blog post description
description: ''

# Blog post hero image. Used to override the default hero background image.
# eg: image: "/images/my_blog_heroimg.png"
hero_image: ''

# Blog post thumbnail
# eg: image: "/images/my_blog_thumbnail.png"
image: ''

# Blog post author
author: 'pbalcer'

# Categories to which this blog post belongs
blogs: ['mysql']

tags: []

# Redirects from old URL
aliases: ['/2015/06/02/obj-mysql.html']

# Blog post type
type: 'post'
---

The focus of the pmemobj library, like the name suggests, is storing **objects** on a **persistent** medium. A different, but very common, approach of doing exactly the same is to use a database with a specialized interface to manipulate the collection of data. MySQL is one such database, it processes SQL queries by calling (usually) multiple methods of the storage engine used for the data tables that query operates on. This tutorial covers basics of the pmemobj library, including non-transactional allocations and very simple transactions.

For simplicity sake the engine described here is very rudimentary. It does not support multiple simultaneous connections and there's no implementation of indexes nor transactions.

### Design

To have a working storage engine, implementation of a few methods is required. First of all, a table has to be created, opened and then closed - for our engine a table will be represented by a single pmemobj pool in a file with **.obj** extension. Once that's done we have to decide on the record representation in the internal database engine, the easiest way is simply to use the mysql format and not to parse the record data at all - just note that for more advanced engines that may be not the best choice. Only one major design decision left - there has to be a way of iterating through the records, thankfully the pmemobj library internally stores the allocated objects for us and has an interface to access them one at a time, so our engine doesn't have to do anything special, but of course if you want to implement indexing it won't be sufficient.

Since the engine will handle the record data directly, the implementation of INSERT will be simple, allocate an object to the internal collection and copy the data. How to handle SELECTs? That should be obvious by now. MySQL requires implementation of a simple iterator with 2 operations: init and next which can be almost directly mapped to `POBJ_FIRST` and `POBJ_NEXT` pmemobj API functions. The only two operations left that the engine will support are UPDATE and DELETE. They both operate on the most recently read object by the iterator, the UPDATE is a simple memcpy and DELETE is a free. Yes, it's really that simple :)

### Implementation

This tutorial covers only the essential information about MySQL internals required to implement storage engine, for more information visit [this excellent tutorial](https://dev.mysql.com/doc/internals/en/custom-engine.html). Our starting point will be the example storage engine available on the [MySQL github page](https://github.com/mysql/mysql-server). You can modify the example itself or copy it with a different name to the same directory, CMake will automatically build it with the entire project.

#### Defining pool extension

To make use of MySQL automagically cleaning up the tables for us on DROP TABLE the extension of the pool file has to be specified, to do that we have to define the extension:

```c++
#define PMEMOBJ_EXT ".obj"
```

Declare a table with extensions terminated by `NullS` variable:

```c++
static const char *ha_obj_exts[] = {
    PMEMOBJ_EXT,
    NullS
};
```

and then implement the `bas_ext` method of the handler class:

```c++
const char **ha_obj::bas_ext() const
{
    return ha_obj_exts;
}
```

From now on, we don't have to worry about leftover pool files created by the engine, all files with the `.obj` extension in the database directory will get erased when no longer needed.

#### Creating the table

As mentioned previously, we will create a new pmemobj pool for each table. We have to remember to add the file extension to the table name and then just create the pool. A good candidate for the `layout` parameter of the `pmemobj_create` method is the `name`, because it uniquely identifies that table in the database. Since the pmemobj pool cannot grow we have to choose some arbitrary value here represented by the `DEFAULT_TAB_SIZE`. Keep in mind that `create` function does not mean `open`, so there's no need to hold the `PMEMobjpool` pointer anywhere.

```c++
int ha_obj::create(const char *name, TABLE *table_arg, HA_CREATE_INFO *create_info)
{
    DBUG_ENTER("ha_obj::create");

    char path[MAX_PATH_LEN];
    snprintf(path, MAX_PATH_LEN, "%s%s", name, PMEMOBJ_EXT);

    PMEMobjpool *pop = pmemobj_create(path, name, DEFAULT_TAB_SIZE, S_IRWXU);
    if (pop == NULL)
    	DBUG_RETURN(HA_ERR_OUT_OF_MEM);

    pmemobj_close(pop);

    DBUG_RETURN(0);
}
```

Just to keep things simple, the `MAX_PATH_LEN` evaluates to `255` in the example code, actual implementation might want to do this properly.

#### Opening the table

Now that we have the pool created, it would be a good idea to open it and this time actually keep the pointer. To do that, add a new variable to the table class that will hold the `PMEMobjpool`:

```c++
class ha_obj: public handler
{
    ...

private:
    PMEMobjpool *objtab;
};
```

The function this variable should be initialized in is `open`, and it looks like this:

```c++
int ha_obj::open(const char *name, int mode, uint test_if_locked)
{
    DBUG_ENTER("ha_obj::open");

    char path[MAX_PATH_LEN];
    snprintf(path, MAX_PATH_LEN, "%s%s", name, PMEMOBJ_EXT);

    objtab = pmemobj_open(path, name);
    if (objtab == NULL)
    	DBUG_RETURN(HA_ERR_CRASHED_ON_USAGE);

    DBUG_RETURN(0);
}
```

No surprises here: add the extension and open the pool - done.

#### Closing the table

For sake of completeness, here's the `close` method:

```c++
int ha_obj::close(void)
{
    DBUG_ENTER("ha_obj::close");

    pmemobj_close(objtab);
    objtab = NULL;

    DBUG_RETURN(0);
}
```

#### Implementing INSERT statement

Now that we can properly manage tables it's time to populate them. Here's the method that has to be implemented:

```c++
virtual int write_row(uchar *buf);
```

The return value is 0 when the row was properly written and `HA_ERR_OUT_OF_MEM` when a new row cannot be allocated. The byte buffer `buf` contains the record data in MySQL internal format - this engine will not interpret it in any way. You might have noticed that there's no length accompanying this buffer, this is because its maximum size is equal to the record length - which the user has to specify in `CREATE TABLE`, that size is stored in `table->s->reclength` variable.
Now let's define the row structure we are going to allocate. Since at compile time the `reclength` is not known, this has to suffice:

```c++
POBJ_LAYOUT_BEGIN(mysql_obj);
POBJ_LAYOUT_TOID(mysql_obj, struct table_row);
POBJ_LAYOUT_END(mysql_obj);

struct table_row {
    uchar buf[];
};
```

Now that we have everything we need, just allocate an object from the table pool and do a memcpy:

```c++
TOID(struct table_row) row;
row = POBJ_ZALLOC(objtab, &row, struct table_row, table->s->reclength);
if (TOID_IS_NULL(row))
    DBUG_RETURN(HA_ERR_OUT_OF_MEM);
memcpy(D_RW(row)->buf, buf, table->s->reclength);
```

Looks good, right? Maybe at the first glance, but there are in fact two things wrong here. First of all, the data is never 'persisted' to the memory which means that there's no way of knowing when the record data will be in fact stored on the desired storage medium and not in cache. Luckily there's an easy fix for that in the library - the `pmemobj_memcpy_persist` function, that, where possible, uses non-temporal stores to bypass the cache and write the data directly to the memory. So what's the other wrong thing? Well, imagine someone pulled the plug from the server right in the middle of the memcpy operation - the object will be there in the table but with only half the data and, what's worse, there is no way of detecting if this row is invalid. There are two different tools our library provides to combat this fundamental problem: transactions and allocation constructors. Using the first approach, correct implementation of this method might look like this:

```c++
TX_BEGIN(objtab) {
    TOID(struct table_row) row = TX_ALLOC(struct table_row, table->s->reclength);
    /* don't do pmemobj_tx_add_range, the object is new */
    memcpy(D_RW(row)->buf, buf, table->s->reclength);
} TX_ONABORT {
    err = HA_ERR_OUT_OF_MEM;
} TX_END
```

Which is perfectly OK, the allocation will be placed in the `TABLE_ROW_TYPE` collection only if the entire transaction successfully finishes. Also notice that there's no NULL-check like before, this is because the transaction will get aborted if any of the `pmemobj_tx_*` methods inside it fails. Even though we invested a lot of effort in making sure the transactions don't add too much overhead, far more is done behind the scenes here then with a simple allocation and that's where the constructor comes in to play, so let's write slightly more efficient, and final, version of the entire method.

First - the constructor, its only task is to copy persistently the MySQL-provided buffer to the allocated object. The constructor callback has two arguments: raw pointer to allocated object and context argument. There's no need to use `pmemobj_direct` or `D_*` on the ptr because it's already a direct pointer, and don't forget to persist the modified data. It might be tempting, for other uses, to allocate another pmem object within the constructor but that is not permitted and leads to undefined behavior. The constructor is invoked after the memory has been reserved in a volatile state, but before any persistent changes to the heap layout has been made, so when an interruption happens there's nothing to rollback - there simply won't be a trace of this allocation. Also it's important to note that no locks are held during the constructor call, hence the user is responsible for providing any synchronization.

```c++
struct row_args {
    uchar *buf;
    size_t len;
};

void row_construct(PMEMobjpool *pop, void *ptr, void *args)
{
    struct row_args *r = (struct row_args *)args;
    struct table_row *row = (struct table_row *)ptr;
    pmemobj_memcpy_persist(pop, row->buf, r->buf, r->len);
}
```

The rest is very simple, allocate the object and check if the operation succeeded.

```c++
int ha_obj::write_row(uchar *buf)
{
    DBUG_ENTER("ha_obj::write_row");

    if (table->timestamp_field_type & TIMESTAMP_AUTO_SET_ON_INSERT)
    	table->timestamp_field->set_time();

    struct row_args args = {buf, table->s->reclength};
    TOID(struct table_row) row;
    POBJ_ALLOC(objtab, &row, struct table_row, table->s->reclength, row_construct, &args);

    if (row.oid.off == 0)
    	DBUG_RETURN(HA_ERR_OUT_OF_MEM);

    stats.records++;

    DBUG_RETURN(0);
}
```

Just to showcase different methods of achieving the same thing, raw `PMEMoid` is used here and NULL-checked.

#### Implementing SELECT statement

The engine wouldn't be very useful if the only supported operation was INSERT. Let's implement the methods to read row data, but keep in mind that locks and generally synchronization of the storage engine isn't topic of this tutorial, for that please refer to the MySQL documentation.

```c++
virtual int rnd_init(bool scan) = 0;
virtual int rnd_end();
virtual int rnd_next(uchar *buf) = 0;
virtual int rnd_pos(uchar *buf, uchar *pos) = 0;
virtual void position(const uchar *record) = 0;
```

As you can see there is quite a lot of functions to implement, but no worries, it's quite easy. First we have to add two persistent memory pointers to the table class that will serve us as iterators. The reason for the two variables will become clear later.

```c++
class ha_obj : public handler
{
    ...
    private:
    ...
    TOID(struct table_row) iter;
    TOID(struct table_row) current;
};
```

The `rnd_init` is called before table scans, here we initialize the iterator, pointing it to the first object in the collection. Similarly the `rnd_end` is called after table scans, it is where the cleanup happens.

```c++
int ha_obj::rnd_init(bool scan)
{
    DBUG_ENTER("ha_obj::rnd_init");

    iter = POBJ_FIRST(objtab, struct table_row);

    DBUG_RETURN(0);
}

int ha_obj::rnd_end()
{
    DBUG_ENTER("ha_obj::rnd_end");

    TOID_ASSIGN(iter, OID_NULL);
    TOID_ASSIGN(current, OID_NULL);

    DBUG_RETURN(0);
}
```

Hope that's clear and easy to understand.

Moving on to the `rnd_next` which is required for sequential reads, basically the whole job is to populate a buffer with data from current object and move the iterator forward. If there are no more objects to read the return value has to be `HA_ERR_END_OF_FILE`. Here's the complete implementation:

```c++
int ha_obj::rnd_next(uchar *buf)
{
    DBUG_ENTER("ha_obj::rnd_next");
    MYSQL_READ_ROW_START(table_share->db.str, table_share->table_name.str, TRUE);

    if (OID_IS_NULL(iter)) {
    	MYSQL_READ_ROW_DONE(HA_ERR_END_OF_FILE);
    	DBUG_RETURN(HA_ERR_END_OF_FILE);
    }

    memcpy(buf, D_RO(iter)->buf, table->s->reclength);

    current = iter;
    iter = POBJ_NEXT(iter);

    MYSQL_READ_ROW_DONE(0);
    DBUG_RETURN(0);
}
```

One might wonder why this time a regular memcpy is used, look at the destination of the copying - it is into volatile memory.

The remaining two methods are for non-sequential access. The `position` is called to save the most recently read row pointer into the record buffer, there's a helper macro to do just that. The `off` variable of `PMEMoid` stores the offset of the pointer relative to the pool, that's why this simple trick works:

```c++
void ha_obj::position(const uchar *record)
{
    DBUG_ENTER("ha_obj::position");

    my_store_ptr(ref, ref_length, current.oid.off);
}
```

The opposite operation `rnd_pos` has to read the buffer based on the previously stored value. This method is similar to the `rnd_next`, but notice the way `PMEMoid` is recreated basing on the stored offset.

```c++
int ha_obj::rnd_pos(uchar *buf, uchar *pos)
{
    DBUG_ENTER("ha_obj::rnd_pos");

    MYSQL_READ_ROW_START(table_share->db.str, table_share->table_name.str,
    		FALSE);

    PMEMoid oid = {iter.oid.pool_uuid_lo, my_get_ptr(pos, ref_length)};
    if (oid.off == 0) {
    	MYSQL_READ_ROW_DONE(HA_ERR_END_OF_FILE);
    	DBUG_RETURN(HA_ERR_END_OF_FILE);
    }

    TOID_ASSIGN(iter, oid);
    TOID_ASSIGN(current, oid);

    memcpy(buf, D_RO(iter)->buf, table->s->reclength);

    MYSQL_READ_ROW_DONE(0);
    DBUG_RETURN(0);
}
```

Now we have a functional storage engine that actually works ;) You can INSERT and SELECT rows, and even ORDER BY works.

#### Implementing DELETE statement

To do this operation the MySQL server first reads the row to be deleted and then calls the `delete_row` method. So we just have to free the `current` variable.

```c++
int ha_obj::delete_row(const uchar *buf)
{
    DBUG_ENTER("ha_obj::delete_row");

    POBJ_FREE(&current);
    stats.records--;

    DBUG_RETURN(0);
}
```

#### Implementing UPDATE statement

This is the same deal as with DELETE, but instead of free we have to do a memcpy. You might be tempted to write something like this:

```c++
memcpy(D_RW(current)->buf, new_data, table->s->reclength);
```

That would be obviously wrong, remember INSERT? Here we have to use a transaction:

```c++
int ha_obj::update_row(const uchar *old_data, uchar *new_data)
{
    DBUG_ENTER("ha_obj::update_row");

    if (table->timestamp_field_type & TIMESTAMP_AUTO_SET_ON_UPDATE)
    	table->timestamp_field->set_time();

    TX_BEGIN(objtab) {
    	TX_MEMCPY(D_RW(current)->buf, new_data, table->s->reclength);
    	// don't persist, the TX will know to flush this memory range
    } TX_END

    DBUG_RETURN(0);
}
```

This will store the old value in the pool and, in case of an interruption, will do a rollback or, when everything succeeds, discards the old data backup.

### Conclusion

Just to verify if the implementation works, run the following command:

```bash
$ ./mysqlslap -u[user] -p[pass] --auto-generate-sql --auto-generate-sql-execute-number=5000 --engine=obj --verbose --iterations=1 --number-int-cols=5 --number-char-cols=20
```

If it doesn't segfault then congratulations, you've just implemented a MySQL storage engine using only libpmemobj. I personally hope the experience wasn't too bad.

Here's an example output of the above command:

```
Benchmark
    Running for engine obj
    Average number of seconds to run all queries: 6.334 seconds
    Minimum number of seconds to run all queries: 6.334 seconds
    Maximum number of seconds to run all queries: 6.334 seconds
    Number of clients running queries: 1
    Average number of queries per client: 5000
```

Keep in mind that while this isn't made up, the numbers don't matter ;)
