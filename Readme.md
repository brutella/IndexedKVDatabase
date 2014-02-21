# Summary

This documents shows a very common approach on how to create a fully indexed key-value database. I've summarized it to get a better overview in this topic.

# Key-Value Databases

Databases like [leveldb][leveldb] and [Redis](http://redis.io) store data as key-value pairs and are very simple to use. Therefore they often prefered over other (more complex) databases like [Postgresql]() or [sqlite]().

## Problem

The main problem of key-value databases is that they have no *built-in* support for indexes.

Consider an app which stores data about persons and has the following data model

    Person
    - firstname
    - email
    - age

The data can be stored as a key value pair like

    # <id>=<data>
    '1'='firstname:'Bob',email:bob@mail.com',age:'30'
    '2'='firstname:'Alice',email:alice@mail.com',age:'28'

The `key` is a random number (called `id`) and the `value` contains the data. The data about a person is accessible by reading the value of key `1`.

But there may be use cases where the app wants to get all persons with the age of `28`. A very naive and expensive approach would be to search all key value paris where `age=28`. A better idea is to build an index for `age`.

## Solution

An index for `age` could look like

    # <key>=<value>
    'age/28'='1'
    'age/30'='2'
    
where the key consists of the index name (in this case the property) and property values, and the value are the ids of all persons with the specified age.

All persons with the age of `28` and now accessible via the `age/28` key in the database.

## Strategy

A simple strategy to create indexes is to create keys for every indexed property with the following format

	<property>/<property-value>=<ids>

When a person is deleted from the database, we have to update the index by removing the id from the indexes. A naive and expensive approach would be to iterate over all indexes and look for the deleted id It would be better to store all indexes related to an id by creating an index for id (called **id-index**)like

	ids/<id>=<indexes>

which specifies all indexes for a specific id.

A complete key-value database with the above mentioned data model and all properties index looks like

	'1'='firstname:'Bob',email:bob@mail.com',age:'30'
    '2'='firstname:'Alice',email:alice@mail.com',age:'28'
    
    # Index for firstname
    'firstname/Bob'='1'
    'firstname/Alice'='2'
    
    # Index for age
    'age/28'='2'
    'age/30'='1'
    
    # Index for email
    'email/bob@gmail.com'='1'
    'email/alice@gmailcom'='2'
    
    # Index for ids
	'ids/1'='firstname/Bob','age/30','email/bob@gmail.com'
	'ids/2'='firstname/Alice','age/28','email/alice@gmail.com'
	
	
	
## Database Operations

### Create

1. Create a unique `id`
2. Save data entry as `id=<data>`
3. Create index for every indexed property
	1. Create index key `<property>/<property-value>=<ids>`
	2. Append id if index key already exists, otherwise create new key-value pair
4. Update **id-index**

### Update

1. Delete id from all indexes
2. Update data entry
3. Create index for every indexed property
	1. Create index key `<property>/<property-value>=<ids>`
	2. Append id if index key already exists, otherwise create new key-value pair
4. Update **id-index**

### Delete

1. Delete id from all indexes
2. Delete data entry

### Get by id

That's easy. Read the value from id.

### Query

#### Where Property Equals

E.g. Get all persons with the age of 28.

1. Reads values from index key `age/28`
2. Return value for ids.

#### Where Property In Range

This query requires the keys to be sorted - thank god that [leveldb][leveldb] does this already.

E.g. Get all persons between the age of 27 and 31.

1. Read values for keys between `age/27` and `age/31`
2. Return value for ids.

#### Chained Where Property Equals

E.g. Get all persons with the age of 28 and name Bob.

1. Intersect values from index key `age/28` and `name/Bob`
2. Return value for `id`s.

## Performance Issues

The performance bottle necks are currently related to the expensive index update operations.

### Index Update

In my current implementation the index values are sets of `id`s. I'm using a set because an `id` must not be listed twice in an index (does not make any sense). Since Go does not provide sets by default, I'm using maps, where the key is the `id` and the value is a simple `true` boolean.

The index update operation is expensive because I have to read and parse the stored set, add the id and save it again. It would be better to just append data the value of a key - I'm not sure if this is possible in leveldb.

[leveldb]: https://code.google.com/p/leveldb