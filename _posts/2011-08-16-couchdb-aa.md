---
layout: post
title: CouchDB Authentication & Authorization
tags: [couchdb,authentication,authorization,nosql]
date: '2011-08-16'
author: Evan Liu
---
Recently I looked a lot into couchdb and found it has great potential to extremely power the mobile application development, because it is:

- schema free, so no need to worry about schema design & change
- rest ready, so no need to code a middleware in PHP, python or nodejs
- scale-out easy, so no need to worry refactoring too much later on
- other cool stuffs, such as master-master replication, crash-only design, MVCC & etag, \_changes API, etc

However, it needs some time to master. Documents and manuals scatter everywhere. A cookbook is lacked.

Here I tried to describe the authentication and authorization system of couchdb to the fans on the couch :)

## Recipe 1. close the door to admin party
Once you successfully installed couchdb by, for example, the following command in ubuntu:

    $ sudo apt-get install couchdb

You can notice that the couchdb is listening the port 5984 of 127.0.0.1 (so only localhost can access it right now):

    $ netstat -na | grep 5984
    tcp        0      0 127.0.0.1:5984          0.0.0.0:*               LISTEN

Then try to:

    $ curl localhost:5984
    {"couchdb":"Welcome","version":"1.0.1"}

Voila, it works!

However everybody can fully access the database because there is no server admin and all are in the so-called _admin party_. let's have a try to create a db:

    $ curl -XPUT localhost:5984/testdb1
    {"ok":true}

It is easy to turn off the admin party, just create a server admin:

    $ curl -XPUT localhost:5984/_config/admins/root -d '"secret"'
    ""

As shown above, the admin username is _root_ and its password is _secret_. and let's try to create another db:

    $ curl -XPUT localhost:5984/testdb2
    {"error":"unauthorized","reason":"You are not a server admin."}

The door to admin party has been closed now.

We can show our passport to complete our task:

    $ curl -XPUT root:secret@localhost:5984/testdb2
    {"ok":true}

## Recipe 2. user sign-up
The best practice to manage your app's user base is to use couchdb's \_users database with which you can enjoy couchdb's AA system later on.

To sign up a user, we need to PUT it to /\_users. you may directly get this:

    $ curl -XPUT root:secret@localhost:5984/_users/SOMETHING

And you quickly start to worry that if you write the root:secret in your mobile app then it may be stolen.

Stop worrying. in fact, the \_users database is *public* which means everyone can CRUD (Create Read Update Delete) its documents.

Scared? no matter. we have validate\_doc\_update in /\_users/\_design/\_auth to protect the CUD operations:

- anonymous can only Read everyone's profile document
- anyone can only CUD his/her own profile document
- only server admin can CUD everyone's profile document as well as the \_design documents

Therefore, your app can just PUT to create a new user without revealing the root:secret like:

    $ curl -XPUT localhost:5984/_users/org.couchdb.user:test1 -d @-
    {
            "type":"user",
            "name":"test1",
            "roles":["testdb1_user"],
            "password_sha":"7bf406fd422fc7cd6041b3370f6f34a5f233be83",
            "salt":"4eb026e94d31db46928297c9a49c3e77"
    }
    ^D
    {"error":"forbidden","reason":"Only _admin may set roles"}

Wait... what happened?

    $ curl -XPUT localhost:5984/_users/org.couchdb.user:test1 -d @-
    {
            "type":"user",
            "name":"test1",
            "roles":[],
            "password_sha":"7bf406fd422fc7cd6041b3370f6f34a5f233be83",
            "salt":"4eb026e94d31db46928297c9a49c3e77"
    }
    ^D
    {"ok":true,"id":"org.couchdb.user:test1","rev":"1-2dbb49257496d609220bce8c226d8746"}

Aha, anonymous cannot set roles of a user. Neither does the user himself/herself:

    $ curl -XPUT test1:secret@localhost:5984/_users/org.couchdb.user:test1 -d @-
    {
            "_rev":"1-2dbb49257496d609220bce8c226d8746",
            "type":"user",
            "name":"test1",
            "roles":["testdb1_user"],
            "password_sha":"7bf406fd422fc7cd6041b3370f6f34a5f233be83",
            "salt":"4eb026e94d31db46928297c9a49c3e77"
    }
    ^D
    {"error":"forbidden","reason":"Only _admin may edit roles"} 

Safe, uh? Otherwise the user can change his/her role to get unexpected access permissions.

## Recipe 3. user authorization

Users are authorized to do stuffs against the db by ACL (Access Control List, for reading) and validation functions (for writing).

For reading, there is a special JSON stored in each database called _security. Let's take a look:

    $ curl localhost:5984/testdb1/_security
    {}

Nothing there. Since no ACLs there, the db is defaultly public (which is good):

    $ curl localhost:5984/testdb1
    {"db_name":"testdb1","doc_count":0,"doc_del_count":0,"update_seq":0,
    "purge_seq":0,"compact_running":false,"disk_size":79,
    "instance_start_time":"1313509476519859","disk_format_version":5,
    "committed_update_seq":0}

Now let's add some ACLs:

    $ curl -XPUT localhost:5984/testdb1/_security -d @-
    {
        "admins": {
            "roles": [],
            "names": []
        },
        "readers": {
            "roles": [],
            "names": ["test1"]
        },
    }
    ^D
    {"error":"unauthorized","reason":"You are not a db or server admin."}

Emmm. Let's try again with the server admin credential:

    $ curl -XPUT root:secret@localhost:5984/testdb1/_security -d @-
    {
        "admins": {
            "roles": [],
            "names": []
        },
        "readers": {
            "roles": [],
            "names": ["test1"]
        },
    }
    ^D
    {"ok":true}

