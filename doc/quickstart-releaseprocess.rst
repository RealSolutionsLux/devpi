.. include:: links.rst

.. _quickstart-releaseprocess:

Quickstart: uploading, testing, pushing releases 
------------------------------------------------

This quickstart document walks you through setting up a self-contained
pypi release upload, testing and staging system for your Python packages.

Installing devpi client and server
++++++++++++++++++++++++++++++++++

We want to run the full devpi system on our laptop::

    pip install -U devpi-web devpi-client

Note that the ``devpi-web`` package will pull in the core
``devpi-server`` package.  If you don't want a web interface you 
can just install the latter only.

initializing a basic server and index
+++++++++++++++++++++++++++++++++++++

We need to perform a couple of steps to get an index
where we can upload and test packages:

- start a background devpi-server at ``http://localhost:3141``

- configure the client-side tool ``devpi`` to connect to the newly
  started server

- create and login a user, using as defaults your current login name 
  and an empty password.

- create an index and directly use it.

So let's first start a background server::

    $ devpi-server --start --init
    2016-12-14 11:15:22,559 INFO  NOCTX Loading node info from /tmp/home/.devpi/server/.nodeinfo
    2016-12-14 11:15:22,560 INFO  NOCTX generated uuid: 728ebed5646b4098b4b5e6e0c686ac51
    2016-12-14 11:15:22,570 INFO  NOCTX wrote nodeinfo to: /tmp/home/.devpi/server/.nodeinfo
    2016-12-14 11:15:22,572 INFO  NOCTX DB: Creating schema
    2016-12-14 11:15:22,580 INFO  [Wtx-1] setting password for user 'root'
    2016-12-14 11:15:22,580 INFO  [Wtx-1] created user 'root' with email None
    2016-12-14 11:15:22,580 INFO  [Wtx-1] created root user
    2016-12-14 11:15:22,581 INFO  [Wtx-1] created root/pypi index
    2016-12-14 11:15:22,583 INFO  [Wtx-1] fswriter0: committed: keys: '.config','root/.config'
    starting background devpi-server at http://localhost:3141
    /tmp/home/.devpi/server/.xproc/devpi-server$ /home/devpi/devpi/bin/devpi-server
    process 'devpi-server' started pid=23968
    devpi-server process startup detected
    logfile is at /tmp/home/.devpi/server/.xproc/devpi-server/xprocess.log

Then we point the devpi client to it::

    $ devpi use http://localhost:3141
    using server: http://localhost:3141/ (not logged in)
    no current index: type 'devpi use -l' to discover indices
    ~/.pydistutils.cfg     : http://localhost:4040/alice/dev/+simple/
    ~/.pip/pip.conf        : http://localhost:4040/alice/dev/+simple/
    ~/.buildout/default.cfg: http://localhost:4040/alice/dev/+simple/
    always-set-cfg: no

Then we add our own "testuser"::

    $ devpi user -c testuser password=123
    user created: testuser

Then we login::

    $ devpi login testuser --password=123
    logged in 'testuser', credentials valid for 10.00 hours

And create a "dev" index, telling it to use the ``root/pypi`` cache as a base
so that all of pypi.python.org packages will appear on that index::

    $ devpi index -c dev bases=root/pypi
    http://localhost:3141/testuser/dev:
      type=stage
      bases=root/pypi
      volatile=True
      acl_upload=testuser
      mirror_whitelist=
      pypi_whitelist=

Finally we use the new index::

    $ devpi use testuser/dev
    current devpi index: http://localhost:3141/testuser/dev (logged in as testuser)
    ~/.pydistutils.cfg     : http://localhost:4040/alice/dev/+simple/
    ~/.pip/pip.conf        : http://localhost:4040/alice/dev/+simple/
    ~/.buildout/default.cfg: http://localhost:4040/alice/dev/+simple/
    always-set-cfg: no

We are now ready to go for uploading and testing packages.

.. _`quickstart_release_steps`:

devpi install: installing a package
+++++++++++++++++++++++++++++++++++

