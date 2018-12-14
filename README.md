# A Generic Buildpack for Cloud Foundry

[![Build Status](https://travis-ci.org/oetiker/sourcey-buildpack.svg)](https://travis-ci.org/oetiker/sourcey-buildpack)

Isn't it simply amazing to see these demos, where they throw a bunch of php,
ruby, Java or python code at a Cloud Foundry site and it gets magically
turned into a running web applications.  Alas for me, life is often a wee
bit more complicated than that.  My projects always seem to required a few
extra libraries or they are written in a language that doesn't have a specific
buildpack.

That's where `sourcey-buildpack` comes in. It allows you to easily compile
any libraries and binaries from source.  It takes care of putting everything
into the right spot so that the end result happily lives in
`/home/vcap/sourcey` and it even knows that it does, and therefore does not
require any `LD_LIBRARY_PATH` or other special magic to make it work.

The sourcey-buildpack expects to find three special files in your application directory:

`SourceyBuild.sh` (optional) to compile all the binaries you need.

`SourceyBuildApp.sh` (optional) to prepare the actual application if this needs any prepping.

`SourceyStart.sh` (mandatory) to launch the application at runtime.

## The `SourceyBuild.sh` script

In this script you build your third-party software. At the most basic level,
you just have to make sure to install the result into `$PREFIX`.

You may want to use `$WORK_DIR` to unpack your source. And if you need other
files from your application you can find them in `$BUILD_DIR`.

For a classic autotools packaged application, your setup instructions might
look like this:

```shell
cd $WORK_DIR
wget http://cool-site.com/tool.tar.gz
tar xf tool.tar.gz
cd tool
./configure --prefix=$PREFIX
make install
cd ..
rm -r tool
```

When your script has run through, Sourcey goes to work.

1. It moves the content of `$PREFIX` into `$STAGE_DIR/sourcey`, ready for packaging.

2. It creates a copy of `$STAGE_DIR/sourcey` in `$CACHE_DIR` and tags it
   with the md5 sum of your `SourceyBuild.sh`.  If you re-deploy the same
   app again, without changing `SourceyBuild.sh` the content of the
   `$CACHE_DIR` will be used instead of rebuilding everything.


The directory layout at build time:

```
/home
   /vcap (aka $HOME)
      /app
          /sourcey (aka $PREFIX)

/tmp
   /sourcey.$$ (aka $WORK_DIR)
   /cache (aka $CACHE_DIR)
   /stage
      /app (aka $BUILD_DIR)
          /sourcey
```

The `/home/vcap/app` directory is where your application will reside at runtime. By compiling
your binaries into that location, their world view will be intact at runtime as well.

The `/tmp/stage/app` directory gets packaged up and transfered to the run-time
environment.  At first only your application will be sitting there (in the
`/tmp/stage/app` directory).

The content of the `/tmp/cache` directory will be made available whenever you
push your application again.  Sourcey uses this location to cache compiled
binaries and restore them when you push an update without changing the
`SourceyBuild.sh` file.

### Helpers

To make life a bit simpler still, Sourcey provides a few helper functions:

#### `buildAuto <url> [options]`

Does essentially the same build process as described in the autotools
example above.  If you want to specify extra configure options, just add
them as extra arguments at the end of the function call:

```shell
buildAuto http://mysite/tool.tar.gz --default-answer=42 --switch-off
```

#### `buildPerl <version>`

Creates the Perl version of your choice.

Since most Cloud Foundry setups are on Ubuntu lucid (10.04) stacks, perl is
at version 5.10.1 which is about 100 years out of date.  This is was what got
me to write this in the first place.

```sh
buildPerl 5.20.2
```

#### `buildPerlModule [any cpanm option]`

This is a wrapper for `cpanm` which you can use to install extra perl
modules.  The new modules will get installed into your freshly installed
Perl.  If you have not installed your own perl with the `buildPerl` call the
system Perl will be used and the modules will go to
`/home/vcap/sourcey/lib/perl5`.  Sourcey will take care of setting the
`PERL5LIB` variable accordingly.

## The `SourceyBuildApp.sh` script ...

can do whatever you deem necessary to get your actual
application into shape for execution.  Nothing will be cached.  If you push
an update for your application, this script will run again.


## The `SourceyStart.sh` script ...

gets executed when your application should be started. Sourcey will
take care of setting the `$PATH` variable so that all these shiny new 3rd
party tools get found automatically.  If you have not setup your own copy of
Perl, the `$PERL5LIB` path will be set, so that the system Perl finds any
new modules you may have compiled.

At the end of your `SourceyStart.sh` someone should be listening port
`$PORT` for incoming web requests.

In order for your application to integrate with the Cloud Foundry
infrastructure, you want to JSON decode the content of the environment
variables `$VCAP_SERVICES` and `$VCAP_APPLICATION`.

The directory layout at runtime.

```
/home
   /vcap
      /app  (aka $HOME !!!)
         /sourcey
```

## Debugging

If things are not going according to plan. You can put the following variables
into your `SourceyBuild.sh` file.

`SOURCEY_VERBOSE=1` will cause all output generated at build time to be sent
to STDOUT.  Note that this does look like an environment variable, but
the compile script actually runs `grep` on your `SourceyBuild.sh` to detect it.

`SOURCEY_REBUILD=1` will ignore any cached copy of your binaries and rebuild the lot
every time you push a new version.

## Example

The code in the example directory demonstrates how to setup a simple
some simple apps.

The following instructions assume you have already setup a Cloud Foundry
account and you have logged yourself in with `cf login`. Make sure to modify
the `manifest.yml` file in the example directory to suit your needs.

```sh
cd example/EXAMPLE-OF-YOUR-CHOICE
cf push
```