Well done. Let's check it out:

    $ curl localhost:5984/testdb1/_security
    {"error":"unauthorized","reason":"You are not authorized to access this db."}

Aha~ because we set a reader for the db so from now on only test1 (and the admins) can read the db.

    $ curl test1:secret@localhost:5984/testdb1/_security
    {"admins":{"roles":[],"names":[]},"readers":{"roles":[],"names":["test1"]}}

Got it!

However, the problem is if we want our mobile app to CRUD testdb1 in the name of the logged-in user, it's not a good idea to list all the usernames in the _names_ list of the _readers_.

The obvious better choice is to use _roles_, i.e., set the role of the users to  _testdb1\_user_ and add _testdb1\_user_ into the _roles_ list of the _readers_.

Bad news is that we cannot change the role of the users to _testdb1\_user_ without showing the admin credentials.

Eventually here comes two choices:

1. write either a middleware or daemon listening _change to always add the role to every user
2. keep the db public

The latter is a simpler and better choice, because anonymous can sign up and get the role always where is no difference with the latter choice at all, surprisingly.

Let's clear the _security first:

    $ curl -XPUT root:secret@localhost:5984/testdb1/_security -d '{}'
    {"ok":true}

Notice that _security is a special doc without MVCC.

Imagine a case of a BBS:

- anyone can sign up, write articles and publish them
- one can read all articles
- one can only update/delete his/her own article
- BMs (Board Managers) can update/delete anyone's articles
- Site operators can assign BMs
- server admin can assign site operators

How to authorize the db writing correctly? The answer is to use the validation function which is named _validate\_doc\_update_ in any \_design docs. For example:

    $ curl -XPUT test1:secret@localhost:5984/testdb1/_design/auth -d @-
    {
        "validate_doc_update":"function(newDoc, oldDoc, userCtx) { \ 
        if (oldDoc && oldDoc.author) { if (oldDoc.author != userCtx.name) \ 
        throw({forbidden:'You may only update documents with author ' \ 
        + oldDoc.author});} }"
    }
    ^D
    {"error":"unauthorized","reason":"You are not a db or server admin."}

As above, db readers cannot write _design docs. We need to provide admin credential to do it:

    $ curl -XPUT root:secret@localhost:5984/testdb1/_design/auth -d @-
    {   
        "validate_doc_update":"function(newDoc, oldDoc, userCtx) { \ 
        if (oldDoc && oldDoc.author) { if (oldDoc.author != userCtx.name) \ 
        throw({forbidden:'You may only update documents with author ' \ 
        + oldDoc.author});} }"
    }
    ^D
    {"ok":true,"id":"_design/auth","rev":"1-7473924693b4c1cf0f5af9100b867951"}

Let's create a new article:

    $ curl -XPUT test1:secret@localhost:5984/testdb1/article1 -d @-
    {"author":"test2"}
    ^D
    {"ok":true,"id":"article1","rev":"1-0cb3999630b9b1ed3846029f66d4d62b"}

Check if test1 can update it:
    
    $ curl -XPUT test1:secret@localhost:5984/testdb1/article1 -d @-
    {"_rev":"1-0cb3999630b9b1ed3846029f66d4d62b", "author":"test1"}
    ^D
    {"error":"forbidden","reason":"You may only update documents with author test2"}

Even admin cannot update:

    $ curl -XPUT root:secret@localhost:5984/testdb1/article1 -d @-
    {"_rev":"1-0cb3999630b9b1ed3846029f66d4d62b", "author":"test1"}
    ^D
    {"error":"forbidden","reason":"You may only update documents with author test2"}

Let's create a new user test2:

    $ curl -XPUT localhost:5984/_users/org.couchdb.user:test2 -d @-
    {
            "type":"user",
            "name":"test2",
            "roles":[],
            "password_sha":"7bf406fd422fc7cd6041b3370f6f34a5f233be83",
            "salt":"4eb026e94d31db46928297c9a49c3e77"
    }
    ^D
    {"ok":true,"id":"org.couchdb.user:test2","rev":"1-9432c34ac5a6b82e7cc7ce6faaa45452"}

and update the article with test2:

    $ curl -XPUT test2:secret@localhost:5984/testdb1/article1 -d @-
    {"_rev":"1-0cb3999630b9b1ed3846029f66d4d62b", "author":"test1"}
    ^D
    {"ok":true,"id":"article1","rev":"2-208da7abbc6ef27dde976be716a13cd6"}

It is fun, isn't it?

The final question is how to implement private messaging within the BBS website?

There might be a lot of ways to do that one of which is to store all the private messages in another non-public database and use a middleware to control the read access.

The reason couchdb doesn't have document-level read access control is because such kind of access control is a kind of business logic but a data logic.

Another idea is to configure nginx proxy to only expose the \_design docs (views) to the app instead of documents direct access. Then you can control the information you want the user to get by designing different views.

## More recipes to be continued...

- configure nginx to publicize couchdb
- integrate oauth
- captcha to anti-spamming

## References

- A good overview of couchdb AA by couchdbase. http://blog.couchbase.com/what%E2%80%99s-new-couchdb-10-%E2%80%94-part-4-security%E2%80%99n-stuff-users-authentication-authorisation-and-permissions
- Office couchdb security features overview. http://wiki.apache.org/couchdb/Security_Features_Overview
- The Validation Functions chapter in the book _couchdb the definitive guide_ als helped. http://guide.couchdb.org/editions/1/en/validation.html

