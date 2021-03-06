# Django Postgres access control (DPAC)

DPAC provides very **granular row and column level role based access control** for your Django & Postgres applications.
In a nutshell, DPAC enable you to utilize the vast access control capabilities already built into PostgreSQL in a way which also feels comfortable from the Django side.

## What problem does it solve

Django has a built-in users and groups management system as part of the `django.contrib.auth` package.
However, there is no built-in solution for enforcing a rule such as "Users belonging into the `librerian` group can see the titles and ISBN codes of books in the library they are working for." There is not even a way to enforce the simpler rule "All users can see their own orders."

Of course you can write complex filtering functions in your views, but there is no way to enforce the same rules all across the application. And by to burdon yourself with enforcing the rules in an imperative way, if declerative rules are much easier to understand and maitain?

## How it works under to hood

PostgreSQL ships with a highly sophisticated set of access control features including both the standard SQL options like `GRANT` statements as well as its own additions like row level security.

The problem is that Django does not provide a convenient way to leverage these features. So you are faced with a hard dilemma: to give up the Django models and write custom SQL queries or to be cut off the valuable tools your database could offer you. And even if you would be all but happy with writting all SQL queryes by hand, most likly your co-workers could not keep up with you.

DPAC provides a bridge between Postgres And Django.

Both Postgres and Django think in terms of users and groups (or 'roles' in Postgres). However, for both of them the same terms means different things. For Django, its user is an instance of the `User` (`AUTH_USER_MODEL` to be precise) class, but for Postgres it is just a regular row in a regular table. The same goes with groups. A Postgres cluster might have many users belonging to multiple roles, but Django only has one user hardcoded into its settings and since it executes all the queries in the right of the same user, it does not benefit from what the database has to offer.

To remedy this, DPAC does the following:
* it creates a one-to-one correspondence between a Django user and a database user. E.g. a Django user `smith` has a corresponding DB user `user_smith`. However, not all DB users must have a corresponding DJango users, for there might be DB users for administrators, other applications, backups, and other purposes not related to Django.
* it creaes a one-to-one correspondence between a Django gorup and a database role. E.g. a Django group `librerians` would correspond to the `role_librerians` in the DB. Again, the opposite in not necesarily true.
  * these relationships are implemented as triggers in the database, which will fire once a user or group gets created or modified.
* it provides a way for the Django queries to drop their superuser privileges and execute with the prmissions of some other user. E.g. Django might use its superuser privileges to run the migrations, but switch into the privileges of `user_smith` to retreave all books that Mr Smith is allowed to see.
  * this is achieved by appending `SET SESSION AUTHORIZATION {username};` at the beginning of each of the lower priviledged queryes and executing `SET SESSION AUTHORIZATION DEFAULT;` to return to superuser priviledges.
* its provides a declarative way to list the access-related SQL clauses in the Django model definition and runs them during the `migrate` command.

## How it looks on the Django side

This is how it looks like:

First, add the access granting clauses to your Django model:

```py
from django.db import models
from django import settings

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.PROTECT)
    notes = models.TextField()

    class Access:
        permissions = [
            # Coulmn level security: allow all members of the `author` group to see the `title` and `author` of the books
            "GRANT SELECT (title, author) ON books_book TO author",
            # Row level security: allow members of the `author` group to see their own books
            "CREATE POLICY authors_can_see_their_books ON books_book USING (author_id = current_user) TO author;"
        ]
```

By default the DB requests are made as the user specified in the settings file.
You can move between different sets of privileges using context managers:

```py
Book.objects.all().only("title") # executed in superuser privileges
with switch_user(db_username(request.user)):
    Book.objects.all().only("title") # executed in the logged in user privileges
```

## Why to implement access control in the DB, not in the application layer

It would also be possible to write access control code yourself and put it e.g. into the Django managers layer. However, having it in the DB will provide numerour advantages:
1. Consistency: all common set of permissions is enforced across the application. The same rules will also apply if the data is accessed in some other way without going through the application.
2. Adaptability: PostgreSQL has a lot of access control mechanisms built in. They provide much more granularity and versatility compared to the options provided by Django.
3. Security: in case a bug in the application (or any of its third-party dependencies) makes it vulnerable to SQL-injection, even if the user crafts a mallicious query the database makse sure that the query cannot do anything the user is not permitted to do.
4. Maintainability: The database system will likely overlive any of the application development platforms and maybe even the languages the platforms are based on. PostgeSQL is a very mature and commonly used project which will we developed and supported for a long time. Even if PostgreSQL will eventually be adundoned in the distant future, the parts of the code which adhere to the SQL standard will continue to work on other database systems.
5. Speed: doing the permission checks in the database is much faster than transporting a lot of data to the application and doing the filtering there. This becomes more and more relevant as the application starts to grow both in terms of the complexity and the amount of data stored in the database.
6. ACIDRain attacks: the Django code sees only one request at a time, the Django process will not be aware of any other queries made by other users. This open up the possibility to ACIDRain type of attacks. The database sees all the queries at the same time and provides methods to prevent cuncurrent access from causeing trouble.


## Project status

DPAC is currently under active develpment and has not reached a stable version 1.0 yet.

