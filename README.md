# bgproc

A `bash` loop to run tasks in the background. Used as a `anacron` alternative.

Uses a lock file at `/tmp/bgproc.lock` to make sure duplicate `bgproc` processes aren't running, though it wouldn't make a huge difference since commands are run behind `evry` anyways.

This uses [`evry`](https://github.com/seanbreckenridge/evry) to schedule commands/run things periodically. `evry` saves persistent files with timestamps to the computer for each job, which means this follows `anacron`s philosophy - the computer doesn't have to be running 24 x 7. `evry` checks when tasks were last run, and if that duration has elapsed (e.g. `2 days`), it runs the task.

## How?

This runs any other files it finds recursively with `find` from the current directory that end with `.job`. You could also put this on your `$PATH`, `cd` to some directory that has your `.job` scripts, and run it from there.

A potential `.job` file might look like:

```bash
#!/bin/bash
# backup the logfile from my server once a day

evry 1 day -backup_logfile && {
  scp vps_server:~/app.log ~/.cache/app.log
}
```

This runs each `.job` file explicitly with `bash`, but you could easily write a wrapper like:

```bash
#!/bin/bash
# every 2 days, run some python script

evry 2 days -my_task && {
  printlog "running python script..."
  exec python3 /usr/local/bin/run_task.py
}
```

## Configuration

Usage:

```
Runs tasks in the background. Run without flags to start the background loop
	-o		Runs the task loop once
	-d		Runs the task loop once, in debug mode
```

See [here](https://gist.github.com/seanbreckenridge/e7ad77320c065d96f282f6d45deaa842) for example debug output.

---

If you want to save logs somewhere else, you can set the `BGPROC_LOGFILE` variable to a different location. Defaults to saving temporary logs at `/tmp/bgproc.log`

Logs are very basic, just saves the timestamp and the message passed (see the `printlog` function), like:

```
1597283664:Job List:
1597283664:./personal_jobs/raspi.job
1597283664:./personal_jobs/update_rss.job
1597283664:Starting loop...
```

The `printlog` function is exported into the bash environment, so its accessible from any other bash scripts `bgproc` runs; e.g. its used in [`personal_jobs/raspi.job`](personal_jobs/raspi.job).

---

This waits for `60 seconds` between running jobs, if you want to increase/change that, you can set the `BGPROC_SLEEPTIME` environment variable. To wait for 10 minutes between trying to run jobs:

`BGPROC_SLEEPTIME=600 bgproc`

---

If you want to run multiple `bgproc` instances for different directories/jobs, put `bgproc` on your `$PATH`, `cd` elsewhere, and set the `BGPROC_LOCKFILE` to allow multiple instances of `bgproc` to run at the same time:

```
cd /some/other/directory/
BGPROC_LOCKFILE=/tmp/personal_jobs BGPROC_LOGFILE=/tmp/personal_logs bgproc
```

For an example wrapper, see my [`HPI`](https://github.com/seanbreckenridge/HPI/blob/master/bgproc) `bgproc` wrapper.

This doesn't offer a way to run this automatically, thats should be handled by you. To daemonize this, I run this at the beginning of my X session (on linux). Can also run (kill/restart) it with `pkill bgproc`/`setsid bgproc`. Could potentially use a `systemd` service (on linux flavors that have that) or an [Automator script](https://stackoverflow.com/questions/6442364/running-script-upon-login-mac) on macOS.

### Performance

Despite this running `evry` all the time to check if times have elapsed, it doesn't cause much of a footprint, each call takes about `1ms`:

```
$ hyperfine --warmup 3 -i 'evry 1 day -run_benchmark'
Benchmark #1: evry 1 day -run_benchmark
  Time (mean ± σ):       1.0 ms ±   0.7 ms    [User: 0.6 ms, System: 0.6 ms]
  Range (min … max):     0.1 ms …   3.0 ms    1329 runs
```
