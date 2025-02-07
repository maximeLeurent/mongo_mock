|travis| |pypi_version| |pypi_license| |pypi_wheel|

.. image:: https://codecov.io/gh/mongomock/mongomock/branch/develop/graph/badge.svg
  :target: https://codecov.io/gh/mongomock/mongomock


What is this?
-------------
Mongomock is a small library to help testing Python code that interacts with MongoDB via Pymongo.

To understand what it's useful for, we can take the following code:

.. code-block:: python

 def increase_votes(collection):
     for document in collection.find():
         collection.update_one(document, {'$set': {'votes': document['votes'] + 1}})

The above code can be tested in several ways:

1. It can be tested against a real mongodb instance with pymongo.
2. It can receive a record-replay style mock as an argument. In this manner we record the
   expected calls (find, and then a series of updates), and replay them later.
3. It can receive a carefully hand-crafted mock responding to find() and update() appropriately.

Option number 1 is obviously the best approach here, since we are testing against a real mongodb
instance. However, a mongodb instance needs to be set up for this, and cleaned before/after the
test. You might want to run your tests in continuous integration servers, on your laptop, or
other bizarre platforms - which makes the mongodb requirement a liability.

We are left with #2 and #3. Unfortunately they are very high maintenance in real scenarios,
since they replicate the series of calls made in the code, violating the DRY rule. Let's see
#2 in action - we might write our test like so:

.. code-block:: python

 def test_increase_votes():
     objects = [dict(...), dict(...), ...]
     collection_mock = my_favorite_mock_library.create_mock(Collection)
     record()
     collection_mock.find().AndReturn(objects)
     for obj in objects:
         collection_mock.update_one(document, {'$set': {'votes': document['votes']}})
     replay()
     increase_votes(collection_mock)
     verify()

Let's assume the code changes one day, because the author just learned about the '$inc' instruction:

.. code-block:: python

 def increase_votes(collection):
     collection.update_many({}, {'$inc': {'votes': 1}})

This breaks the test, although the end result being tested is just the same. The test also repeats
large portions of the code we already wrote.

We are left, therefore, with option #3 -- you want something to behave like a mongodb database
collection, without being one. This is exactly what this library aims to provide. With mongomock,
the test simply becomes:

.. code-block:: python

 def test_increase_votes():
     collection = mongomock.MongoClient().db.collection
     objects = [dict(votes=1), dict(votes=2), ...]
     for obj in objects:
         obj['_id'] = collection.insert_one(obj).inserted_id
     increase_votes(collection)
     for obj in objects:
         stored_obj = collection.find_one({'_id': obj['_id']})
         stored_obj['votes'] -= 1
         assert stored_obj == obj # by comparing all fields we make sure only votes changed

This code checks *increase_votes* with respect to its functionality, not syntax or algorithm, and
therefore is much more robust as a test.

If the code to be tested is creating the connection itself with pymongo, you can use
mongomock.patch:

.. code-block:: python

  @mongomock.patch(servers=(('server.example.com', 27017),))
  def test_increate_votes_endpoint():
    objects = [dict(votes=1), dict(votes=2), ...]
    client = pymongo.MongoClient('server.example.com')
    client.db.collection.insert_many(objects)
    call_endpoint('/votes')
    ... verify client.db.collection


Important Note About Project Status & Development
-------------------------------------------------

MongoDB is complex. This library aims at a reasonably complete mock of MongoDB for testing purposes,
not a perfect replica. This means some features are not likely to make it in any time soon.

Also, since many corner cases are encountered along the way, our goal is to try and TDD our way into
completeness. This means that every time we encounter a missing or broken (incompatible) feature, we
write a test for it and fix it. There are probably lots of such issues hiding around lurking, so feel
free to open issues and/or pull requests and help the project out!

**NOTE**: We don't include pymongo functionality as "stubs" or "placeholders". Since this library is
used to validate production code, it is unacceptable to behave differently than the real pymongo
implementation. In such cases it is better to throw `NotImplementedError` than implement a modified
version of the original behavior.

Contributing
------------

When submitting a PR, please make sure that:

1. You include tests for the feature you are adding or bug you are fixing. Preferably, the test should
   compare against the real MongoDB engine (see `examples in tests`_ for reference).
2. No existing test got deleted or unintentionally castrated
3. The travis build passes on your PR.

To download, setup and perfom tests, run the following commands on Mac / Linux:

.. code-block:: bash

 git clone git@github.com:vmalloc/mongomock.git
 pip install tox
 cd mongomock
 tox


Branching model
~~~~~~~~~~~~~~~

The branching model used for this project follows the `gitflow workflow`_.  This means that pull requests
should be issued against the `develop` branch and *not* the `master` branch. If you want to contribute to
the legacy 2.x branch than your pull request should go into the `support/2.x` branch.

Acknowledgements
----------------

Mongomock has originally been developed by `Rotem Yaari <https://github.com/vmalloc/>`_, then by `Martin Domke <https://github.com/mdomke`. It is currently being developed and maintained by `Pascal Corpet <https://github.com/pcorpet>`_ .

Also, many thanks go to the following people for helping out, contributing pull requests and fixing bugs:

* Alec Perkins
* Alexandre Viau
* Austin W Ellis
* Andrey Ovchinnikov
* Arthur Hirata
* Baruch Oxman
* Corey Downing
* Craig Hobbs
* Daniel Murray
* David Fischer
* Diego Garcia
* Dmitriy Kostochko
* Drew Winstel
* Edward D'Souza
* Emily Rosengren
* Eugene Chernyshov
* Grigoriy Osadchenko
* Israel Teixeira
* Jacob Perkins
* Jason Burchfield
* Jason Sommer
* Jeff Browning
* Jeff McGee
* Joël Franusic
* `Jonathan Hedén <https://github.com/jheden/>`_
* Julian Hille
* Krzysztof Płocharz
* Lyon Zhang
* Marc Prewitt
* Marcin Barczynski
* Marian Galik
* Michał Albrycht
* Mike Ho
* Nigel Choi
* Omer Gertel
* Omer Katz
* Papp Győző
* Paul Glass
* Scott Sexton
* Srinivas Reddy Thatiparthy
* Taras Boiko
* Todd Tomkinson
* Zachary Carter
* catty (ca77y _at_ live.com)
* emosenkis
* hthieu1110
* יppetlinskiy
* pacud
* tipok
* waskew (waskew _at_ narrativescience.com)
* jmsantorum (jmsantorum [at] gmail [dot] com)
* lidongyong


.. _examples in tests: https://github.com/vmalloc/mongomock/blob/master/tests/test__mongomock.py#L108

.. _gitflow workflow: https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow


.. |travis| image:: https://travis-ci.org/mongomock/mongomock.svg?branch=develop
    :target: https://travis-ci.org/mongomock/mongomock
    :alt: Travis CI build

.. |pypi_version| image:: https://img.shields.io/pypi/v/mongomock.svg
    :target: https://pypi.python.org/pypi/mongomock
    :alt: PyPI package

.. |pypi_license| image:: https://img.shields.io/pypi/l/mongomock.svg
    :alt: PyPI license

.. |pypi_wheel| image:: https://img.shields.io/pypi/wheel/mongomock.svg
    :alt: PyPI wheel status