We can now use the ``devpi`` command line client to trigger a ``pip
install`` of a pypi package using the index from our already running server::

    $ devpi install pytest
    -->  /home/devpi/devpi/doc$ /tmp/docenv/bin/pip install -U -i http://localhost:3141/testuser/dev/+simple/ pytest  [PIP_PRE=1,PIP_USE_WHEEL=1]
    Collecting pytest
      Downloading http://localhost:3141/root/pypi/+f/790/888ee21cfd19c/pytest-3.0.5-py2.py3-none-any.whl (170kB)
    Requirement already up-to-date: py>=1.4.29 in /tmp/docenv/lib/python3.5/site-packages (from pytest)
    Installing collected packages: pytest
    Successfully installed pytest-3.0.5

The ``devpi install`` command configured a pip call, using the
pypi-compatible ``+simple/`` page on our ``testuser/dev`` index for
finding and downloading packages.  The ``pip`` executable was searched
in the ``PATH`` and found in ``docenv/bin/pip``.

Let's check that ``pytest`` was installed correctly::

    $ py.test --version
    This is pytest version 3.0.5, imported from /tmp/docenv/lib/python3.5/site-packages/pytest.py

You may invoke the ``devpi install`` command a second time which will
even work when you have no network.

.. _`devpi upload`:

devpi upload: uploading one or more packages
++++++++++++++++++++++++++++++++++++++++++++

We are going to use ``devpi`` command line tool facilities for 
performing uploads (you can also 
:ref:`use plain setup.py <configure pypirc>`).

Let's verify we are logged in to the correct index::

    $ devpi use
    current devpi index: http://localhost:3141/testuser/dev (logged in as testuser)
    ~/.pydistutils.cfg     : http://localhost:4040/alice/dev/+simple/
    ~/.pip/pip.conf        : http://localhost:4040/alice/dev/+simple/
    ~/.buildout/default.cfg: http://localhost:4040/alice/dev/+simple/
    always-set-cfg: no

Now go to the directory of a ``setup.py`` file of one of your projects  
(we assume it is named ``example``) to build and upload your package
to our ``testuser/dev`` index::

    example $ devpi upload
    using workdir /tmp/devpi1
    copied repo /home/devpi/devpi/.hg to /tmp/devpi1/upload/devpi/.hg
    pre-build: cleaning /home/devpi/devpi/doc/example/dist
    -->  /tmp/devpi1/upload/devpi/doc/example$ /tmp/docenv/bin/python setup.py sdist --formats gztar 
    built: /home/devpi/devpi/doc/example/dist/example-1.0.tar.gz [SDIST.TGZ] 0.689kb
    register example-1.0 to http://localhost:3141/testuser/dev/
    file_upload of example-1.0.tar.gz to http://localhost:3141/testuser/dev/

There are three triggered actions:

- detection of a mercurial repository, leading to copying all versioned
  files to a temporary work dir.  If you are not using mercurial,
  the copy-step is skipped and the upload operates directly on your source
  tree.

- registering the ``example`` release as defined in ``setup.py`` to 
  our current index

- building and uploading a ``gztar`` formatted release file from the
  workdir to the current index (using a ``setup.py`` invocation under
  the hood).

We can now install the freshly uploaded package::

    $ devpi install example
    -->  /home/devpi/devpi/doc$ /tmp/docenv/bin/pip install -U -i http://localhost:3141/testuser/dev/+simple/ example  [PIP_USE_WHEEL=1,PIP_PRE=1]
    Collecting example
      Downloading http://localhost:3141/testuser/dev/+f/4b9/381a0cc2c090c/example-1.0.tar.gz
    Building wheels for collected packages: example
      Running setup.py bdist_wheel for example: started
      Running setup.py bdist_wheel for example: finished with status 'done'
      Stored in directory: /tmp/home/.cache/pip/wheels/0f/67/6c/852cc9af0cc20562236ae2af197a12c2b7847a055d66f3c748
    Successfully built example
    Installing collected packages: example
    Successfully installed example-1.0

This installed your just uploaded package from the ``testuser/dev``
index where we previously uploaded the package.

.. note::

    ``devpi upload`` allows to simultanously upload multiple different 
    formats of your release files such as ``sdist.zip`` or ``bdist_egg``.
    The default is ``sdist.tgz``.



devpi test: testing an uploaded package
+++++++++++++++++++++++++++++++++++++++

