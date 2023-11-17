# freebsd-puma 

>[!WARNING]  
>ARCHIVED: Please use <https://github.com/gamecreature/freebsd-puma>.


An init script for running puma on FreeBSD. Based on [freebsd-sidekiq].

This rc script works with either RVM or RBENV installed in your deploy user's directory, or with a globally installed ruby.

This script runs puma via the Freebsd daemon tool.
So it should be compatible with the latest puma version, which dropped internal daemonizing support (https://github.com/puma/puma/blob/master/5.0-Upgrade.md)

Simply place the `puma` script in your `/usr/local/etc/rc.d` directory, modify it if necessary, and configure your application via variables in `/etc/rc.conf`
This has been tested on **FreeBSD 12.1** and **FreeBSD 13.2**

## Make sure puma starts after your database launches!

The only thing you might need to configure in the rc script is to change the `REQUIRE` line to specify your database (I use PostreSQL so that's what's in the repo)

For example, if you were using MySQL, you would change
    # REQUIRE: LOGIN postgresql
to
    # REQUIRE: LOGIN mysql-server

You might need to add other services to this list if your Rails application requires them.

## Note there are 2 scripts here

`puma` names the service 'puma'
`puma5` names the service 'puma5' (In case you would like to mix puma versions)


## Quick Setup

To get up and running quickly, adjust the `REQUIRE` line like above, and add edit your `/etc/rc.conf`:

For Capistrano or Capistrano-like directory layouts:

    puma_enable="YES"

    # this is the path to where your application is deployed via Capistrano
    # (the parent directory of the `current` directory)
    puma_directory="/u/application"


For Non-Capistrano-like layouts:

    puma_enable="YES"
    puma_command="/u/application/bin/puma"
    puma_pidfile="/u/application/tmp/pids/puma.pid"
    puma_config="/u/application/config/puma.rb"
    puma_chdir="/u/application"
    puma_user="deploy"

## Starting/Stopping/Restarting and Upgrading puma

You can now start puma like any other FreeBSD service:

    /usr/local/etc/rc.d/puma start

There's also a handy `show` command to look at your final puma configuration:

    /usr/local/etc/rc.d/puma show

To prepare shutdown, you can send a quiet (prestop) command.
(kill TSTP)
Note: I tried to name it quiet, but that is used by FreeBSD internally?

    /usr/local/etc/rc.d/puma stop


## `/etc/rc.conf` Details

### Using a Capistrano directory layout

The rc script does as much as possible to help you out. If you are using Capistrano, or a Capistrano-like directory structure, then you can just specify the directory of your application (the parent directory of `current`):

    puma_enable="YES"
    puma_directory="/u/application"

This infers all sorts of information about your app (you can always run `/usr/local/etc/rc.d/puma show` to see what your configuration is. **Note** the variable names listed here are without the leading `puma_` prefix that you would need to specify in `/etc/rc.conf`):

#
# puma Configuration 
#

    command:        /u/app/current/bin/puma
    command_args:   /u/app/current/config.ru
    rackup:         /u/app/current/config.ru
    pidfile:        /u/app/shared/tmp/pids/puma.pid
    old_pidfile:    /u/app/shared/tmp/pids/puma.pid.oldbin
    listen:
    config:         /u/app/current/config/puma.rb
    log:            /u/app/current/log/puma.log
    init_config:    /u/app/current/.env
    bundle_gemfile: /u/app/current/Gemfile
    chdir:          /u/app/current
    user:           user
    nice:
    env:            production
    flags:          -e production -C /u/app/current/config/puma.rb

    start_command:

    su -l username -c "export BUNDLE_GEMFILE=/u/app/current/Gemfile && . /u/app/current/.env && cd /u/app/current && /usr/sbin/daemon -f -p /u/app/shared/tmp/pids/puma.pid -o /u/app/current/log/puma.log /u/app/current/bin/puma -e production -C /u/app/current/config/puma.rb  /u/app/current/config.ru "


Let's look at these settings one by one:

`command`: By default, it uses the `current/bin/puma` [bundler binstub][binstub] located in your project to ensure your gems are loaded. `command` comes from FreeBSD's `rc.subr` init system functions.

`command_args`: This is the standard FreeBSD's `rc.subr` variable that holds the arguments to the above `command`.  Typically you don't need to set this.

`pidfile`: This is also part of FreeBSD's `rc.subr` system. This is where the built in functions will look for the pid of the process. By default, this rc script looks in the `current/tmp/pids/puma.pid` file.

`config`: This is the path to puma's config file where puma will find it's settings. By default this rc script looks for a file called `current/config/puma.rb`

`init_config`: This is a shell script file that is included in the environment before puma is executed. In this file you can include `export VAR=value` statements to pass environment variables into your rails app. By default, this init script looks for a file called `current/.env` and uses that. If that file doesn't exist, this rc script will skip this functionality (as seen in the above example).

This could be used in conjunction with [dotenv][dotenv] in development since dotenv accepts lines beginning with `export`


`bundle_gemfile`: This is the path to the `Gemfile` of your project. This rc script sets the `BUNDLE_GEMFILE` environment variable to this value. By default it looks to `current/Gemfile`. This is required so that puma uses the most current `Gemfile` (rather than the one in the specific deployment directory) when an upgrade is performed.

`chdir`: This is the directory we `cd` into before running puma. By default it's the currently deployed version of your application "`current/`"

`user`: This is the user that puma will be run as.

`nice`: The `nice` level to run puma at. Usually you'll leave this alone.

`flags`: This is a variable defined by FreeBSD's `/etc/rc.subr` init system, and contains the flags passed to the command (puma in this case) when run. This variable is built up from the variables above, but you could manually specify `puma_flags` in your `/etc/rc.conf` to override them.

`start_command`: Here you can see the full command that will be run when you start this service. It's a beaut' isn't it?

You can override any of these parameter in your `/etc/rc.conf` by simply specifying the variables like you see below (you can pick and choose which to override).


### Using a custom directory layout

Using your own layout is easy, you can just leave the `puma_directory` variable out of your `/etc/rc.conf` and specify all of the above variables manually. Here's a list of those variables for your convenience:

    puma_command:      The path to the puma command
    puma_command_args: The non-flag arguments passed to the above command.  Typically you do not need to set this.
    puma_pidfile:      The path where puma will put its pid file
    puma_config:       The path to the puma config file
    puma_log:          The path to the puma log file
    puma_chdir:        The path where this script will `cd` to before starting puma
    puma_user:         The user to run puma as
    puma_nice:         The `nice` level to run puma as. Leave blank to run un-niced
    puma_env:          The RAILS_ENV (or RACK_ENV) to run your application as. (default: production)
    puma_flags:        The flags passed in to puma when starting (not counting the puma_command_args specified above). Override this for complete control of how to start puma.

### Deploying multiple applications

This is all find and dandy, but some of you might have multiple applications running on the same server (even if it's just a staging and production version of your app).

This rc script can work with multiple applications. It works similarly to how postgresql's rc script works on FreeBSD.

You simply specify your profiles in your `/etc/rc.conf` with the `puma_profiles` variable. This is a space separated list of application names.

Then you can customize each application by specifying variables in this form:

    puma_<application-name>_variable=VALUE

Here's a simple example (I can leave the _env variable out of the production declaration since it's the default value)

    puma_enable="YES"
    puma_profiles="application_staging application_production"

    puma_application_staging_enable="YES"
    puma_application_staging_directory="/u/application_staging"
    puma_application_staging_env="staging"

    puma_application_production_enable="YES"
    puma_application_production_directory="/u/application_production"

You can use the simplified Capistrano `directory`-based configuration like above, or you can specify all of the variable's separately, for a fully custom setup

### Customizing the script

If you want to customize the default, calculated values you want to look in the `_setup_directory()` function. This is what is called when you specify that you want to use a Capistrano-like directory layout by specifying `puma_directory` or `puma5_<application-name>_directory` in your `/etc/rc.conf`.

If you use a different deployment strategy than Capistrano, you could adjust the default values to work with your system.


[freebsd-puma5]: https://github.com/snake66/freebsd-puma5
[dotenv]: https://github.com/bkeepers/dotenv
[binstub]: https://github.com/sstephenson/rbenv/wiki/Understanding-binstubs
[freebsd-unicorn]: https://github.com/caleb/freebsd-unicorn
[Caleb]: https://github.com/caleb
