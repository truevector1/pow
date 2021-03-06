Pow User's Manual
=================

**Pow is a zero-configuration Rack server for Mac OS X.** It makes
developing Rails and Rack applications as frictionless as
possible. You can install it in ten seconds and have your first app up
and running in under a minute. No mucking around with `/etc/hosts`, no
compiling Apache modules, no editing configuration files or installing
preference panes. And running multiple apps with multiple versions of
Ruby is trivial.

How does it work? A few simple conventions eliminate the need for
tedious configuration. Pow runs as your user on an unprivileged port,
and includes both an HTTP and a DNS server. The installation process
sets up a firewall rule to forward incoming requests on port 80 to
Pow. It also sets up a system hook so that all DNS queries for a
special top-level domain (`.dev`) resolve to your local machine.

To serve a Rack app, just symlink it into your `~/.pow`
directory. Let's say you're working on an app that lives in
`~/Projects/myapp`. You'd like to access it at
`http://myapp.dev/`. Setting it up is as easy as:

    $ cd ~/.pow
    $ ln -s ~/Projects/myapp

That's it! The name of the symlink (`myapp`) determines the hostname
you use (`myapp.dev`) to access the application it points to
(`~/Projects/myapp`).

-----

## Installation ##

Pow requires Mac OS X version 10.6 or newer. To install or upgrade
Pow, just open a terminal and run this command:

    $ curl get.pow.cx | sh

