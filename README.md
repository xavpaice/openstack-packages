# Creating OpenStack debs for everyday operators

Xav Paice, Jan 2015

## Abstract

For a large number of sysadmins, the thought of taking a project consisting of
a repo on GitHub and making packages for running in production isn’t realistic
at all.  Installing packages from Pip is sometimes frowned upon, PPAs are
viewed with suspicion, and we are left to choose from the packages maintained
upstream in the distribution of choice.  The limitation we face in real life is
that the packages from distributions are often at least a few weeks behind the
upstream code, and since the OpenStack ecosystem is a fast moving and dynamic
field which many people contribute to, there’s a significant chance we’ll want
to add a patch to code that might not even be merged upstream, let alone
included in a stable release ready for a distro to package it up. This article
describes how the folks at Catalyst IT in New Zealand go about taking the
distro packages, and combining that with custom code built into packages
ready for deployment. We focus on Ubuntu packages because that's what Catalyst
IT uses, but also because there's already a number of good quality tools
focussed on rpm packages.

## Motivation for creating custom packages

When we first started deploying OpenStack, we selected Ubuntu (Precise) as the
platform because it appeared to be the platform of choice for a large number of
OpenStack devs. We don’t regret that choice, so far it’s been an excellent
platform. The ready-made repository of current OpenStack packages for that
(LTS) platform is the Ubuntu Cloud Archive (UCA) - a place where Canonical
produce packages from stable branches of OpenStack code.  We can be reasonably
well assured that the packages released have undergone some level of testing
and quality control, and installing from these should be a pretty safe bet.
There also exists an excellent set of Debian packages that are compatible with
Ubuntu, plus there are OpenStack packages in the base Ubuntu repos.

Up till we first needed to get some customisations, the UCA was a pretty good
source of packages. Sometimes, updates were a bit slow to come to light, but we
could wait a few weeks, and urgent security issues seem to turn up within a
reasonable time-frame. But we then added some patches to Nova, to support RBD
block devices, and to Ceilometer for some custom pollsters. The RBD patches
weren't merged upstream in that (older) version of OpenStack, the Ceilometer
pollsters weren't needed upstream, then we added some custom Horizon code for
our environment.  All in, we found ourselves either having to install existing
packages then use a configuration management tool to monkey-wrench some of the
files delivered by packages, or make our own packages.

## Limitations

The method below has been tested only in the Catalyst IT environment - we use
Ubuntu (currently Precise for OpenStack, and Trusty for all our workstations
and dev environments). Currently we use Icehouse.

Working with a hybrid of packages from our repo, plus some pre-requisite packages
from the Ubuntu distro (or Ubuntu Cloud Archive) brought about it’s own set of
issues. We may have published a package for Nova, with version 2014.1.2, then
2014.1.3 is released in Ubuntu Cloud Archive, excluding our set of ’extra’
patches. If we’re not on top of things, we wind up upgrading and losing our
patches. This was something we could have solved in a number of ways, but in
the end we combined both prepending the version with 99:\<thing\> making the
package manager always select our higher priority version above the others
within the series, and also ensuring that we're regularly refreshing packages.
Additionally, when we started making our packages, we noticed that pbr was
generating version numbers based on the last commit id - meaning there's a good
chance that an older package will have a higher version than a fresh one.  This
was easily solved by using a datestamp instead.

## Method

Our first goal was to get the minor customizations we made to the OpenStack
code into production with a minimum of effort and risk.  There's no need to
re-invent the wheel when the work to build packages has already been done, so
we opted to make use of the existing resources and merely add the changes from
our fork of the source.

First off, we needed to make a fork of the OpenStack code - e.g. for Nova, we
cloned the git repo to our own Gerrit server, so we could manage the changes to
sources with revision control. Although we created the projects in Gerrit, you
could easily substitute any kind of git location here.

The following example is to clone Nova:

```shell
git clone git@github.com:openstack/nova.git openstack-nova
cd openstack-nova
git remote add gerrit ssh://<user>@gerrit.example.com:29418/openstack-nova
git review -s
git push gerrit origin/master:refs/heads/master
git push gerrit origin/stable/icehouse:refs/heads/stable/icehouse
```

We now have a copy of the upstream OpenStack Nova code on our local repo.  Next
up, we make the changes we wanted to make - either cherrypick some commits,
make some new commits, or whatever.  Ensure the tests run OK, merge the code.

