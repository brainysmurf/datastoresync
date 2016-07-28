#datastoresync

##Explanation

A python framework that makes syncing between two data sets straight-forward. Suppose you have user data in a CSV file and the respective accounts are needed to be created in an external application. On a daily basis, you need to read in that data from both, store them as primitive python values, and then be able to select the differences, and then act on those differences.

This framework solves this problem with the use of a `datastore`, which is defined as a tree-like structure where the root is concerned with storing and retrieving the actual data, and there are at least two nodes on the second level, where one of them is the "source" or "point of truth" data set. Then the third level are mirror images of each other, i.e. 

An graphic example with two trees, and two branches, where we want to sync user account information as well as group information (collections of users):
```
datastore          "trees"             "branches"
---------          -------             ----------
   |
   |                               |--- users
   -----------> source tree -------|--- groups
   |
   |
   -----------> destination tree --|--- users
                                   |--- groups
```

Notice that the branches for each tree have the same number with the same names. This is a requirement, for while you could have asymetrical branches, it is pointless in our syncing scenario. Another requirement is that each item in the branch has to have a unique ID, a string (which we call `idnumber`) that does not get repeated within that branch. The data for the users can be access via the users branch of each respective tree, as if it were a dictionary, i.e. `source.users.get(idnumber)`. Since the datastore aspect is abstracted away, the developer only accesses the data through the trees. For example:

```python
source = SourceTree()
dest = DestinationTree()
```

The class `SourceTree` will need to be defined in such a way so that the branches are available on the tree, which is discussed below in the tutorial. In order to populate the trees with data, we run the importer code on which is avaialble on each branch, with the following syntax:

```python
+source
+dest
```

There is an importer defined for each tree/branch combination, because the code that reads in the user data, say, will be different from reading in the other bits of information that need to be synced over, for example group information. In this particular application, the source will read information in from the CSV file and populate the users branch that way. But the CSV file does not have group information, but can be derived by inspecting the users branch and determining which groups the users belong to, depending, for example, on which department they are in. And this is again different for the destination tree, because it will read in information from the database's User table for the users branch, and then read in info from the Groups branch for the groups branch.

Differences are detected by doing the following manipulations:

```python
source - dest    # generator that outputs 'Action' objects that define the differences, used internally by the framework
source > dest    # outputs to stdout the message property of the raw action objects
source >> dest   # 
```

And we can acess the user data from the source via `source.users` but from the destination area via `dest.users`. This makes for comparing the information ideal to detect differences between the two data sets. We need to understand how to use the framework to do the following things: (1) Define the branches, (2) Define the importers, and (3) Manipulate the data model.

##Tutorial

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
