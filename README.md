About
=====

afew is an initial tagging script for notmuch mail:

* http://notmuchmail.org/
* http://notmuchmail.org/initial_tagging/

Its basic task is to provide automatic tagging each time new mail is registered
with notmuch. In a classic setup, you might call it after 'notmuch new' in an
offlineimap post sync hook.

In addition to more elementary features such as adding tags based on email
headers or maildir folders, handling killed threads and spam, it can do some
heavy magic in order to /learn/ how to initially tag your mails based on their
content.

In sync mode, afew will sync tags back to the underlying maildir structure by
moving tagged mails between folders according to configurable rules.

fyi: afew plays nicely with alot, a GUI for notmuch mail ;)

* https://github.com/pazz/alot


Current NEWS
============

afew is quite young, so expect a few user visible API or configuration
format changes, though I'll try to minimize such disruptive events.

Please keep an eye on NEWS.md for important news.


Features
========

* text classification, magic tags aka the mailing list without server
* spam handling (flush all tags, add spam)
* killed thread handling
* tags posts to lists with `lists`, `$list-id`
* autoarchives mails sent from you
* catchall -> remove `new`, add `inbox`
* can operate on new messages [default], `--all` messages or on custom
  query results
* can sync tags to maildir folders, so your sorting will show on your
  traditional mail server (well, almost ;))
* has a `--dry-run` mode for safe testing
* works with both python2.7 and python3.2


Installation
============

You'll need dbacl for the text classification:

    # aptitude install dbacl

And I'd like to suggest to install afew as your unprivileged user.
If you do, make sure `~/.local/bin` is in your path.

    $ python setup.py install --prefix=~/.local
    $ mkdir -p ~/.config/afew ~/.local/share/afew/categories


Configuration
=============

Make sure that `~/.notmuch-config` reads:

```
[new]
tags=new
```

Put a list of filters into `~/.config/afew/config`:

```
# This is the default filter chain
[SpamFilter]
[ClassifyingFilter]
[KillThreadsFilter]
[ListMailsFilter]
[ArchiveSentMailsFilter]
[InboxFilter]
```

And configure rules to sync your tags back to disk, if you want:
~~~ snip ~~~
[TagSyncher]
folders = INBOX Junk
max_age = 15

#rules
INBOX = 'tag:spam':Junk 'NOT tag:inbox':Archive
Junk = 'NOT tag:spam AND tag:inbox':INBOX 'NOT tag:spam':Archive
~~~ snip ~~~


Commandline help
================

```
$ afew --help
Usage: afew [options] [--] [query]

Options:
  -h, --help            show this help message and exit

  Actions:
    Please specify exactly one action (both update actions can be
    specified simultaniously).

    -t, --tag           run the tag filters
    -l LEARN, --learn=LEARN
                        train the category with the messages matching the
                        given query
    -u, --update        update the categories [requires no query]
    -U, --update-reference
                        update the reference category (takes quite some time)
                        [requires no query]
    -c, --classify      classify each message matching the given query (to
                        test the trained categories)
    -s, --sync-tags     sync tags to maildirs

  Query modifiers:
    Please specify either --all or --new or a query string. The default
    query for the update actions is a random selection of
    REFERENCE_SET_SIZE mails from the last REFERENCE_SET_TIMEFRAME days.

    -a, --all           operate on all messages
    -n, --new           operate on all new messages

  General options:
    -C NOTMUCH_CONFIG, --notmuch-config=NOTMUCH_CONFIG
                        path to the notmuch configuration file [default:
                        $NOTMUCH_CONFIG or ~/.notmuch-config]
    -e ENABLE_FILTERS, --enable-filters=ENABLE_FILTERS
                        filter classes to use, separated by ',' [default:
                        filters specified in afew's config]
    -d, --dry-run       don't change the db [default: False]
    -R REFERENCE_SET_SIZE, --reference-set-size=REFERENCE_SET_SIZE
                        size of the reference set [default: 1000]
    -T DAYS, --reference-set-timeframe=DAYS
                        do not use mails older than DAYS days [default: 30]
    -v, --verbose       be more verbose, can be given multiple times
```


Boring stuff
============

Simulation
----------
Adding `--dry-run` to any `--tag` or `--sync-tags` action prevents
modification of the notmuch db. Add some `-vv` goodness to see some
action.


Initial tagging
---------------
Basic tagging stuff requires no configuration, just run

    $ afew --tag --new

To do this automatically you can add the following hook into your
~/.offlineimaprc:

    postsynchook = ionice -c 3 chrt --idle 0 /bin/sh -c "notmuch new && afew --tag --new"


