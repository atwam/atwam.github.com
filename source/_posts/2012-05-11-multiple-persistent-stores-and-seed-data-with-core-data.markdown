---
layout: post
title: "Multiple persistent stores and seed data with core data"
date: 2012-05-11 23:20
comments: true
categories: core-data ios
---

I haven't posted anything for a while now, and after hours of trying to find a solution to my problem, I thought I should share. So here we go.

### The problem

Your nice and shiny iOS app is supposed to have two data components : **User data** and **Seed data**.
For example, you want to have some (seeded) list of postcodes. The size of data is too big to be shipped with your app,
and we assume that the model is too complex to be just filled by your application at runtime from a downloaded csv/txt file.

So, you start thinking that hey, you'll generate a sqlite database (_persistent data store_ as they say), put it on a server and have your app download and use it.
You can either duplicate the whole stack (`NSManagedObjectContext`, `NSPersistentStoreCoordinator` and `NSManagedObjectModel`) or, according to apple :

_You typically use configurations if you want to store different entities in
 different stores. A persistent store coordinator can only have one managed
 object model, so by default each store associated with a given coordinator
 must contain the same entities. To work around this restriction, you can
 create a model that contains the union of all the entities you want to use.
 You then create configurations in the model for each of the subsets of
 entities that you want to use. You can then use this model when you create a
 coordinator. When you add stores, you specify the different store attributes
 by configuration. When you are creating your configurations, though, remember
 that you cannot create cross-store relationships._

