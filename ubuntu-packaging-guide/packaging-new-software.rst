======================
Packaging New Software
======================

While there are thousands of packages in the Ubuntu archive, there are still
a lot nobody has gotten to yet. If there is an exciting new piece of software
that you feel needs wider exposure, maybe you want to try your hand at
creating a package for Ubuntu or a PPA_. This guide will take you through the
steps of packaging new software.

You will want to read the :doc:`Getting Set Up<./getting-set-up>` article first
in order to prepare your development environment.

Checking the Program
--------------------

The first stage in packaging is to get the released tar from upstream (we call
the authors of applications "upstream") and check that it compiles and runs.

This guide will take you through packaging a simple application called GNU Hello
which has been posted on GNU.org_.

Download GNU Hello::

    $ wget -O hello-2.10.tar.gz "http://ftp.gnu.org/gnu/hello/hello-2.10.tar.gz"

Now uncompress it::

    $ tar xf hello-2.10.tar.gz
    $ cd hello-2.10

This application uses the autoconf build system so we want to run ``./configure``
to prepare for compilation.

This will check for the required build dependencies. As ``hello`` is a simple
example, ``build-essential`` and ``texinfo`` should provide everything we need.
For more
complex programs, the command will fail if you do not have the needed libraries
and development files. Install the needed packages and repeat until the command
runs successfully.::

    $ ./configure

Now you can compile the source::

    $ make

If compilation completes successfully you can install and run the program::

    $ sudo make install
    $ hello

Starting a Package
------------------

``bzr-builddeb`` includes a plugin to create a new package from a template. The
plugin is a wrapper around the ``dh_make`` command.  Run the command providing
the package name, version number, and path to the upstream tarball::

    $ sudo apt-get install dh-make bzr-builddeb
    $ cd ..
    $ bzr dh-make hello 2.10 hello-2.10.tar.gz

Note: unfortunately the ``bzr dh-make`` subcommand is no longer available in
releases later than Ubuntu 20.04; so, you might need a container or virtual
machine (see `LXD <LXD_>`_) running Ubuntu 20.04 for this particular step.

When it asks what type of package type ``s`` for single binary. This will import
the code into a branch and add the ``debian/`` packaging directory.  Have a look
at the contents.  Most of the files it adds are only needed for specialist
packages (such as Emacs modules) so you can start by removing the optional
example files::

    $ cd hello/debian
    $ rm *ex *EX

You should now customise each of the files.

In ``debian/changelog`` change the
version number to an Ubuntu version: ``2.10-0ubuntu1`` (upstream version 2.10,
Debian version 0, Ubuntu version 1).  Also change ``unstable`` to the current
development Ubuntu release such as ``trusty``.

Much of the package building work is done by a series of scripts
called ``debhelper``.  The exact behaviour of ``debhelper`` changes
with new major versions, the compat file instructs ``debhelper`` which
version to act as.  You will generally want to set this to the most
recent version which is ``9``.

``control`` contains all the metadata of the package.  The first paragraph
describes the source package. The second and following paragraphs describe
the binary packages to be built.  We will need to add the packages needed to
compile the application to ``Build-Depends:``. For ``hello``, make sure that it
includes at least::

    Build-Depends: debhelper (>= 9)

And append ``texinfo``, as noted earlier::

    Build-Depends: debhelper (>= 9), texinfo

You will also need to fill in a description of the program in the
``Description:`` field.

``copyright`` needs to be filled in to follow the licence of the upstream
source.  According to the hello/COPYING file this is GNU GPL 3 or later.

``docs`` contains any upstream documentation files you think should be included
in the final package.

``README.source`` and ``README.Debian`` are only needed if your package has any
non-standard features, we don't so you can delete them.

``source/format`` can be left as is, this describes the version format of the
source package and should be ``3.0 (quilt)``.

``rules`` is the most complex file.  This is a Makefile which compiles the
code and turns it into a binary package.  Fortunately most of the work is
automatically done these days by ``debhelper 7`` so the universal ``%``
Makefile target just runs the ``dh`` script which will run everything needed.

All of these file are explained in more detail in the :doc:`overview of the
debian directory<./debian-dir-overview>` article.

Finally commit the code to your packaging branch (make sure to ``bzr add``
any untracked files that changed; for example, ``debian/source/format``)::

    $ bzr add debian/source/format
    $ bzr commit -m "Initial commit of Debian packaging."

Building the package
--------------------

Now we need to check that our packaging successfully compiles the package and
builds the .deb binary package::

    $ bzr builddeb -- -us -uc
    $ cd ../../

``bzr builddeb`` is a command to build the package in its current location.
The ``-us -uc`` tell it there is no need to GPG sign the package.  The result
will be placed in ``..``.

