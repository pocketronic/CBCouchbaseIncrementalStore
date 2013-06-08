# CBCouchbaseIncrementalStore #

``NSIncrementalStore`` implementation for _CouchDB_ / _TouchDB_ or _CouchbaseLite iOS_. It uses CouchCocoa or CouchbaseLite API for communicating with the database.

If this document talks about CouchDB, all three are ment, if not otherwise specified.

The main classes are:

  - ``NSCouchCocoaIncrementalStore`` for _TouchDB_ (file URL) and _CouchDB_.
  - ``NSCouchbaseLiteIncrementalStore`` for _CouchbaseLite iOS_

This is a very early version of the store. It is the result of an experiment that worked out rather well. I have been able to replace the SQLite store in one of my main projects by this new store and it worked without any further changes.

I am very much looking forward for your input and help.


# Getting Started #

For _CouchDB_ and _TouchDB_ add the following to your project:

  - ``CouchCocoa.framework`` (https://github.com/couchbaselabs/CouchCocoa)
  - ``TouchDB.framework`` to your project (https://github.com/couchbaselabs/TouchDB-iOS; *only for TouchDB*)
  - Add ``CBCouchbaseAbstractIncrementalStore.h/m`` and ``CBCouchCocoaIncrementalStore.h/m`` to your project
  
For _CouchbaseLite iOS_ add the following to your project:

  - ``CouchbaseLite.framework`` (https://github.com/couchbase/couchbase-lite-ios)
  - Add ``CBCouchbaseAbstractIncrementalStore.h/m`` and ``CBCouchbaseLiteIncrementalStore.h/m`` to your project

Just after you loaded the ``NSManagedObjectModel`` (before using it) update the model (replace class with Couchbase version if applicable):

```
NSURL *modelURL = [[NSBundle bundleForClass:[self class]] URLForResource:@"CouchDBIncrementalStore_iOSDemo" withExtension:@"momd"];
NSManagedObjectModel* model = [[NSManagedObjectModel alloc] initWithContentsOfURL:modelURL];
[CBCouchCocoaIncrementalStore updateManagedObjectModel:model];
```

Now initiate the database as you'd do with other store types but use our new store type:

```
NSPersistentStoreCoordinator* coordinator = [[NSPersistentStoreCoordinator alloc] initWithManagedObjectModel:model];

// TouchDB
NSString *libraryDir = [NSSearchPathForDirectoriesInDomains(NSLibraryDirectory, NSUserDomainMask, YES) lastObject];
NSString *databasePath = [libraryDir stringByAppendingPathComponent:@"test.touchdb"];
NSURL *databaseURL = [[NSURL fileURLWithPath:databasePath] URLByAppendingPathComponent:@"test-eins"];

// CouchDB
// NSURL *databaseURL = [NSURL URLWithString:@"http://127.0.0.1:5984/test-eins"];

NSPersistentStore *persistenStore = [coordinator addPersistentStoreWithType:[CBCouchCocoaIncrementalStore type]
                                                              configuration:nil URL:databaseURL
                                                                    options:nil error:&error];

NSManagedObjectContext* managedObjectContext= [[NSManagedObjectContext alloc] init];
managedObjectContext.persistentStoreCoordinator = coordinator;
```


# Replication #

You can add replication URLs simply by calling this method:

```
[store replicateWithURL:[NSURL URLWithString:@"http://server.host.name:5984/database"] exclusively:NO];
```

The replication works like a charm. I tested replicating some TouchDB clients with one CouchDB server.


# Some Implementation Details #

  - ``NSManagedObjectID``s lastPathComponent is the CouchDB documentID (a UUID generated by the store) with a ``p`` as prefix
  - ``NSDate`` values are stored as ISO strings
  - ``NSData`` values are not supported, yet (should be stored as attachments).
  - For to-many relationships a view is created that groups the destination entities by source entities
  - Two other views are created: all entites by type and all entity-IDs by type (we need to check if we really need them)


# Performance #

Performance improvements have been achieved by:

  - Creating views for to-many relationships to limit the requested data
  - Evaluating ``NSPredicates`` on the CouchDB data instead of always instanciate a ``NSManagedObject``
  - Cache ``NSFetchRequest`` results for certain presets
  
If a fetch request takes more than one second, the store outputs some profiling information that can be used to add some additional views. Example:

```
2013-06-08 22:22:06.414 otest[20559:303] [tdis] fetch request ---------------- 
[tdis]   entity-name:Entry
[tdis]   resultType:NSCountResultType
[tdis]   fetchPredicate: text == "Test2"
[tdis] --> took 8.230835 seconds
[tids]---------------- 
```

You could use this info and add additional views to fetch entities by a property. Example:

```
[store defineFetchViewForEntity:@"Entry" byProperty:@"text"];
```


# Missing / Improvements #

  - Change tracking doesn't yet work in the CouchbaseLite version.
  - Change tracking could be improved: should only happen in ``-[NSManagedObjectContext processPendingChanges]``, etc.
  - Not all expression types are supported when evaluating ``NSExpression``
  - Error handling could be improved (wrapping by own error domain, etc.)