Well, that's pretty much all the doc you'll get from apple.
There are a few mentions of this problem [there](http://stackoverflow.com/questions/9970103/what-is-an-efficient-way-to-merge-two-ios-core-data-persistent-stores),
[or there](http://stackoverflow.com/questions/10224016/coredata-with-multiple-stores-configuration-woes)
but not in a clear enough form for me. So, here's how it works...

<!-- more -->

### Models

We want to have two separate models (because that's the way it is, or because your seed data can be used in apps that have nothing to do with this one).
Let's create two models in xcode (I'll be using dumb _Model_, _Conf_ suffixes just to help understanding) :

* Our _UserModel_ model has one entity `Rating` with two attributes : `postcode` (an `NSString`) and `rating` (an integer).
* Our _PostCodesModel_ model has two entities `Postcode` (with a `postcode` attribute, and let's say some location and address attributes), and `Counties` that
  which postcode belongs to which county.

> We assume that we have already generated a 'PostCodes.sqlite' using normal core data stuff, based on `PostCodesModel` only. That's the seed file we'll want to download.

### How it'll work
Since we can only have one model for one `NSPersistentStoreCoordinator`, we'll need to merge our two models.
We'll create one configuration in each model, with the entities of this model.
We'll then add two `NSPersistentStore`, one per `.sqlite` file, and with a configuration set up to make sure that core data
uses the correct store for the correct entities.

### Adding configurations.
We add one configuration named _UserConf_ to our _UserModel_ model, and drag & drop all our entities to it.
We do the same with a _PostCodesConf_ configuration for our postcodes model.

### Creating a merged model
We change our `-(NSManagedObjectModel*)managedObjectModel` to the following :

``` objc
- (NSManagedObjectModel *)managedObjectModel
{
  if (__managedObjectModel != nil) {
    return __managedObjectModel;
  }
  NSURL *uModelURL = [[NSBundle mainBundle] URLForResource:@"UserModel" withExtension:@"momd"];
  NSManagedObjectModel* uModel = [[NSManagedObjectModel alloc] initWithContentsOfURL:uModelURL];
  
  NSURL *pdModelURL = [[NSBundle mainBundle] URLForResource:@"PostCodesModel" withExtension:@"momd"];
  NSManagedObjectModel* pdModel = [[NSManagedObjectModel alloc] initWithContentsOfURL:pdModelURL];

  __managedObjectModel = [NSManagedObjectModel modelByMergingModels:[NSArray arrayWithObjects:uModel, pdModel, nil]];

  return __managedObjectModel;
}
```

Our new model should now have our three entities, and two configurations : _UserConf_ for our `Rating` entity, and _PostcodesConf_ for the two others.

### Creating a persistent store

That's where the difficulty is. One would think that just calling `addPersistentStoreWithType:configuration:URL:options:error`
once per _.sqlite_ file with the correct configuration would be enough. It isn't. When you add the first one (our postcodes data),
it finds that the store (our downloaded sqlite file) wasn't created with this model : we only used the `PostCodesModel` model to create
our _.sqlite_ file, not our merged model that we are now using.

We could then think of using migration, but then the migrated model used by core data when migrating won't have
our _PostCodesConf_ configuration anymore. *I think that's a bug of core data*.
The solution is to :

1. Add the `postcodes.sqlite` persistent store, *without a configuration, but with the auto-migration options*. Core data will figure out 
   that his sqlite file has just some missing tables (the _UserModel_ tables). This store needs to be writable for the migration to work
   properly.
2. Remove our newly created persistent store (he had a default configuration, we want him to use _UserConf_.
3. Add the same persistent store again, which has now the correct metadata and can be used with our merged model.
4. The user data sqlite file is usually fine, because it's created with the merged model anyway when the app run.
   If not, you could think of using [this plugin architecture](http://www.cimgf.com/2009/05/03/core-data-and-plug-ins/)

Here is what I end up with :

``` objc
- (NSPersistentStoreCoordinator *)persistentStoreCoordinator
{
  if (__persistentStoreCoordinator != nil) {
    return __persistentStoreCoordinator;
  }

  NSError *error = nil;
  __persistentStoreCoordinator = [[NSPersistentStoreCoordinator alloc] initWithManagedObjectModel:[self managedObjectModel]];

  [self addSeedDataToCoordinator:__persistentStoreCoordinator];
  
  NSURL* userURL = [[self applicationDocumentsDirectory] URLByAppendingPathComponent:@"UserData.sqlite"];
  // Note that we use our UserConf here
  if (![__persistentStoreCoordinator addPersistentStoreWithType:NSSQLiteStoreType configuration:@"UserConf" URL:userURL options:nil error:&error])
  {
    NSLog(@"Error %@",error);
  }
}

- (void) addSeedDataToCoordinator:(NSPersistentStoreCoordinator *)storeCoordinator
{
  // Our destination url, writtable. Make sure this is in Library/Cache if you don't want iCloud to backup this.
  NSURL *storeURL = [[self applicationDocumentsDirectory] URLByAppendingPathComponent:@"Postcodes.sqlite"];

  // If we don't have our migrated store, prepare it
  if (![[NSFileManager defaultManager] fileExistsAtPath:[storeURL path]])
  {
    // Our source url should come from a download, but let's use our bundle for debug purposes in the simulator
    NSURL *baseURL = [[NSBundle mainBundle] URLForResource:@"Postcodes" withExtension:@"sqlite"];
    [[NSFileManager defaultManager] copyItemAtURL:baseURL toURL:adURL error:&error];

    // Create one coordinator that just migrates, but isn't used.
    NSDictionary *options = [NSDictionary dictionaryWithObjectsAndKeys:
      [NSNumber numberWithBool:YES], NSMigratePersistentStoresAutomaticallyOption,
      [NSNumber numberWithBool:YES], NSInferMappingModelAutomaticallyOption,
      nil];

    // This will just handle the migration, without any configuration or else ...
    NSPersistentStore* tmpStore = [storeCoordinator addPersistentStoreWithType:NSSQLiteStoreType configuration:nil URL:storeURL options:options error:&error];
    // And remove it !
    [storeCoordinator removePersistentStore:tmpStore error:&error];
  }

  // And now add the coordinator with the correct 'PostCodesConf' configuration, in readonly mode
  NSDictionary *options = [NSDictionary dictionaryWithObjectsAndKeys:[NSNumber numberWithBool:YES], NSReadOnlyPersistentStoreOption, nil];
  [storeCoordinator addPersistentStoreWithType:NSSQLiteStoreType configuration:@"PostCodesConf" URL:storeURL options:options error:&error];
}
```

### Conclusion

Well, adding a bit more information, or a sample somewhere could have been helpful, apple.