Tag filters
-----------
Tag filters are plugin-like modules that encapsulate tagging
functionality. There is a filter that handles the archiving of mails
you sent, one that handles spam, one for killed threads, one for
mailing list magic...

The tag filter concept allows you to easily extend afew's tagging
abilities by writing your own filters. Take a look at the default
configuration file (`afew/defaults/afew.config`) for a list of
available filters and how to enable filters and create customized
filter types.


Sync mode
---------
To invoke afew in sync mode, provide the --sync-tags option on the command line.
Sync mode will respect --dry-run, so throw in --verbose and watch what effects
a real run would have.

In sync mode, afew will check all mails (or only recent ones) in the configured
maildir folders, deciding whether they should be moved to another folder. 

The decision is based on rules defined in your config file. A rule is bound to a
source folder and specifies a target folder into which a mail will be moved
that is matched by an associated query.


-- Rules --

First you need to specify which folders should be synced (as a whitespace
separated list): 

~~~ snip ~~~
folders = INBOX Junk
~~~ snip ~~~

Then you have to specify rules that define move actions of the form

<src> = ['<qry>':<dst>]+

Every mail in <src> that matches a <qry> will be moved into the assigned
<dst>.

You can bind as many rules to a maildir folder as you deem necessary. Just add
them as elements of a (whitespace separated) list.

Please note though that you need to specify at least one rule for every folder
given by the 'folders' option and at least one folder to sync in order to use
the sync mode.

~~~ snip ~~~
INBOX = 'tag:spam':Junk
~~~ snip

will bind one rule to the maildir folder 'INBOX' that states that all mails in
said folder that carry (potentially among others) the tag 'spam' are to be moved
into the folder 'Junk'.

With <qry> being an arbitrary notmuch query, you have the power to construct
arbitrarily flexible rules. You can check for the absence of tags and look out
for combinations of tags:

~~~ snip ~~~
Junk = 'NOT tag:spam AND tag:inbox':INBOX 'NOT tag:spam':Archive
~~~ snip ~~~

Above rules will move all mails in 'Junk' that don't have the 'spam' tag but an
'inbox' tag into the directory 'INBOX', all other mails not tagged as spam into
'Archive'.

Note how this example binds two rules to the source folder and how the sequence
in which the rules have been specified matters: If you were to specify the
second rule first, it would match *all* mails not tagged as 'spam' and the then
second rule would never be triggered. More on that under 'Limitations'.


-- Max Age --

You can limit the age of mails you wanna sync. By providing

~~~ snip ~~~
max_age = 15
~~~ snip ~~~

afew will only check mails at most 15 days old.


-- Limitations --

(1) Rules don't manipulate tags.

~~~ snip ~~~
INBOX = 'NOT tag:inbox':Archive
Junk = 'NOT tag:spam':INBOX
~~~ snip ~~~

Above combination of rules might prove tricky, since you might expect de-spammed
mails to end up in 'INBOX'. But since the 'Junk' rule will *not* add an 'inbox'
tag, the next run in sync mode might very well move the matching mails into
'Archive'.

Then again, if you remove the 'spam' tag and do not set an 'inbox' tag, how
would you come to expect the mail would end up in your INBOX folder after a
sync? ;)

(2) There is no 1:1 mapping between folders and tags. And that's a feature. If
you tag a mail with two tags and there is a rule for each of them, the rule you
specified first will apply.

(3) The same principle applies if you bind two rules to a source of which one is
a prefix to the other. If rule A is more generic than rule B (as in 'easier to
satisfy', less specific), you want to give it *after* rule B, or else it will
consume all mails that would trigger rule B before that more specific rule may
apply.


The real deal
=============

Let's train on an existing tag `spam`:

    $ afew --learn spam -- tag:spam

Let's build the reference category. This is important to reduce the
false positive rate. This may take a while...

    $ afew --update-reference

And now let's create a new tag from an arbitrary query result:

    $ afew -vv --learn sourceforge -- sourceforge

Let's see how good the classification is:

    $ afew --classify -- tag:inbox and not tag:killed
    Sergio López <slpml@sinrega.org> (2011-10-08) (bug-hurd inbox lists unread) --> no match
    Patrick Totzke <reply+i-1840934-9a702d09342dca2b120126b26b008d0deea1731e@reply.github.com> (2011-10-08) (alot inbox lists) --> alot
    [...]

As soon as you trained some categories, afew will automatically
tag your new mails using the classifier. If you want to disable this
feature, either use the `--enable-filters` option to override the default
set of filters or remove the files in your afew state dir:

    $ ls ~/.local/share/afew/categories
    alot juggling  reference_category  sourceforge  spam

You need to update the category files periodically. I'd suggest to run

    $ afew --update

on a weekly and

    $ afew --update-reference

on a monthly basis.



Have fun :)
