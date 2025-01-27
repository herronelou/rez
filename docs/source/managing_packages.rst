=================
Managing packages
=================

Ignoring Packages
=================

Packages can be ignored. When this happens, the package is still present in its
repository, but it will not be visible to the rez API nor to any newly resolved
runtimes. Any runtimes that are currently using an ignored package are unaffected,
since the package's payload has not been removed.

To ignore a package via commandline:

.. code-block:: console

   $ # you need to specify the repo, but you'll be shown a list if you don't
   $ rez-pkg-ignore foo-1.2.3
   No action taken. Run again, and set PATH to one of:
   filesystem@/home/ajohns/packages

   $ rez-pkg-ignore foo-1.2.3 filesystem@/home/ajohns/packages
   Package is now ignored and will not be visible to resolves

Via API:

.. code-block:: python

   >>> from rez.package_repository import package_repository_manager
   >>>
   >>> repo_path = "filesystem@/home/ajohns/packages"
   >>> repo = package_repository_manager.get_repository(repo_path)
   >>> repo.ignore_package("foo", "1.2.3")
   1  # -1: pkg not found; 0: pkg already ignored; 1: pkg ignored

Both of these options generate a :file:`.ignore{{version}}` file (e.g.
``.ignore3.1.2``) next to the package version directory.

You can also do the reverse (ie unignore a package). Use the :option:`-u <rez-pkg-ignore -u>` option of
:ref:`rez-pkg-ignore`, or the :meth:`~rez.package_repository.PackageRepository.unignore_package` method on the package repository
object.

Copying Packages
================

Packages can be copied from one :ref:`package repository <package-repositories-concept>`
to another, like so:

Via commandline:

.. code-block:: console

   $ rez-cp --dest-path /svr/packages2 my_pkg-1.2.3

Via API:

.. code-block:: python

   >>> from rez.package_copy import copy_package
   >>> from rez.packages import get_latest_package
   >>>
   >>> p = get_latest_package("python")
   >>> p
   Package(FileSystemPackageResource({'location': '/home/ajohns/packages', 'name': 'python', 'repository_type': 'filesystem', 'version': '3.7.4'}))
   >>>
   >>> r = copy_package(p, "./repo2")
   >>>
   >>> print(pprint.pformat(r))
   {
      'copied': [
         (
               Variant(FileSystemVariantResource({'location': '/home/ajohns/packages', 'name': 'python', 'repository_type': 'filesystem', 'index': 0, 'version': '3.7.4'})),
               Variant(FileSystemVariantResource({'location': '/home/ajohns/repo2', 'name': 'python', 'repository_type': 'filesystem', 'index': 0, 'version': '3.7.4'}))
         )
      ],
      'skipped': []
   }

Copying packages is actually done one variant at a time, and you can copy some
variants of a package if you want, rather than the entire package. The API call's
return value shows what variants were copied. The 2-tuple in ``copied`` lists the
source (the variant that was copied from) and destination (the variant that was
created) respectively.

.. danger::
   Do not simply copy package directories on disk.
   You should always use :ref:`rez-cp` or use the API. Copying directly on disk is bypassing rez and
   this can cause problems such as a stale resolve cache. Using :ref:`rez-cp` and the API give
   you more control anyway.

.. _enabling-package-copying:

Enabling Package Copying
------------------------

Copying packages is enabled by default, however you're also able to specify which
packages are and are not *relocatable*, for much the same reasons as given
:ref:`here <enabling-package-caching>`.

You can mark a package as non-relocatable by setting :attr:`relocatable`
to ``False`` in its package definition file. There are also config settings that affect relocatability
in the event that relocatable is not defined in a package's definition. For example,
see :data:`default_relocatable`, :data:`default_relocatable_per_package`
and :data:`default_relocatable_per_repository`.

Attempting to copy a non-relocatable package will raise a :exc:`~rez.exceptions.PackageCopyError`.
However, note that there is a ``force`` option that will override this. Use at
your own risk.