The resulting repo is enough for Python packages with the extra code - but
that's not a .deb which we would be ready to install in production.  The
missing part is already provided in the UCA packaging repo, and we can grab a
copy of that like this:

```
git clone bzr::lp:~ubuntu-server-dev/nova/icehouse
```

At this stage, we opted to make the packaging tools into a separate repo, so we
can keep things tidy.  In this case, we have a repo called
openstack-nova-packaging which contains the content of the checkout above -
which we added to Gerrit just like we did for openstack-nova.  If we're willing
to maintain our own copy of that packaging code, we're able to make changes so
we can add in our own files, or add/remove quilt patches as appropriate.  We
did find that some of our code added files which weren't included in the
original packages - e.g. an additional dashboard in Horizon adds files outside
of the original directories list and required either edits to the
openstack-dashboard.dirs file, or creation of a fresh package within that set.

I should note that the debian directory obtained from the UCA
has a number of Ubuntu specific patches in the debian/patches/ directory.
These make use of a tool called quilt that manages patches to code before it's
packaged - but not integrated into the upstream code itself.  That would suit
our needs just fine, but these patches applied by quilt aren't included in the
code that our developers are using - they're specific to the Ubuntu packages -
and so tests that might pass in dev might not be quite so clean in the packaged
code.  Likewise, if our devs write a patch, they want that code in their source
tree rather than have to build/install packages to see the result of the patch.
For that reason, we opted not to make use of that mechanism and to try to keep
the code our devs use as consistent with what's running in production as
possible.

We're now equipped with two code repositories under our control, and so are
ready to turn that into some debs ready to be put in our private repo that was
made earlier.

The next issue we came across is that the packages we built locally on our
Trusty workstations didn't install cleanly on Precise.  This should come as no
surprise.  It was also necessary to install a bunch of build dependency
packages on the build environment that we may or may not want on a workstation.
To overcome these pitfalls, we used Vagrant to quickly spin up an environment
for building packages.  Our development environment for testing and dev uses
Vagrant so we already had some work we could re-use there, and our devs use
vagrant-lxc so VMs are quite fast to create.

