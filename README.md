# bgproc

A `bash` loop to run tasks in the background. Used as a `anacron` alternative.

Uses a lock file at `/tmp/bgproc.lock` to make sure duplicate `bgproc` processes aren't running, though it wouldn't make a huge difference since commands are run behind `evry` anyways.

Uses:
  * [`evry`](https://github.com/seanbreckenridge/evry) to schedule commands/run things periodically. `evry` saves persistent files with timestamps to the computer for each job, which means this follows `anacron`s philosopy - the computer doesn't have to be running 24 x 7. `evry` checks when tasks were last run, and if that duration has elapsed (e.g. `2 days`), it runs the task.
  * [`wait-for-internet`](https://github.com/seanbreckenridge/wait-for-internet) to make sure the computer has a remote connection before running any jobs

This runs any other files it finds recursively with `find` from the current directory that end with `.job`. Any script that ends with `.priv.job` isn't synced to git. You could potentially put this on your `$PATH`, and then `cd` to some directory that has your `.job` scripts, and run it from there.

This runs each `*.job` file directly with `bash`, but you could easily write a wrapper like:

`some_task.job`:

```
#!/bin/bash
exec python3 /usr/local/bin/run_task.py
```

If you want to save logs somewhere else, you can set the `BGPROC_LOGFILE` variable to a different location. Defaults to saving temporary logs at `/tmp/bgproc.log`

Logs are very basic, just saves the timestamp and the message passed (see the `printlog` function), like:

```
1597283664:Job List:
1597283664:./personal_jobs/raspi.job
1597283664:./personal_jobs/update_rss.job
1597283664:Starting loop...
```

The `printlog` function is exported into the bash environment, so its accessible from any other bash scripts `bgproc` runs; e.g. its used in [`personal_jobs/raspi.job`](personal_jobs/raspi.job).

If you want to run multiple `bgproc` instances for different directories/jobs, you can change the lockfile to allow different instances to run like:

```
BGPROC_LOCKFILE=/tmp/personal_jobs BGPROC_LOGFILE=/tmp/personal_logs ./bgproc
```

For an example wrapper, see my [`hpi`](https://github.com/seanbreckenridge/HPI/blob/master/bgproc) repo.

This doesn't offer a way to run this automatically, thats should be handled by you. To daemonize this, I run this at the beginning of my X session (on linux). Can also run (kill/restart) it with `pkill bgproc`/`setsid ./bgproc`. Could potentially use a `systemd` service (on linux flavors that have that) or an [Automator script](https://stackoverflow.com/questions/6442364/running-script-upon-login-mac) on macOS.