.. _moving-packages:

Moving Packages
===============

Packages can be moved from one :ref:`package repository <package-repositories-concept>`
to another. Be aware that moving a package does not actually delete the source
package however. Instead, the source package is hidden (ignored). It is up to
you to delete it at some later date.

To move a package via commandline:

.. code-block:: console

   $ rez-mv --dest-path /packages2 python-3.7.4 /packages

Via API:

.. code-block:: python

   >>> from rez.package_move import move_package
   >>> from rez.packages import get_package_from_repository
   >>>
   >>> p = get_package_from_repository("python", "3.7.4", "/packages")
   >>> p
   Package(FileSystemPackageResource({'location': '/packages', 'name': 'python', 'repository_type': 'filesystem', 'version': '3.7.4'}))
   >>>
   >>> new_p = move_package(p, "/packages2")
   >>> new_p
   Package(FileSystemPackageResource({'location': '/packages2', 'name': 'python', 'repository_type': 'filesystem', 'version': '3.7.4'}))
   >>>
   >>> p = get_package_from_repository("python", "3.7.4", "/packages")
   >>> p
   None

Be aware that a non-relocatable package is also not movable (see
:attr:`here <relocatable>`. Like package
copying, there is a ``force`` option to move it regardless.

A typical reason you might want to move a package is to archive packages that are
no longer in use. In this scenario, you would move the package to some archival
package repository. In case an old runtime needs to be resurrected, you would add
this archival repository to the packages path before performing the resolve.

.. note::
   You will probably want to use the :option:`--keep-timestamp <rez-mv --keep-timestamp>` option when doing this,
   otherwise rez will think the package did not exist prior to its archival date.

.. _removing-packages:

Removing Packages
=================

Packages can be removed. This is different from ignoring. The package and its
payload is deleted from storage, whereas ignoring just hides it. It is not
possible to un-remove a package.

To remove a package via commandline:

.. code-block:: console

   $ rez-rm --package python-3.7.4 /packages

Via API:

.. code-block:: python

   >>> from rez.package_remove import remove_package
   >>>
   >>> remove_package("python", "3.7.4", "/packages")

During the removal process, package versions will first be ignored so that
partially-deleted versions are not visible.

It can be useful to ignore packages that you don't want to use anymore, and
actually remove them at a later date. This gives you a safety buffer in case
current runtimes are using the package. They won't be affected if the package is
ignored, but could break if it is removed.

To facilitate this workflow, :ref:`rez-rm` lets you remove all packages that have
been ignored for longer than N days (using the timestamp of the
:file:`.ignore{{version}}` file). Here we remove all packages that have been ignored
for 30 days or longer:

.. code-block:: console

   $ rez-rm --ignored-since=30 -v
   14:47:09 INFO     Searching filesystem@/home/ajohns/packages...
   14:47:09 INFO     Removed python-3.7.4 from filesystem@/home/ajohns/packages
   1 packages were removed.

Via API:

.. code-block:: python

   >>> from rez.package_remove import remove_packages_ignored_since
   >>>
   >>> remove_packages_ignored_since(days=30)
   1

.. _package-caching:

Package Caching
===============

Package caching is a feature that copies package payloads onto local disk in
order to speed up runtime environments. For example, if your released packages
reside on shared storage (which is common), then running say, a Python process,
will fetch all source from the shared storage across your network. The point of
the cache is to copy that content locally instead, and avoid the network cost.

.. note::
   Please note: Package caching does **NOT** cache package
   definitions. Only their payloads (ie, the package root directory).

Build behavior
--------------

Package caching during a package build is disabled by default. To enable caching during
a package build, you can set :data:`package_cache_during_build` to True.

.. _enabling-package-caching:

Enabling Package Caching
========================

Package caching is not enabled by default. To enable it, you need to configure
:data:`cache_packages_path` to specify a path to
store the cache in.

You also have granular control over whether an individual package will or will
not be cached. To make a package cachable, you can set :attr:`cachable`
to False in its package definition file. Reasons you may *not* want to do this include
packages that are large, or that aren't relocatable because other compiled packages are
linked to them in a way that doesn't support library relocation.

There are also config settings that affect cachability in the event that :attr:`cachable`
is not defined in a package's definition. For example, see
:data:`default_cachable`, :data:`default_cachable_per_package`
and :data:`default_cachable_per_repository`.

Note that you can also disable package caching on the command line, using
:option:`rez-env --no-pkg-cache`.

Verifying
---------

When you resolve an environment, you can see which variants have been cached by
noting the ``cached`` label in the right-hand column of the :ref:`rez-context` output,
as shown below:

.. code-block:: console

   $ rez-env Flask

   You are now in a rez-configured environment.

   requested packages:
   Flask
   ~platform==linux   (implicit)
   ~arch==x86_64      (implicit)
   ~os==Ubuntu-16.04  (implicit)

   resolved packages:
   Flask-1.1.2         /home/ajohns/package_cache/Flask/1.1.2/d998/a                                     (cached)
   Jinja2-2.11.2       /home/ajohns/package_cache/Jinja2/2.11.2/6087/a                                   (cached)
   MarkupSafe-1.1.1    /svr/packages/MarkupSafe/1.1.1/d9e9d80193dcd9578844ec4c2c22c9366ef0b88a
   Werkzeug-1.0.1      /home/ajohns/package_cache/Werkzeug/1.0.1/fe76/a                                  (cached)
   arch-x86_64         /home/ajohns/package_cache/arch/x86_64/6450/a                                     (cached)
   click-7.1.2         /home/ajohns/package_cache/click/7.1.2/0da2/a                                     (cached)
   itsdangerous-1.1.0  /home/ajohns/package_cache/itsdangerous/1.1.0/b23f/a                              (cached)
   platform-linux      /home/ajohns/package_cache/platform/linux/9d4d/a                                  (cached)
   python-3.7.4        /home/ajohns/package_cache/python/3.7.4/ce1c/a                                    (cached)

For reference, cached packages also have their original payload location stored to
an environment variable like so:

.. code-block:: console

   $ echo $REZ_FLASK_ORIG_ROOT
   /svr/packages/Flask/1.1.2/88a70aca30cb79a278872594adf043dc6c40af99

How it Works
------------

Package caching actually caches :doc:`variants`, not entire packages. When you perform
a resolve, or source an existing context, the variants required are copied to
local disk asynchronously (if they are cachable), in a separate process called
:ref:`rez-pkg-cache`. This means that a resolve will not necessarily use the cached
variants that it should, the first time around. Package caching is intended to have
a cumulative effect, so that more cached variants will be used over time. This is
a tradeoff to avoid blocking resolves while variant payloads are copied across
your network (and that can be a slow process).

Note that a package cache is **not** a package repository. It is simply a store
of variant payloads, structured in such a way as to be able to store variants from
any package repository, into the one shared cache.

Variants that are cached are assumed to be immutable. No check is done to see if
a variant's payload has changed, and needs to replace an existing cache entry. So
you should **not** enable caching on package repositories where packages may get
overwritten. It is for this reason that caching is disabled for local packages by
default (see :data:`package_cache_local`).

Commandline Tool
----------------

Inspection
++++++++++

Use the :ref:`rez-pkg-cache` tool to view the state of the cache, and to perform
warming and deletion operations. Example output follows:

.. code-block:: console

   $ rez-pkg-cache
   Package cache at /home/ajohns/package_cache:

   status   package             variant uri                                             cache path
   ------   -------             -----------                                             ----------
   cached   Flask-1.1.2         /svr/packages/Flask/1.1.2/package.py[0]         /home/ajohns/package_cache/Flask/1.1.2/d998/a
   cached   Jinja2-2.11.2       /svr/packages/Jinja2/2.11.2/package.py[0]       /home/ajohns/package_cache/Jinja2/2.11.2/6087/a
   cached   Werkzeug-1.0.1      /svr/packages/Werkzeug/1.0.1/package.py[0]      /home/ajohns/package_cache/Werkzeug/1.0.1/fe76/a
   cached   arch-x86_64         /svr/packages/arch/x86_64/package.py[]          /home/ajohns/package_cache/arch/x86_64/6450/a
   cached   click-7.1.2         /svr/packages/click/7.1.2/package.py[0]         /home/ajohns/package_cache/click/7.1.2/0da2/a
   cached   itsdangerous-1.1.0  /svr/packages/itsdangerous/1.1.0/package.py[0]  /home/ajohns/package_cache/itsdangerous/1.1.0/b23f/a
   cached   platform-linux      /svr/packages/platform/linux/package.py[]       /home/ajohns/package_cache/platform/linux/9d4d/a
   copying  python-3.7.4        /svr/packages/python/3.7.4/package.py[0]        /home/ajohns/package_cache/python/3.7.4/ce1c/a
   stalled  MarkupSafe-1.1.1    /svr/packages/MarkupSafe/1.1.1/package.py[1]    /home/ajohns/package_cache/MarkupSafe/1.1.1/724c/a

Each variant is stored into a directory based on a partial hash of that variant's
unique identifier (its "handle"). The package cache is thread and multiprocess
proof, and uses a file lock to control access where necessary.

Cached variants have one of the following statuses at any given time:

* **copying**: The variant is in the process of being copied into the cache, and is not
  yet available for use;
* **cached**: The variant has been cached and is ready for use;
* **stalled**: The variant was getting copied, but something went wrong and there is
  now a partial copy present (but unused) in the cache.

Logging
+++++++

Caching operations are stored into logfiles within the cache directory. To view:

.. code-block:: console

   $ rez-pkg-cache --logs
   rez-pkg-cache 2020-05-23 16:17:45,194 PID-29827 INFO Started daemon
   rez-pkg-cache 2020-05-23 16:17:45,201 PID-29827 INFO Started caching of variant /home/ajohns/packages/Werkzeug/1.0.1/package.py[0]...
   rez-pkg-cache 2020-05-23 16:17:45,404 PID-29827 INFO Cached variant to /home/ajohns/package_cache/Werkzeug/1.0.1/fe76/a in 0.202576 seconds
   rez-pkg-cache 2020-05-23 16:17:45,404 PID-29827 INFO Started caching of variant /home/ajohns/packages/python/3.7.4/package.py[0]...
   rez-pkg-cache 2020-05-23 16:17:46,006 PID-29827 INFO Cached variant to /home/ajohns/package_cache/python/3.7.4/ce1c/a in 0.602037 seconds

Cleaning The Cache
++++++++++++++++++

Cleaning the cache refers to deleting variants that are stalled or no longer in use.
It isn't really possible to know whether a variant is in use, so there is a
configurable :data:`package_cache_max_variant_days`
setting, that will delete variants that have not been used (ie that have not appeared
in a created or sourced context) for more than N days.

You can also manually remove variants from the cache using :option:`rez-pkg-cache -r`.
Note that when you do this, the variant is no longer available in the cache,
however it is still stored on disk. You must perform a clean (:option:`rez-pkg-cache --clean`)
to purge unused cache files from disk.

You can use the :data:`package_cache_clean_limit`
setting to asynchronously perform some cleanup every time the cache is updated. If
you do not use this setting, it is recommended that you set up a cron or other form
of execution scheduler, to run :option:`rez-pkg-cache --clean` periodically. Otherwise,
your cache will grow indefinitely.

Lastly, note that a stalled variant will not attempt to be re-cached until it is
removed by a clean operation. Using :data:`package_cache_clean_limit` will not clean
stalled variants either, as that could result in a problematic variant getting
cached, then stalled, then deleted, then cached again and so on. You must run
:option:`rez-pkg-cache --clean` to delete stalled variants.
