#datastoresync

A python framework that makes syncing between two data sets straight-forward. Suppose you have user data in a CSV file and the respective accounts are needed to be created in an external application.

```
class Source(dss.trees.AbstractTree):
    _importer = '__main__.MyCSVImporter'

class Dest(dss.trees.AbstractTree):
    _importer = '__main__.MyPGImporter'

class MyCSVImporter(dss.importers.CSVImporter):
    def get_path(self):
        return "/tmp/file.csv"   # or wherever

    # def readin(self):
    #      this function does not need to be defined because the imported class has code that handles it

class MyPGImporter(dss.importers.PostgressImporter):
    def readin(self):
        """
        Importers that define a readin function need to provide a 
        Code to use sqlalchemy is missing for simplicity's sake
        """
        with self.dbsession() as session:
            yield from session.query(Users).all()

source = Source()
dest = Dest()
```

Both `source` and `dest` have access to the datastore, but this is handled abstractly. The developer only needs to write code that imports the data, which it'll do in two different ways. But in the end, we'll have the data as primitive types stored into the datastore, and then we can extract the differences.

```
source = Source()
dest = Dest()
```

That will initialize the framework, but to actually import the data, we use the following notation:

```
+source
+dest
```

Afterwords, we can access the data, starting with source for example:

```
source.students
-> 
```