You can [review the install script](http://get.pow.cx/) yourself
before running it, if you'd like. Always a good idea.

The installer unpacks the latest Pow version into
`~/Library/Application Support/Pow/Versions` and points the
`~/Library/Application Support/Pow/Current` symlink there. It also
installs
[launchd](http://developer.apple.com/library/mac/#documentation/Darwin/Reference/ManPages/man8/launchd.8.html)
scripts for your user (the Pow server itself) and for the system (to
set up the `ipfw` rule), if necessary. Then it boots the server.

**Note**: The firewall rule installed by Pow redirects all incoming
  traffic on port 80 to port 20559, where Pow runs. This means if you
  have another web server running on port 80, like the Apache that
  comes with Mac OS X, it will be inaccessible without either
  disabling the firewall rule or updating that server's configuration
  to listen on another port.

### Installing From Source ###

To install Pow from source, you'll need Node 0.6.0 or higher and npm:

    $ git clone https://github.com/37signals/pow.git
    $ cd pow
    $ npm --global install
    $ npm --global run-script pow restart

For detailed instructions on installing Pow from source, including
instructions on how to install Node and npm, see the
[Installation](https://github.com/37signals/pow/wiki/Installation)
wiki page.

### Uninstalling Pow ###

If you decide Pow's not for you, uninstallation is just as easy:

    $ curl get.pow.cx/uninstall.sh | sh

([Review the uninstall script](http://get.pow.cx/uninstall.sh).)

## Managing Applications ##

Pow deals primarily with Rack applications. For the purposes of this
document, a _Rack application_ is a directory with a `config.ru`
rackup file (and optionally a `public` subdirectory containing static
assets). For more information on rackup files, see the [Rack::Builder
documentation](http://rack.rubyforge.org/doc/Rack/Builder.html).

Pow automatically spawns a worker process for an application the first
time it's accessed, and will keep up to two workers running for each
application. Workers are automatically terminated after 15 minutes of
inactivity.

### Using Virtual Hosts and the .dev Domain ###

A _virtual host_ specifies a mapping between a hostname and an
application. To install a virtual host, symlink a Rack application
into your `~/.pow` directory. The name of the symlink tells Pow which
hostname you want to use to access the application. For example, a
symlink named `myapp` will be accessible at `http://myapp.dev/`.

**Note**: The Pow installer creates `~/.pow` as a convenient symlink
  to `~/Library/Application Support/Pow/Hosts`, the actual location
  from which virtual host symlinks are read.

#### Subdomains ####

Once a virtual host is installed, it's also automatically accessible
from all subdomains of the named host. For example, the `myapp`
virtual host described above could also be accessed at
`http://www.myapp.dev/` and `http://assets.www.myapp.dev/`. You can
override this behavior to, say, point `www.myapp.dev` to a different
application &mdash; just create another virtual host symlink named
`www.myapp` for the application you want.

#### Multiple Virtual Hosts ####

You might want to serve the same application from multiple hostnames.
In Pow, an application may have more than one virtual host. Multiple
symlinks that point to the same application will share the same worker
processes.

#### The Default Virtual Host ####

If you attempt to access a domain that Pow doesn't understand, like
`http://localhost/`, you'll see a page letting you know that Pow is
installed and working correctly, with instructions on how to set up an
application.

You can override this behavior to serve all requests for unhandled
domains with a particular Rack application. Create a symlink in
`~/.pow` named `default` and point it to the application of your
choice.

#### Port Proxying ####

Pow's port proxying feature lets you route all web traffic on a
particular hostname to another port on your computer. To use it, just
create a file in `~/.pow` (instead of a symlink) with the destination
port number as its contents.

For example, to forward all traffic for `http://proxiedapp.dev/` to
port 8080:

    $ echo 8080 > ~/.pow/proxiedapp

You can also use port proxying to access web apps written for other
runtimes such as Python or Node.js. Remember that services behind the
proxy won't automatically be started or stopped like Rack apps.

#### Accessing Virtual Hosts from Other Computers ####

Sometimes you need to access your Pow virtual hosts from another
computer on your local network &mdash; for example, when testing your
application on a mobile device or from a Windows or Linux VM.

The `.dev` domain will only work on your development
computer. However, you can use the special [`.xip.io`
domain](http://xip.io/) to remotely access your Pow virtual hosts.

Construct your xip.io domain by appending your application's name to
your LAN IP address followed by `.xip.io`. For example, if your
development computer's LAN IP address is `10.0.1.43`, you can visit
`myapp.dev` from another computer on your local network using the URL
`http://myapp.10.0.1.43.xip.io/`.

### Customizing Environment Variables ###

Pow lets you customize the environment in which worker processes
run. Before an application boots, Pow attempts to execute two scripts
&mdash; first `.powrc`, then `.powenv` &mdash; in the application's
root. Any environment variables exported from these scripts are passed
along to Rack.

For example, if you wanted to adjust the Ruby load path for a
particular application, you could modify `RUBYLIB` in `.powrc`:

    export RUBYLIB="app:lib:$RUBYLIB"

#### Choosing the Right Environment Script ####

Pow supports two separate environment scripts with the intention that
one may be checked into your source control repository, leaving the
other free for any local overrides. If this sounds like something you
need, you'll want to keep `.powrc` under version control, since it's
loaded first.

### Working With Different Versions of Ruby ###

Pow invokes each application's Ruby processes in an isolated
environment. This design makes it possible to use different Ruby
runtimes on a per-application basis.

There are three ways to specify which version of Ruby to use for a
particular application.

#### Specifying Ruby Versions with rbenv ####

You can use [rbenv](https://github.com/sstephenson/rbenv) to specify
per-application Ruby versions for use with Pow.

The `rbenv local` command lets you set a per-project Ruby version by
saving an `.rbenv-version` file in the current directory. For example,
to configure a particular application to run with Ruby 1.9.3-p194,
change to the application's directory and run:

    $ rbenv local 1.9.3-p194

Assuming you have set up rbenv in your login environment, there is no
additional configuration necessary to use it with Pow.

For more information, see the [rbenv
documentation](https://github.com/sstephenson/rbenv#readme).

#### Specifying Ruby Versions with RVM ####

[RVM](http://rvm.io/) is another option for specifying per-application
Ruby versions for use with Pow.

You can create a [project `.rvmrc`
file](https://rvm.io/workflow/rvmrc#project) to specify an
application's Ruby version. For example, to configure your application
to run with Ruby 1.8.7, add the following to `.rvmrc` in the
application's root directory:

    rvm 1.8.7

Because RVM works by injecting itself into your shell, you must first
load it in each application's `.powrc` or `.powenv` file using the
following code:

    if [ -f "$rvm_path/scripts/rvm" ] && [ -f ".rvmrc" ]; then
      source "$rvm_path/scripts/rvm"
      source ".rvmrc"
    fi

For more information, see the [RVM web site](http://rvm.io/).

#### Specifying Ruby Versions by Setting $PATH ####

You can adjust the `PATH` environment variable in `.powrc` or
`.powenv` to select Ruby versions on a per-application basis. For
example, if you have Ruby installed into `/opt/ruby-1.8.7`, you can
add the following to `.powenv`:

    export PATH="/opt/ruby-1.8.7/bin:$PATH"

When Pow loads your application, it will find and use the first `ruby`
binary in your `PATH` &mdash; in this case `/opt/ruby-1.8.7/bin/ruby`.

### Serving Static Files ###

Pow automatically serves static files in the `public` directory of your
application. It's possible to serve a completely static site without a
`config.ru` file as long as it has a `public` directory. If you have a static
site and want to keep your files in the root of your project (i.e. not in a
`public` directory), you can do the following:

    $ cd ~/.pow
    $ mkdir your-app-domain
    $ cd !$
    $ ln -s ~/Projects/your-app public

### Restarting Applications ###

You can tell Pow to restart an application the next time it's
accessed. Simply save a file named `restart.txt` in the `tmp`
directory of your application (you'll need to create the directory
first if it doesn't exist). The easiest way to do this is with the
`touch` command:

    $ touch tmp/restart.txt

Restarting an application will also reload any environment scripts
(`.powrc`, `.powenv`, or `.rvmrc`) before booting the app, so don't
forget to touch `restart.txt` if you make changes to these scripts.

It's also fine to kill worker processes manually &mdash; they'll
restart the next time you access the virtual host. A handy way to do
this is with OS X's Activity Monitor. Select "All Processes,
Hierarchically" from the dropdown at the top of the Activity Monitor
window. Then find the `pow` process, expand the disclosure triangle,
find the Ruby worker process you want to kill, and choose "Quit
Process." (You can click "Inspect" on a worker process and choose
"Open Files and Ports" to determine which application the process is
serving.)

#### Restarting Before Every Request ####

It can be useful during development to reload your application with
each request, and frameworks like Rails will handle such reloading for
you. For pure Rack apps, or when using frameworks like Sinatra that
don't manage code reloading, Pow can help.

If the `tmp/always_restart.txt` file is present in your application's
root, Pow will automatically reload the application before each request.

**Note**: `tmp/always_restart.txt` will only reload the application,
   _not_ its environment scripts. To reload `.powrc`, `.powenv`, or
   `.rvmrc`, you must touch `tmp/restart.txt` first.

### Viewing Log Files ###

Pow stores log files in the `~/Library/Logs/Pow` directory so they can
be viewed easily with OS X's Console application. Each incoming
request URL is logged, along with its hostname and HTTP method, in the
`access.log` file. The stdout and stderr streams for each worker
process are captured and logged to the `apps` directory in a file
matching the name of the application.

**Note**: Rails logger output does not appear in Pow's logs. You'll
  want to `tail -f log/development.log` to see those entries.

## Configuring Pow ##

Pow is designed so that most people will never need to configure
it. Sometimes, though, you can't avoid having to adjust a setting or
two.

When Pow boots, it executes the `.powconfig` script in your
home directory if it's present. You can use this script to export
environment variables that will override Pow's default settings.

For example, this `~/.powconfig` file tells Pow to kill idle
applications after 5 minutes (300 seconds) and spawn 3 workers per
app:

    export POW_TIMEOUT=300
    export POW_WORKERS=3

See the [Configuration class
documentation](http://pow.cx/docs/configuration.html#section-5) for a
full list of settings that you can change.

**Note**: After modifying a setting in `~/.powconfig`, you'll need to
  restart Pow for the change to take effect. See the Restarting Pow
  section below.


### Configuring Top-Level Domains ###

The `POW_DOMAINS` environment variable specifies a comma-separated
list of top-level domains for which Pow will serve DNS queries and
HTTP requests. The default value for this list is a single domain,
`dev`, meaning Pow will configure your system to resolve `*.dev` to
127.0.0.1 and serve apps in `~/.pow` under the `.dev` domain.

You can add additional domains to `POW_DOMAINS`:

    export POW_DOMAINS=dev,test

If you want Pow to serve apps under additional top-level domains, but
not serve DNS queries for those domains, use the `POW_EXT_DOMAINS`
variable. Entries in `POW_EXT_DOMAINS` will not be configured with the
system resolver, so you must make sure they point to your computer by
other means.

**Note**: If you change the value of `POW_DOMAINS`, you must reinstall
  Pow using `curl get.pow.cx | sh`. This is because the relevant files in
  `/etc/resolver/` are created at install time.

**WARNING**: Adding top-level domains like ".com" to `POW_DOMAINS` can
  be hazardous to your health! In the (likely) event that at some
  point you lock yourself out of these domains, you will be unable to
  reach important remote addresses like github.com (where you can find
  the source code) and S3 (where Pow's installation and uninstallation
  scripts are hosted). Do not panic! Delete the files Pow has created
  in `/etc/resolver/` and DNS activity will return to normal. (You can
  safely use `POW_EXT_DOMAINS` for these domains instead.)

### Reading the Current Configuration ###

If you are writing software that interfaces with Pow, you can inspect
the running server's status and configuration via HTTP. To access this
information, open a connection to `localhost` and issue a `GET`
request with the header `Host: pow`. The available endpoints are:

* `/status.json`: A JSON-encoded object containing the current Pow
  server's process ID, version string, and the number of requests it's
  handled since it started.
* `/env.json`: A JSON-encoded object representing the running server's
  environment, which is inherited by each application worker.
* `/config.json`: A JSON-encoded object representing the running
  server's [configuration](http://pow.cx/docs/configuration.html).

Example of requesting an endpoint with `curl`:

    $ curl -H host:pow localhost/status.json

Alternatively, if you know the path to the Pow binary, you can
generate an `eval`-safe version of the local configuration by invoking
Pow with the `--print-config` option (useful for shell scripts):

    $ eval $(~/Library/Application\ Support/Pow/Current/bin/pow \
        --print-config)
    $ echo $POW_TIMEOUT
    900

### Restarting Pow ###

Pow runs as a Mac OS X Launch Agent. If the Pow server process
terminates, the OS will restart it automatically.

You may need to manually restart Pow if you make configuration changes
to `~/.powconfig` or modify your login environment. To tell Pow to
quit and restart, touch the global `restart.txt` file:

    $ touch ~/.pow/restart.txt

Alternatively, you can use the Activity Monitor application. Find the
`pow` process in the process listing, select it, and click the Quit
Process button.

## Third-Party Tools ##

* [Powder](https://github.com/Rodreegez/powder) is "syntactic sugar
  for Pow." Install the gem (`gem install powder`) and you'll get a
  `powder` command-line utility that makes it easier to add
  applications, tail your log files, and restart Pow. See the [Powder
  readme](https://github.com/Rodreegez/powder#readme) for more
  examples of what it can do.

* [Powify](https://github.com/sethvargo/powify) is "an easy-to-use
  wrapper for 37signals' Pow." Install the gem (`gem install powify`)
  to get a `powify` command for installing, updating, and managing Pow
  and your virtual hosts. See the [Powify
  readme](https://github.com/sethvargo/powify#readme) for the full
  list of commands.

* [Forward](https://forwardhq.com/) lets you "share localhost
  over the web." It creates a tunnel between your computer and the
  Forward server, then gives you a publicly accessible URL so others
  can see the app or site you're working on. Install the gem via (`gem install forward`) and
  then run `forward yourapp.dev`. Your Pow application will be
  accessible at `https://youraccount.fwd.wf/`.

## Contributing ##

Pow is written in [Node.js](http://nodejs.org/) with
[CoffeeScript](http://jashkenas.github.com/coffee-script/). You can
read the [annotated source code](http://pow.cx/docs/) to learn about
how it works internally. Please report bugs on the [GitHub issue
tracker](https://github.com/37signals/pow/issues).

If you're interested in contributing to Pow, first start by
[installing Pow from
source](https://github.com/37signals/pow/wiki/Installation).

Make your changes and use `cake` to run the test suite:

    $ cake test

Then submit a pull request on GitHub. Your patch is more likely to be
merged if it's well-documented and well-tested. Read through the
[closed issues](https://github.com/37signals/pow/issues?state=closed)
to get a feel for what's already been proposed and what a good patch
looks like.

## Version History ##

* **0.4.1** (May 16, 2013):

  * Compatibility with Node 0.10.x.

* **0.4.0** (June 7, 2012):
  * Pow's port proxying feature lets you proxy virtual hosts to other
    ports on your computer. Just create a file in `~/.pow` with the
    virtual host as the filename and the port number as its contents,
    e.g. `echo 8080 > ~/.pow/myapp`.
  * Built-in support for [xip.io wildcard DNS](http://xip.io/) lets
    you access your Pow virtual hosts from other computers on your
    local network &mdash; perfect for testing your apps in Windows or
    on mobile devices. Just visit e.g. `http://myapp.10.0.0.1.xip.io/`
    (where 10.0.0.1 is your LAN IP address) instead of
    `http://myapp.dev/`.
  * You can now restart Pow itself by touching the
    `~/.pow/restart.txt` file.
  * If you use a shell other than Bash, Pow now properly loads your
    login environment, including `$PATH`.
  * An infinite loop bug that could cause Pow to lock up and consume
    99% CPU under certain circumstances has been fixed by replacing
    the bundled ndns library with an alternative.
  * The bundled nack library has been upgraded to version 0.13.2.
  * Due to the large number of issues it causes, support for
    automatically loading project `.rvmrc` files has been
    deprecated and will be removed in the next major release. See the
    "Specifying Ruby Versions with RVM" section of the manual for
    instructions on how to continue using RVM with Pow.

* **0.3.2** (August 13, 2011):
  * The bundled nack library has been upgraded to version 0.12.3 for
    improved streaming response body support.
  * The bundled Node binary has been upgraded to version 0.4.10.
  * If an `.rvmrc` file is present but rvm is not installed, Pow will
    fall back to whichever Ruby is in `$PATH` rather than displaying
    an error message. This means you can migrate from rvm to rbenv
    without needing to remove `.rvmrc` from your application.
  * A few changes have been made to the (now unmaintained) ndns
    library bundled with Pow in an attempt to avoid the infinite loop
    bug described in issue #99.

* **0.3.1** (May 11, 2011):
  * The `POW_EXT_DOMAINS` option should actually work now. Apologies.

* **0.3.0** (May 10, 2011):
  * The installation script now performs a self-test and attempts to
    reload the system network configuration if DNS resolution fails.
  * Configuration files are installed as mode `0644`, and
    `launchctl load` is invoked with the `-Fw` flags.
  * The uninstallation script now works with older versions of iTerm
    and removes installed files from `/etc/resolver`.
  * Pow can be configured to serve requests for unknown virtual hosts
    by creating a special catch-all symlink named `default` in
    `~/.pow`.
  * If the `tmp/always_restart.txt` file is present in an application,
    Pow will restart it before every request (useful with bare Rack
    apps or frameworks like Sinatra that don't manage code
    reloading).
  * `POW_TIMEOUT` is now specified in seconds instead of milliseconds,
    and a new `POW_EXT_DOMAINS` option defines a list of top-level
    domains that Pow will handle (but not serve DNS requests for).
  * Requests without a `Host:` header no longer raise an exception,
    and URLs with `..` are now properly passed through to the Rack
    application.
  * Error messages have been redesigned to include links to the
    manual and wiki.
  * A more helpful error message has been added for Rails 2.3 apps
    without a `config.ru` file.
  * You can print the full Pow configuration in shell format with
    `pow --print-config`, and third-party apps can request information
    about the running Pow server via the `http://pow/config.json` and
    `http://pow/status.json` endpoints.

* **0.2.2** (April 7, 2011): Initial public release.

## License ##

(The MIT License)

Copyright &copy; 2012 Sam Stephenson, 37signals

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
"Software"), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE
LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION
OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION
WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