The steps to making packages therefore consist of the following:
* make a build environment VM
* clone the code into a directory accessible by the VM
* tidy up the quilt packages (there's usually some fuzz in applying the patches)
* create the code tarball
* set the version
* do the actual package build

I'll go through each step.  Note that we have automated all this in some very
rudimentary shell scripts rather than run each step by hand, but some
things need a little hand-holding and it's helpful to know what's actually
happening.

### Making a space in which to work

There are a few prerequisites for this to work, we found the following to be
reasonable:

* Ubuntu Trusty for a base OS, with current packages
* packages: git, vagrant, lxc
* vagrant plugins: vagrant-lxc
* A current lxc compatible Precise box for vagrant

```shell
$ sudo apt-get install git vagrant lxc
$ vagrant plugin install vagrant-lxc
$ vagrant box add precise64 http://<your template box location>
```

Let's assume we start out in ~/git - a generic working directory.

To make this easy, we have a packaging repo and a code repo for the particular
project we're working (Nova in this example) stored on a Gerrit server
gerrit.example.com.  The packaging repo contains the following files:
* a copy of the files in the packaging repo upstream in bzr::lp:~ubuntu-server-dev/nova/icehouse
* puppet manifests and modules to build the packaging VM (see appendix)
* a Vagrantfile (see appendix)
* some driver scripts to run the commands listed below

### Creating a build VM

cd to the root of the packaging repo directory, and run the following:

```shell
vagrant up
```

This creates a VM which includes the packages needed to actually build a bunch
of debs - the mechanics of Vagrant and Puppet are out of scope for this doc,
but for convenience the appropriate files included below in an appendix.

### Building the actual packages

You'll need a copy of the latest code from our repo, inside the packaging
directory tree. This means we can do the various edits and build steps without
polluting our working code repo.

```shell
git clone https://gerrit.example.com/openstack-nova
cd openstack-nova
git checkout origin/stable/icehouse
```

Now we can switch to the vagrant VM to complete the actual build.

```shell
vagrant ssh
```

Set up some basic environment variables - I like to export the following:
```shell
PROJECT='nova'
UBUNTUVER="0custom1"
DEBEMAIL='My Name <myemail@example.com>'
```

Now we can cd to the appropriate directory and make the source tarball:

```shell
cd /usr/src/${PROJECT}/openstack-${PROJECT}
MAJORVER=$(awk -F "=" '$1 ~ /^ *version *$/ {print $2}' setup.cfg |sed 's/ //')
SOURCEDIR=$(python setup.py sdist | grep -Po 'removing \W\K[^"][\w\.-]*')
TARBALL="${PROJECT}_${MAJORVER}-${UBUNTUVER}.orig.tar.gz"
echo "moving dist/${SOURCEDIR}.tar.gz to $TARBALL"
mv dist/${SOURCEDIR}.tar.gz ../$TARBALL
cd ..
tar -xf $TARBALL
```

Above we extracted the major version number from the setup.cfg in the source
tree, we ran setup.py to create the tarball itself, and from the output of that
command we crudely extracted the resulting location of the tarball.  Then we
moved it to a name and location that the package building tools understand.

Next, we copy the debian directory from our packaging repo into the source tree:

```shell
cp -r icehouse/debian $SOURCEDIR/debian
```

To get the packages created with a vaguely useful version number, we increment
the version in the changelog - note here the prefix of 99: which sets the
preference, and the suffix of the version including a datestamp:

```shell
cd /usr/src/${PROJECT}/$SOURCEDIR
VERSION="99:${MAJORVER}-${UBUNTUVER}-`date +%Y%m%d`"
dch -D stable -v $VERSION "Build from custom git repo"  --force-distribution
```

Inside the debian/patches directory, there's usually some patches.  These must
apply cleanly in order for the packages to build - any amount of 'fuzz' will
cause the build to fail.  Luckily, quilt is very elegant in how it deals with
these things, and a minimal fuzz can easily be fixed up like this:

Create a ~/.quiltrc
```shell
QUILT_PATCHES=debian/patches
QUILT_NO_DIFF_INDEX=1
QUILT_NO_DIFF_TIMESTAMPS=1
QUILT_REFRESH_ARGS="-p ab"
```

Check the patches apply cleanly, and tidy up any fuzz, then remove the patches
again since the package build needs a clean source:
```shell
for i in `quilt series` ; do
  echo $i
  quilt push
  quilt refresh
done
quilt pop -a
```

Finally, we build and install the build-deps, and then actually build the packages:
```shell
sudo mk-build-deps -ir debian/control
debuild -us -uc
```

All things being well, you wind up with some packages at the root of your
directory tree which can be installed in a test environment and put through
their paces. We tend to pop them into a private repo, with either 'stable' or
'testing' distribution, so we can test without upgrading in production till
we're ready.

Appendix: Puppet manifests
--------------------------

To use the Puppet manifest below, you’ll also need to have the apt and stdlib
Puppet modules installed likely under ./modules/...

./manifests/build.pp
```pp
# puppet manifest to make a build server

$packages = [ 'git', 'git-buildpackage', 'build-essential',
              'openstack-pkg-tools', 'python-setuptools',
              'equivs', 'quilt', 'libmysqlclient-dev', 'libpq-dev',
              'libxml2-dev', 'libxslt-dev', 'libffi-dev' ]

package { $packages:
  ensure => present,
}

include apt

apt::source { 'cloudarchive':
  location          => 'http://ubuntu-cloud.archive.canonical.com/ubuntu',
  release           => 'precise-updates/icehouse',
  repos             => 'main',
  required_packages => 'ubuntu-cloud-keyring',
}

apt::source { 'cloudarchive-havana':
  ensure => absent,
}

Apt::Source['cloudarchive'] -> Package<||>
```

Appendix: Vagrantfile
---------------------

./Vagrantfile
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # Cachier is a handy module to save bandwidth
  if defined? VagrantPlugins::Cachier
      config.cache.enable :apt
  end

  # box being used
  config.vm.box = "precise64"

  # synced folders with the host (puppet config + puppet modules)
  config.vm.synced_folder "./", "/usr/src/horizon"

  # lxc specific configuration
  config.vm.provider :lxc do |lxc, override|
    lxc.customize "cgroup.memory.limit_in_bytes", "2048M"
  end

  config.vm.define 'build', primary: true do |node|
    node.vm.hostname = 'build.local'
    node.vm.provision :puppet do |puppet|
      puppet.manifests_path = "./manifests"
      puppet.module_path = "./modules"
      puppet.manifest_file = "build.pp"
    end
  end
end
```


Useful links
------------

https://wiki.debian.org/Python/Packaging

http://docs.openstack.org/developer/pbr/index.html

