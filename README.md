# remote_syslog2

[![Download remote_syslog2](http://papertrail.github.io/remote_syslog2/images/download.png)][releases]

remote_syslog tails one or more log files and sends syslog messages to a
remote central syslog server. It generates packets itself, ignoring the system
syslog daemon, so its configuration doesn't affect system-wide logging.

Uses:

 * Collecting logs from servers & daemons which don't natively support syslog
 * When reconfiguring the system logger is less convenient than a
   purpose-built daemon (e.g., automated app deployments)
 * Aggregating files not generated by daemons (e.g., package manager logs)

This code is tested with the hosted log management service [Papertrail]
and should work for transmitting to any syslog server.

## Migrating from remote_syslog 1

remote_syslog2 is a rewrite of the ruby [remote_syslog] package. Not all
features of the ruby version are supported, and there are some backwards
incompatible changes.

### Which should I use?

Use remote_syslog2 (this README and application) unless you have a
specific reason to use remote_syslog1.

### Changes from remote_syslog 1

* The syntax of some command-line arguments have changed slightly,
though most are identical.
* Default hostname has been removed. Either the `host` config file
option or the `-d` invocation flag are required.



## Installing

Precompiled binaries for Mac, Linux and Windows are available on the
[remote_syslog2 releases page][releases].

Untar the package, copy the "remote_syslog" executable into your $PATH,
and then customize the included example_config.yml with the log file paths
to read and the host/port to log to.

Optionally, move and rename the configuration file to `/etc/log_files.yml` so
that remote_syslog picks it up automatically. For example:

    sudo cp ./remote_syslog /usr/local/bin
    sudo cp example_config.yml /etc/log_files.yml
    sudo vi /etc/log_files.yml

Configuration directives can also be specified as command-line arguments (below).

## Usage

    Usage of remote_syslog2:
      -c, --configfile="/etc/log_files.yml": Path to config
          --debug-log-cfg="": the debug log file
      -d, --dest-host="": Destination syslog hostname or IP
      -p, --dest-port=514: Destination syslog port
          --eventmachine-tail=false: No action, provided for backwards compatibility
      -f, --facility="user": Facility
          --hostname="": Local hostname to send from
          --log="<root>=INFO": set loggo config, like: --log="<root>=DEBUG"
          --new-file-check-interval={0}: How often to check for new files
      -D, --no-detach=false: Don't daemonize and detach from the terminal
          --no-eventmachine-tail=false: No action, provided for backwards compatibility
          --pid-file="": Location of the PID file
      -s, --severity="notice": Severity
          --tcp=false: Connect via TCP (no TLS)
          --tls=false: Connect via TCP with TLS


## Example

Daemonize and collect messages from files listed in `./example_config.yml` as
well as the file `/var/log/mysqld.log`. Write PID to `/tmp/remote_syslog.pid`
and send to port `logs.papertrailapp.com:12345`:

    $ remote_syslog -c example_config.yml -p 12345 --pid-file=/tmp/remote_syslog.pid /var/log/mysqld.log

Stay attached to the terminal, look for and use `/etc/log_files.yml` if it
exists, and send with facility local0 to `a.example.com:514`:

    $ remote_syslog -D -d a.example.com -f local0 /var/log/mysqld.log


## Auto-starting at boot

Sample init files can be found [in the examples directory](examples/). You may be able to:

    $ cp examples/remote_syslog.init.d /etc/init.d/remote_syslog
    $ chmod 755 /etc/init.d/remote_syslog

And then ensure it's started at boot, either by using:

    $ sudo update-rc.d remote_syslog defaults

or by creating a link manually:

    $ sudo ln -s /etc/init.d/remote_syslog /etc/rc3.d/S30remote_syslog

remote_syslog will daemonize by default.

Additional information about init files (`init.d`, `supervisor`, `systemd` and `upstart`) are
available [in the examples directory](examples/).


## Sending messages securely ##

If the receiving system supports sending syslog over TCP with TLS, you can
pass the `--tls` option when running `remote_syslog`:

    $ remote_syslog -D --tls -p 1234 /var/log/mysqld.log

or add `protocol: tls` to your configuration file.


## Configuration

By default, remote_syslog looks for a configuration in `/etc/log_files.yml`.

The archive comes with a [sample config](https://github.com/papertrail/remote_syslog2/blob/master/example_config.yml). Optionally:

    $ cp example_config.yml.example /etc/log_files.yml

`log_files.yml` has filenames to log from (as an array) and hostname and port
to log to (as a hash). Wildcards are supported using * and standard shell
globbing. Filenames given on the command line are additive to those in
the config file.

Only 1 destination server is supported; the command-line argument wins.

    files:
     - /var/log/httpd/access_log
     - /var/log/httpd/error_log
     - /var/log/mysqld.log
     - /var/run/mysqld/mysqld-slow.log
    destination:
      host: logs.papertrailapp.com
      port: 12345
      protocol: tls

remote_syslog sends the name of the file without a path ("mysqld.log") as
the syslog tag (program name).

After changing the configuration file, restart `remote_syslog` using the
init script or by manually killing and restarting the process. For example:

    /etc/init.d/remote_syslog restart


## Advanced Configuration (Optional)

Here's an [advanced config](https://github.com/papertrail/remote_syslog2/blob/master/examples/log_files.yml.example.advanced) which uses all options.

### Override hostname

Provide `--hostname somehostname` or use the `hostname` configuration option:

    hostname: somehostname


### Detecting new files

remote_syslog automatically detects and activates new log files that match
its file specifiers. For example, `*.log` may be provided as a file specifier,
and remote_syslog will detect a `some.log` file created after it was started.
Globs are re-checked every 10 seconds.

Note: messages may be written to files in the 0-10 seconds between when the
file is created and when the periodic glob check detects it. This data is not
acted on.

If globs are specified on the command-line, enclose each one in single-quotes
(`'*.log'`) so the shell passes the raw glob string to remote_syslog (rather
than the current set of matches). This is not necessary for globs defined in
the config file.


### Log rotation

External log rotation scripts often move or remove an existing log file
and replace it with a new one (at a new inode). The Linux standard script
[logrotate](http://iain.cx/src/logrotate/) supports a `copytruncate` config
option.  With that option, `logrotate` will copy files, operate on the copies,
and truncate the original so that the inode remains the same.

This comes closest to ensuring that programs watching these files (including
`remote_syslog`) will not be affected by, or need to be notified of, the
rotation. The only tradeoff of `copytruncate` is slightly higher disk usage
during rotation, so we recommend this option whether or not you use
`remote_syslog`.


### Excluding files from being sent

Provide one or more regular expressions to prevent certain files from being
matched.

    exclude_files:
      - \.\d$
      - .bz2
      - .gz


### Excluding lines matching a pattern

There may be certain log messages that you do not want to be sent.  These may be
repetitive log lines that are "noise" that you might not be able to filter out
easily from the respective application.  To filter these lines, use the
exclude_patterns with an array or regexes:

    exclude_patterns:
     - exclude this
     - \d+ things


### Multiple instances

Run multiple instances to specify unique syslog hostnames.

To do that, provide an alternate PID path as a command-line option to the
additional instance(s). For example:

    --pid-file=/var/run/remote_syslog_2.pid

Note: Daemonized programs use PID files to identify whether the program is already
running ([more](http://unix.stackexchange.com/questions/12815/what-are-pid-and-lock-files-for/12818#12818)). Like other daemons, remote_syslog will refuse to run as a
daemon (the default mode) when a PID file is present. If a .pid file is
present but the daemon is not actually running, remove the PID file.

### Choosing app name

remote_syslog uses the log file name (like "access_log") as the syslog
program name, or what the syslog RFCs call the "tag." This is ideal unless
remote_syslog watches many files that have the same name.

In that case, tell remote_syslog to set another program name by creating
symbolic link to the generically-named file:

    cd /path/to/logs
    ln -s generic_name.log unique_name.log

Point `remote_syslog` at `unique_name.log`. `remote_syslog` will send its
contents with the program name `unique_name.log`.


## Troubleshooting

### Generate debug log

To output debugging events with maximum verbosity, run:

```
remote_syslog --debug-log-cfg=logfile.txt --log="<root>=DEBUG"
```

.. as well as any other arguments which are used in normal operation. This 
will set [loggo](https://github.com/juju/loggo#func-parseconfigurationstring)'s
root logger to the `DEBUG` level and output to `logfile.txt`.

### Truncated messages

To send messages longer than 1024 characters, use TCP (either TLS or cleartext
TCP) of UDP. See "[Sending messages securely](#sending-messages-securely)" to
use TCP with TLS for messages of any length.

[Here's why](http://help.papertrailapp.com/kb/configuration/troubleshooting-remote-syslog-reachability/#message-length) longer UDP messages are impossible to send over
the Internet.

### inotify

When running remote_syslog in the foreground using the `-D` switch, if you
receive the error:

    Error creating fsnotify watcher: inotify_init: too many open files

determine the maximum number of inotify instances that can be created using:

    cat /proc/sys/fs/inotify/max_user_instances

and then increase this limit using:

    echo VALUE >> /proc/sys/fs/inotify/max_user_instances

where VALUE is greater than the present setting. Confirm that remote_syslog starts
up and then apply this new value permanently by adding the following to
`/etc/sysctl.conf:`:

    fs.inotify.max_user_instances = VALUE

### "No space left on device"

When monitoring a large number of files, this error may occur:

    FATAL -- Error watching /path/here : no space left on device

To solve this, determine the maximum number of user watches that can be
created using:

    cat /proc/sys/fs/inotify/max_user_watches

and then increase them using:

    echo VALUE >> /proc/sys/fs/inotify/max_user_watches

Once again, confirm that remote_syslog starts and then apply this value permanently by adding the following to `/etc/sysctl.conf:`:

    fs.inotify.max_user_watches = VALUE

## Credits

* [Paul Morton](https://twitter.com/mortonpe)
* [Papertrail](https://papertrailapp.com/) staff
* [Paul Hammond](http://paulhammond.org/)

## Reporting bugs

1. See whether the issue has already been reported: <https://github.com/papertrail/remote_syslog2/issues/>
2. If you don't find one, create an issue with a repro case.


## Development

remote_syslog2 is written in go, and uses [godep] to manage
dependencies. To get everything set up, [install go][goinstall] then
run:

    go get github.com/kr/godep
    go get github.com/mitchellh/gox
    go get github.com/papertrail/remote_syslog2

To run tests:

    # run all tests
    godep go test ./...
    # run all tests except the slower syslog reconnection tests
    godep go test -short ./...


## Building

    make


## Contributing

Once you've made your great commits:

1. [Fork][fk] remote_syslog
2. Create a topic branch - `git checkout -b my_branch`
3. Commit the changes without changing the Rakefile or other files unrelated to your enhancement.
4. Push to your branch - `git push origin my_branch`
5. Create a Pull Request or an [Issue][is] with a link to your branch
6. That's it!


[Papertrail]: http://papertrailapp.com/
[remote_syslog]: https://github.com/papertrail/remote_syslog
[releases]: https://github.com/papertrail/remote_syslog2/releases

[godep]: https://github.com/kr/godep
[goinstall]: http://golang.org/doc/install

[fk]: http://help.github.com/forking/
[is]: https://github.com/papertrail/remote_syslog/issues/