If you have a package which uses tox_ for testing you may now invoke::

    $ devpi test example  # package needs to contain tox.ini
    received http://localhost:3141/testuser/dev/+f/4b9/381a0cc2c090c/example-1.0.tar.gz
    unpacking /tmp/devpi-test2/downloads/example-1.0.tar.gz to /tmp/devpi-test2/targz
    /tmp/devpi-test2/targz/example-1.0$ tox --installpkg /tmp/devpi-test2/downloads/example-1.0.tar.gz -i ALL=http://localhost:3141/testuser/dev/+simple/ --recreate --result-json /tmp/devpi-test2/targz/toxreport.json -c /tmp/devpi-test2/targz/example-1.0/tox.ini
    python create: /tmp/devpi-test2/targz/example-1.0/.tox/python
    python installdeps: pytest
    python inst: /tmp/devpi-test2/downloads/example-1.0.tar.gz
    python installed: example==1.0,py==1.4.31,pytest==3.0.5
    python runtests: PYTHONHASHSEED='3252273403'
    python runtests: commands[0] | py.test
    ============================= test session starts ==============================
    platform linux -- Python 3.5.1+, pytest-3.0.5, py-1.4.31, pluggy-0.4.0
    rootdir: /tmp/devpi-test2/targz/example-1.0, inifile: 
    collected 1 items
    
    test_example.py .
    
    =========================== 1 passed in 0.01 seconds ===========================
    ___________________________________ summary ____________________________________
      python: commands succeeded
      congratulations :)
    wrote json report at: /tmp/devpi-test2/targz/toxreport.json
    posting tox result data to http://localhost:3141/testuser/dev/+f/4b9/381a0cc2c090c/example-1.0.tar.gz
    successfully posted tox result data

Here is what happened:

- devpi got the latest available version of ``example`` from the current
  index

- it unpacked it to a temp dir, found the ``tox.ini`` and then invoked
  tox, pointing it to our ``example-1.0.tar.gz``, forcing all installations
  to go through our current ``testuser/dev/+simple/`` index and instructing
  it to create a ``json`` report.

- after all tests ran, we send the ``toxreport.json`` to the devpi server
  where it will be attached precisely to our release file.
 
We can verify that the test status was recorded via::

    $ devpi list example
    http://localhost:3141/testuser/dev/+f/4b9/381a0cc2c090c/example-1.0.tar.gz
    devpi-testing linux   python     3.5.1+ tests passed

.. versionadded:: 2.6

With ``--index`` you can get the release from another index. Full URLs to
another devpi-server are also supported.

.. note::

    Since version 2.2.0 testing of universal wheels is supported if there
    also is an sdist which contains the neccessary tox.ini and tests files.
    Wheels typically don't contain them as they are a pure installation
    package.

devpi push: staging a release to another index
++++++++++++++++++++++++++++++++++++++++++++++

Once you are happy with a release file you can push it either
to another devpi-managed index or to an outside pypi index server.

Let's create another ``staging`` index::

    $ devpi index -c staging volatile=False
    http://localhost:3141/testuser/staging:
      type=stage
      bases=
      volatile=False
      acl_upload=testuser
      mirror_whitelist=
      pypi_whitelist=

We created a non-volatile index which means that one can not 
overwrite or delete release files. See :ref:`non_volatile_indexes` for more info
on this setting.

We can now push the ``example-1.0.tar.gz`` from above to
our ``staging`` index::

    $ devpi push example==1.0 testuser/staging
       200 register example 1.0 -> testuser/staging
       200 store_releasefile testuser/staging/+f/4b9/381a0cc2c090c/example-1.0.tar.gz
       200 store_toxresult testuser/staging/+f/4b9/381a0cc2c090c/example-1.0.tar.gz.toxresult0

This will determine all files on our ``testuser/dev`` index belonging to
the specified ``example==1.0`` release and copy them to the
``testuser/staging`` index. 

devpi push: releasing to an external index
++++++++++++++++++++++++++++++++++++++++++

Let's check again our current index::

    $ devpi use
    current devpi index: http://localhost:3141/testuser/dev (logged in as testuser)
    ~/.pydistutils.cfg     : http://localhost:4040/alice/dev/+simple/
    ~/.pip/pip.conf        : http://localhost:4040/alice/dev/+simple/
    ~/.buildout/default.cfg: http://localhost:4040/alice/dev/+simple/
    always-set-cfg: no

