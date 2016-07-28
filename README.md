#datastoresync

##Explanation

A python framework that makes syncing between two applications straight-forward. Suppose you have user data in a CSV file (exported via a cronjob) and the respective accounts are needed to be created in an external application that uses a postgres database. On a daily basis, you need to read in that data from both, store them as primitive python values, and then be able to select the differences, and then act on those differences.

This framework solves this problem with the use of a `datastore`, which is defined as a tree-like structure where the root is concerned with storing and retrieving the actual data, and there are at least two nodes on the second level, where one of them is the "source" or "point of truth" data set. The third level ("branches") represents a category for each kind of object that is in our object model. In our particular instance, we have a source tree that reads in from a CSV file, and a destination tree that reads in from a postgres database. The types of data that are to be synced are users, and groups (collections of groups). 

This is represented visually here, with labels. Note that it represents a datastore that has two sync actions required (change the name of the user whose idnumber is '11111' to "Old Name", and add new user '22222' whose name is "New User"):

```
Level 1            Level 2            Level 3                   Level 4
datastore          "trees"            "branches"                "objects"
(abstract)         source             source.users              source.users.get(idnumber)
---------          --------           --------------------      --------------------------
   |
   |                                  |--- users   -------------| idnumber='11111',name="Old Name"
   |                                  |                         | idnumber='22222',name="New User"
   --------------- source tree -------|--- groups  ---|
                                                      |---------| idnumber='students',members=['11111']   
   |
   |
   --------------- destination tree --|--- users   -------------| idnumber='11111',name="New Name"
                                      |--- groups  ---|
                                                      |---------| idnumber='students',members=['11111']
```

Notice that the branches (at the third level) for each tree have the same number with the same names. This is a convention, for while you could have asymetrical branches, it is pointless in our syncing scenario. There is a requirement, however, that each object in all of the branches has to have a unique ID, a string (which we call `idnumber`). In this way, the data for the users in our CSV file can be accessed via the users branch of each respective tree, as if it were a dictionary, i.e. `source.users.get(idnumber)`. Since the datastore aspect is abstracted away, the developer only accesses the data through the trees.

Let's get started with a bit of pseudocode:

```python
source = SourceTree()
dest = DestinationTree()
```

The class `SourceTree` and `DestinationTree` will need to be defined in such a way so that the branches are available on the tree, which is discussed below in the tutorial. Each branch also has its own importer code. The destination tree will also need to have a template defined, which is triggered when the syncing operation is to commense. (These aspects are discussed in the tutorial below). 

In order to populate the trees with data, we run the importer code which is available on each branch, with the following syntax:

```python
+source
+dest
```

There is an importer defined for each tree/branch combination, because the code that reads in the user data, say, will be different from reading in the other bits of information that need to be synced over, for example group information. In this particular application, the source will read information in from the CSV file and populate the users branch that way. But the CSV file does not have group information, but can be derived by inspecting the users branch and determining which groups the users belong to, depending, for example, on which department they are in. And this is again different for the destination tree, because it will read in information from the database's User table for the users branch, and then read in info from the Groups branch for the groups branch.

Following that, we can inspect the objects that have been created, via the following:

```python
u1 = source.users.get('11111')
u2 = dest.users.get('11111')

u1
User(idnumber='11111', name="Old Name")

u2
User(idnumber='11111', name="New Name")

u1 - u2
update_name(idnumber='11111', attr="name", value="New Name")
```

You can find the differences between particular users, as above, or, with the following, detect the differences among all of the branches, and connect a template object to the action:

```python
source - dest    
# result is a generator of 'Action' objects that define the differences, used internally by the framework

source > dest    
update_name(idnumber='11111', attr="name", value="newname")
# outputs to stdout the message property of the raw action objects, in the above scenario 
# useful for logging or inspection

source >> dest
# called the 'update_name' function of the template.
```

The last part of this framework to understand is what exists at Level 4, the "objects". These are instances of python classes, often with properies defined on them. The framework takes the primitive values that are yielded from the importer, and instantiates a new instance of the class by keyword, where idnumber is the first and only parameter. A vital aspect to understand is that the objects specified by source.users and dest.users have to have the exact same properties.

The way to ensure the objects are symmetical is uing python code to augment itself. For example, one application can export the user names, but does so in a "Last, First Middle" convention, whereas the other application has two fields "firstname" and "lastname".  Since the importer passes the values to the object as found in the source, at the object level, python code can be used augment itself in order to match the other side, by creating a property "name" that operates on `self._lastfirst`. The initial underscore in `_lastfirst` indicates to the framework that it doesn't sync and is to be ignored (and doesn't have to be matched on the other side).

So we need to understand, in depth, how to use the framework to do the following things: (1) Define the branches, (2) Define the code up the importers, and (3) Define the objects.

##Tutorial

This is example code that takes us through each step that is required in the framework.

```
# First we need to define the trees, and then the branches as well. We do that by making a class and indicating in the 
# _branches class variable a dotted notation to the path of the class. 
# The tree will then pick up any subclass of this defined class, and "append" it to the tree as a branch

import dss 

class Source(dss.trees.DataStoreTree):
    """ Our source tree """
    _branches = '__main__.SourceBranches'

class SourceBranches(dss.branches.DataStoreBranches):
    """ subclasses of this will be picked up by dss framework """
    pass

class users_branch_mixin:
    """ Just define the name, i.e. source.users """
    _name = 'users'

class groups_branch_mixin:
    _name = 'groups'

class SourceUsersBranch(SourceBranches, users_branch_mixin):
    """ 
    The source.users branch uses the SourceImporter for the importer
    and the SourceUser class for the objects
    """
    _klass = '__main__.SourceUser'
    _importer = '__main__.SourceImporter'

class SourceUser(dss.model.Base):
    """ 
    Each object in source.users is an instance of this class

    No __init__ method is required, you can assume that the object has been augmented with
    self.properties where properties are the columns provided in the CSV.
    """

    @property
    def name(self):
        """
        The importer populates the objects, but the source data does not have a "name"
        just a "lastfirst", so we define the name property as a derivation of the lastfirst one.
        Note that this lastfirst property starts with an underscore which indicates that the framework
        should ignore it.
        """
        self.lastname, self.firstname = [s.strip(' ') for s in self._lastfirst.split(',')]
        return "{firstname} {lastname}".format(self)

class SourceImporter(dss.importers.CSVImporter):
    def get_path(self):
        return "/tmp/file.csv"   # or wherever

    # def readin(self):
    #      this function does not need to be defined because the imported class has code that handles it

class Dest(dss.trees.DataStoreTree):
    _branches = '__main__.DestBranches'

class DestUser(DestBranches):
    _name = ''


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
