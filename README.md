# Proposal for GSoC 2021

### Scrapy's Feed Enhancements

---

## Personal Details

Name: D R Siddhartha

University: Birla Institute of Technology, Mesra

E-mail: siddharthadr11@gmail.com

Github: [github.com/drs-11](https://github.com/drs-11)

Country: India

---

## Abstract

This project aims to add enhancements to Scrapy's Item Pipeline components. These enhancements consists of item filters, feed compression and batch delivery triggers. 

## Deliverables

...

## Technical Details

1. #### Filter Items

**Solution:** Items can be filtered based on their class type or/and on some certain conditions.  These filters can be either defined in ```settings.py``` or in user's custom ```Item``` class. A criteria argument can be passed to ```Field``` when declaring fields in user's custom Item. There can be a set of buitlin filters such as 'greeater than', 'less than', etc. ```_FeedSlot``` can then can use these filters to tell ```FeedExporter``` to export them or not.

There could be conflicting criterias when declaring both in settings.py as well in custom class items so some precedence order here could be necessary.

**Definitions/Examples:** 

```settings.py``` example:

```python
FEEDS = {
    'items.json': {
        'format': 'json',
        'fields': ['field1', 'field2', 'field3'],
        'fields_filter': {
            'field1': ('gt', 100),    # (comparator, value_to_compare_against)
            'field2': ('lt', 200),    # field2 < 200
            'field3': ('cus', 300),   # custom logic with 300 as threshold
         },
        'item_classes': (Item_1, Item_2),
    },
}
```

Filtering method prototype in ```_FeedSlot```:

```python
def accepts(self, item):
    if self.item_classes and type(item) not in self.item_classes:
        return False

    for field, criteria in self.fields_filter:
         if not item.matches(field, criteria):
            return False

    for field, field_obj in item.fields.items():
        if not item.matches(field, field_obj.criteria):
            return False

    return True
```

Matching method which can be overriden by user in ```Item```:

```python
class Item:
#
# original code
#
    def matches(self, field, criteria):
        if criteria[0] == "cus":
            return self.custom_matcher(field, criteria)

        return self._matcher(field, criteria)

    def _matcher(self, field, criteria)
        #
        # some builtin criteria checks
        #
        return match_status

    def custom_matcher(self, field, criteria):
        raise NotImplementedError

class CustomItem(Item):
    field1 = Field(criteria=('gt', 500))
    field2 = Field()

    def custom_matcher(self, field, criteria):
        #
        # some custom criteria checks
        # 
        return match_status
```

**Control Flow:** 

```
1. slots are created with attributes including fields_filter and item_classes
    imported from settings
2. when an item is scraped
    i. iterate over slots
    ii. pass item to current slot's acceptance method
    iii. the acceptance method will use slot's attributes and item's 
        matches method to determine match_status
    iv. if item is accepted by slot, it is exported by slot
    v. else iteration continues
        
```



2. #### Feed Compression

**Solution:** 

I propose to implement 3 convenient ways to compress feeds. As archiving and compression go hand in hand we can use them both. Option 1 can be archiving all the files into a single file be it from separate feeds or batches. This option will be useful when you just want a single archived file as output. Option 2 can be archiving just a feed when it has batches enabled. Option 3 can be simple compression of output files. No archiving here.

As python has 2 archiving modules (tarfile and zipfile) and 3 compressing modules (gzip, bz2, lzma) in its standard library we can give user the choice to choose from those.

There will be certain precedence and assumpations to get consistent output files. Such as when declaring to archive a feed, it should also have declared ```batch_item_count``` otherwise it wouldn't make sense to archive just a single file. If that happens then only compression shall take place. ```FEED_ARCHIVE_ALL``` option will take higher precedence than to those options declared in ```FEEDS``` dict.

**Definitions/Examples:**

```settings.py``` example:

```python
FEEDS = {
    'item1.json' : {
        'compression': 'gzip',
        'archive': 'tar',
        'batch_item_count': 10,   # output will be item1.tar.gz
    },
    'item2.xml' : {
        'compression': 'bz2',   # output will be item2.xml.bz2
    },
    'item3.jl' : {
        'compression': 'lzma',
        'batch_item_count': 10   # output will be item3-1.jl.xz, item3-2.jl.xz, ...
    },
}


FEED_ARCHIVE_ALL = True   # will be used to archive all of the feeds
FEED_ARCHIVE_ALL_COMPR = 'zipfile'   # default: tarfile
```

**Control Flow:**

```
1) if archive_all conditions are satisfied
    i) open root_archive file and save ptr to FeedExporter
    ii) when a slot closes if root_archive exists then add slot's output file to it
    iii) when spider is closed, close and save the root_archive
2) else if archive_feed conditions are satisfied
    i) open a feed_archive and save ptr to feed's attributes
    ii) when a slot closes, if it's uri template matches feed's then add 
        slot's output file to feed's archive
    iii) when spider is closed, close all feed archives
3) else if compression_only conditions are satisfied
    i) pass compression_type to batch_creator_method
    ii) batch_creator_method will load appropriate compression module and 
        use the compression file like object
```

3. #### Batch Delivery Triggers

**Solution:** 

Batch delivery can be simplified and made extensible by creating a class ```Batch```. It will contain information about current batch so it can be used to detect when a given contraint has been exceeded and new batch needs to be created.

```BatchPerXItems```, ```BatchPerXMins```, and ```BatchPerXBytes``` can be created as builtins with each inhereting parent class ```Batch```. These can be located in  ```scrapy/utils```.  

Desired Batch class can then be activated in ```settings.py``` with a constraint. If no constraint is set, it will be pointless to load the specified Batch class. Users can add their own custom Batch class by specifying their class path.

To stop and create a new batch from the Spider itself a signal can be used. This will require ```Spider``` class to have a method ```self.trigger_batch(feed_uri)``` which will send signal ```signal.stop_batch``` with the feed's URI as argument which can then be intercepted by ```FeedExporter``` and appropriately call a method to stop and start a new batch for the specified feed. A new method in ```FeedExporter``` will be needed to trigger batch delivery as ```item_scraped``` method is used when an item is scraped.

**Definitions/Examples:** 

Batch class template:

```python
class Batch:

    def __init__(self, slot_uri, constraint=None, para_val=None):
        self.uri = slot_uri
        self.constraint = constraint
        self.para_val = para_val

    def update(self, *args):
        # logic to update para_val

    def check(self):
        # logic to check if para_val has crossed the constraint
```

settings.py example:

```python
{
    'items1.json': {
        'format': 'json',
        'batch_constraint': ('item_count', 10)   # (batch_trigger, constraint)
    },
    'items2.xml': {
        'format': 'xml',
        'batch_constraint': ('byte_size', 100)
    },
    'items3.json': {
        'format': 'json',
        'batch_constraint': ('custom', 10)
    },
}


FEED_BATCH_TRIGGER = {
    'custom': 'myproject.customclassfile.CustomBatch'
}


FEED_BATCH_TRIGGER_BASE = {
    'item_count': 'scrapy.utils.feedbatch.BatchByXItems',
    'byte_size': 'scrapy.utils.feedbatch.BatchByXBytes',
    'time_interval': 'scrapy.utils.feedbatch.BatchByXMins',
}
```

**Control Flow:** 

When using a Batch class as a trigger:

```
1. if an item is scraped
    i. iterate through slots
    ii. if slot accepts the item       // assuming this feature is implemented after item filter
        a. export the item
        b. update the parameter value of the feed's batch
        c. if batch's parameter val exceeds constraint
            1. close and start a new batch of the feed
    iii. continue slot iteration till complete
```

When using a signal as a trigger:

```
1. logic code from spider generates signal.stop_batch with the feed_uri as argument
2. FeedExporter catches signal.stop_batch and invokes trigger_batch method
3. trigger_batch method determines which feed's batch to close using feed_uri argument
4. feed_uri's batch is closed and a new batch is created using helper functions
```

## Timeline

...

## Possible Roadblocks

- undesirable side effects of new implementations
- ...

## Technical Knowledge

- ### Programming Experience:
  
  - ...

- ### Personal Projects:
  
  - ...

- ### Open Source Contributions:
  
  - ...