Let's now use our ``testuser/staging`` index::

    $ devpi use testuser/staging
    current devpi index: http://localhost:3141/testuser/staging (logged in as testuser)
    ~/.pydistutils.cfg     : http://localhost:4040/alice/dev/+simple/
    ~/.pip/pip.conf        : http://localhost:4040/alice/dev/+simple/
    ~/.buildout/default.cfg: http://localhost:4040/alice/dev/+simple/
    always-set-cfg: no

and check the test result status again::

    $ devpi list example
    http://localhost:3141/testuser/staging/+f/4b9/381a0cc2c090c/example-1.0.tar.gz
    devpi-testing linux   python     3.5.1+ tests passed

Good, the test result status is still available after the push
from the last step.

We may now decide to push this release to an external
pypi-style index which we have configured in the ``.pypirc`` file::

    $ devpi push example-1.0 pypi:testrun
    no pypirc file found at: /tmp/home/.pypirc

this will push all release files of the ``example-1.0`` release
to the external ``testrun`` index server, using credentials
and the URL found in the ``pypi`` section in your
``.pypirc``.

index inheritance re-configuration
++++++++++++++++++++++++++++++++++

At this point we have the ``example-1.0`` release and release file
on both the ``testuser/dev`` and ``testuser/staging`` indices.
If we rather want to always use staging packages in our development
index, we can reconfigure the inheritance 
``bases`` for ``testuser/dev``::

    $ devpi index testuser/dev bases=testuser/staging
    /testuser/dev changing bases: ['testuser/staging']
    http://localhost:3141/testuser/dev:
      type=stage
      bases=testuser/staging
      volatile=True
      acl_upload=testuser
      mirror_whitelist=
      pypi_whitelist=

If we now switch back to using ``testuser/dev``::

    $ devpi use testuser/dev
    current devpi index: http://localhost:3141/testuser/dev (logged in as testuser)
    ~/.pydistutils.cfg     : http://localhost:4040/alice/dev/+simple/
    ~/.pip/pip.conf        : http://localhost:4040/alice/dev/+simple/
    ~/.buildout/default.cfg: http://localhost:4040/alice/dev/+simple/
    always-set-cfg: no

and look at our example release files::

    $ devpi list example
    http://localhost:3141/testuser/dev/+f/4b9/381a0cc2c090c/example-1.0.tar.gz
    devpi-testing linux   python     3.5.1+ tests passed
    http://localhost:3141/testuser/staging/+f/4b9/381a0cc2c090c/example-1.0.tar.gz
    devpi-testing linux   python     3.5.1+ tests passed

we'll see that ``example-1.0.tar.gz`` is contained in both
indices.  Let's remove the ``testuser/dev`` ``example`` release::

    $ devpi remove -y example
    About to remove the following releases and distributions
    version: 1.0
      - http://localhost:3141/testuser/dev/+f/4b9/381a0cc2c090c/example-1.0.tar.gz
      - http://localhost:3141/testuser/dev/+f/4b9/381a0cc2c090c/example-1.0.tar.gz.toxresult0
    Are you sure (yes/no)? yes (autoset from -y option)
    deleting release 1.0 of example

If you don't specify the ``-y`` option you will be asked to confirm
the delete operation interactively.

The ``example-1.0`` release remains accessible through ``testuser/dev``
because it inherits all releases from its ``testuser/staging`` base::

    $ devpi list example
    http://localhost:3141/testuser/staging/+f/4b9/381a0cc2c090c/example-1.0.tar.gz
    devpi-testing linux   python     3.5.1+ tests passed

::

    $ devpi-server --stop
    2016-12-14 11:15:43,140 INFO  NOCTX Loading node info from /tmp/home/.devpi/server/.nodeinfo
    2016-12-14 11:15:43,142 INFO  NOCTX wrote nodeinfo to: /tmp/home/.devpi/server/.nodeinfo
    killed server pid=23968

running devpi-server permanently
+++++++++++++++++++++++++++++++++

If you want to configure a permanent devpi-server install,
you can go to :ref:`quickstart-server` to get some help.


