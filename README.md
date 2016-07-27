#datastoresync

A python framework that makes syncing between two data sets straight-forward. Suppose you have user data in a CSV file and the respective accounts are needed to be created in an external application. You need to read in that data as primitive python values, and then be able to select the differences, and then act on those differences.

This framework solves this problem with the use of a `datastore`, which is defined as a tree-like structure where the root is concerned with storing and retrieving the actual data, and there are at least two nodes on the second level. Then the third level are mirror images of each other, i.e. 

datastore          "trees"             "branches"
---------          -------             ----------
   |
   |                               |--- users
   -----------> source tree -------|--- groups
   |
   -----------> destination tree --|--- users
                                   |--- groups

Notice that the branches for each tree have the same number with the same names. Since the datastore aspect is abstracted away, the developer only accesses the data through the trees. For example:

```
source = SourceTree()
dest = DestinationTree()
```

And we can acess the user data from the source via `source.users` but from the destination area via `dest.users`. This makes for comparing the information ideal to detect differences between the two data sets. We need to understand how to use the framework to do the following things: (1) Define the branches, (2) Import the data, and (3) Manipulate the data model.

This is example code that takes us through each step that is required in the framework.

```
# First we need to define the trees, and then the branches as well. We do that by making a class and indicating in the 
# _branches class variable a dotted notation to the path of the class. 
# The tree will then pick up any subclass of this defined class, and "append" it to the tree as a branch

import dss 

class Source(dss.trees.DataStoreTree):
    _branches = '__main__.SourceBranches'

class SourceBranches(dss.branches.DataStoreBranches):
    """ subclasses of this will be picked up by dss framework """
    pass

class users_branch_mixin:
    """ Just define the name used """
    _name = 'users'

class SourceUsersBranch(SourceBranches, users_branch_mixin):
    _importer = '__main__.SourceImporter'
    _klass = ''

class Dest(dss.trees.DataStoreTree):
    _branches = '__main__.DestBranches'


class DestUser(DestBranches):
    _name = '

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