Note: if it fails with ``You must run ./configure before running 'make'.``,
add this to ``debian/rules`` (make sure to ``bzr add/commit`` it) and retry
``bzr builddeb``::

    override_dh_auto_clean:
            [ -f Makefile ] || ./configure
            dh_auto_clean

You can view the contents of the package with::

    $ lesspipe hello_2.10-0ubuntu1_amd64.deb

Install the package and check it works (later you will be able to uninstall it
using ``sudo apt-get remove hello`` if you want)::

    $ sudo dpkg --install hello_2.10-0ubuntu1_amd64.deb

You can also install all packages at once using::

    $ sudo debi

Next Steps
----------

Even if it builds the .deb binary package, your packaging may have
bugs.  Many errors can be automatically detected by our tool
``lintian`` which can be run on the source .dsc metadata file, .deb
binary packages or .changes file::

    $ lintian hello_2.10-0ubuntu1.dsc
    $ lintian hello_2.10-0ubuntu1_amd64.deb

To see verbose description of the problems use ``--info`` lintian flag
or ``lintian-info`` command.

For Python packages, there is also a ``lintian4python`` tool that provides
some additional lintian checks.

After making a fix to the packaging you can rebuild using ``-nc`` "no clean"
without having to build from scratch::

    $ bzr builddeb -- -nc -us -uc

Having checked that the package builds locally you should ensure it builds on a
clean system using ``pbuilder``. Since we are going to upload to a PPA
(Personal Package Archive) shortly, this upload will need to be *signed* to
allow Launchpad to verify that the upload comes from you (you can tell the
upload will be signed because the ``-us`` and ``-uc`` flags are not passed to
``bzr builddeb`` like they were before). For signing to work you need to have
set up GPG. If you haven't set up ``pbuilder-dist`` or GPG yet, :doc:`do so
now<./getting-set-up>`::

    $ bzr builddeb -S
    $ cd ../build-area
    $ pbuilder-dist trusty build hello_2.10-0ubuntu1.dsc

When you are happy with your package you will want others to review it.  You
can upload the branch to Launchpad for review::

    $ bzr push lp:~<lp-username>/+junk/hello-package

Uploading it to a PPA will ensure it builds and give an easy way for you and
others to test the binary packages.  You will need to set up a PPA in Launchpad
and then upload with ``dput``::

    $ dput ppa:<lp-username>/<ppa-name> hello_2.10-0ubuntu1.changes

You can ask for reviews in ``#ubuntu-motu`` IRC channel, or on the
`MOTU mailing list <ubuntu-motu_>`_.  There might also be a more specific
team you could ask such as the GNU team for more specific questions.

Submitting for inclusion
------------------------

There are a number of paths that a package can take to enter Ubuntu.
In most cases, going through Debian first can be the best path. This
way ensures that your package will reach the largest number of users
as it will be available in not just Debian and Ubuntu but all of their
derivatives as well. Here are some useful links for submitting new
packages to Debian:

  - `Debian Mentors FAQ <MentorsFAQ_>`_ - debian-mentors is for the mentoring of new and
    prospective Debian Developers. It is where you can find a sponsor
    to upload your package to the archive.

  - `Work-Needing and Prospective Packages <WNPP_>`_ - Information on how to file
    "Intent to Package" and "Request for Package" bugs as well as list
    of open ITPs and RFPs.

  - `Debian Developer's Reference, 5.1. New packages <DevRef_>`_ - The entire
    document is invaluable for both Ubuntu and Debian packagers. This
    section documents processes for submitting new packages.

In some cases, it might make sense to go directly into Ubuntu first. For
instance, Debian might be in a freeze making it unlikely that your
package will make it into Ubuntu in time for the next release. This
process is documented on the `"New Packages" <NewPackages_>`_ section of the Ubuntu wiki.

Screenshots
-----------

Once you have uploaded a package to debian, you should add screenshots
to allow propective users to see what the program is like. These should
be uploaded to http://screenshots.debian.net/upload .

.. _PPA: https://help.launchpad.net/Packaging/PPA
.. _GNU.org: http://www.gnu.org/software/hello/
.. _`packages.ubuntu.com`:  http://packages.ubuntu.com/
.. _ubuntu-motu: https://lists.ubuntu.com/mailman/listinfo/ubuntu-motu
.. _MentorsFAQ: https://wiki.debian.org/DebianMentorsFaq
.. _WNPP: http://www.debian.org/devel/wnpp/
.. _DevRef: http://www.debian.org/doc/manuals/developers-reference/pkgs.html#newpackage
.. _NewPackages: https://wiki.ubuntu.com/UbuntuDevelopment/NewPackages
.. _LXD: https://linuxcontainers.org/lxd/
