# bgproc

A `bash` loop to run tasks in the background. Used as a `anacron` alternative.

Uses a lock file at `/tmp/bgproc.lock` to make sure duplicate `bgproc` processes aren't running, though it wouldn't make a huge difference since commands are run behind `evry` anyways.

This uses [`evry`](https://github.com/seanbreckenridge/evry) to schedule commands/run things periodically. `evry` saves persistent files with timestamps to the computer for each job, which means this follows `anacron`s philosophy - the computer doesn't have to be running 24 x 7. `evry` checks when tasks were last run, and if that duration has elapsed (e.g. `2 days`), it runs the task.

You can see my jobs/bgproc wrapper [here](https://github.com/seanbreckenridge/dotfiles/tree/master/.local/scripts/supervisor)

## How?

This runs any other files it finds recursively with `find` from the current directory that end with `.job`. You can alternatively provide directories which contain `.job` files as positional arguments.

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
Usage: bgproc [-o] [-d] [DIR...]
Runs tasks in the background. Run without flags to start the background loop
	-o		Runs the task loop once
	-d		Runs the task loop once, in debug mode
Any additional arguments should be directories which contain '.job' files
If no directories are provided, searches from the current directory recursively
See https://github.com/seanbreckenridge/bgproc for more info
```

See [here](https://gist.github.com/seanbreckenridge/e7ad77320c065d96f282f6d45deaa842) for example debug output.

### Install

Copy the `bgproc` script onto your `$PATH` somewhere and make it executable. To automate:

`sh <(curl -sSL http://git.io/sinister) -u 'https://raw.githubusercontent.com/seanbreckenridge/bgproc/master/bgproc'`

You could alternatively clone this repository and create a 'jobs' folder (the name doesn't particularly matter), placing `.job` files in that directory. Then, just run the script like `./bgproc` while in this directory, it doesn't have to be on your `$PATH`.

### Configuration

If you want to save logs somewhere else, you can set the `BGPROC_LOGFILE` environment variable to a different location. Defaults to saving temporary logs at `/tmp/bgproc.log`

Logs are very basic, just saves the timestamp and the message passed (see the `printlog` function), like:

```
1613892690:Starting loop...
1613892693:updaterss:updated RSS feeds:0
1613892926:Sleep duration: 60
```

The `printlog` function (which appends output to STDOUT and the bgproc logfile) is exported into the bash environment, so its accessible from any other bash scripts `bgproc` runs.

---

This waits for `60 seconds` between running jobs, if you want to increase/change that, you can set the `BGPROC_SLEEPTIME` environment variable. To wait for 10 minutes between trying to run jobs:

`BGPROC_SLEEPTIME=600 bgproc`

---

If you want to run multiple `bgproc` instances for different directories/jobs, put `bgproc` on your `$PATH`, and set the `BGPROC_LOCKFILE` environment variable to allow multiple instances of `bgproc` to run at the same time:

```
BGPROC_LOCKFILE=/tmp/personal_jobs.lock BGPROC_LOGFILE=/tmp/personal_logs bgproc /some/other/directory
```

This doesn't offer a way to run this automatically, thats should be handled by you. To daemonize this, I run this with [`supervisor`](https://github.com/Supervisor/supervisor) (since it being cross platform means my background processes are platform agnostic) at the beginning of my X session (on linux). Could potentially use a `systemd` service (on linux flavors that have that) or an [Automator script](https://stackoverflow.com/questions/6442364/running-script-upon-login-mac) on macOS.

### Performance

Despite this running `evry` all the time to check if times have elapsed, it doesn't cause much of a footprint, each call takes about `1ms`:

```
$ hyperfine --warmup 3 -i 'evry 1 day -run_benchmark'
Benchmark #1: evry 1 day -run_benchmark
  Time (mean ± σ):       1.0 ms ±   0.7 ms    [User: 0.6 ms, System: 0.6 ms]
  Range (min … max):     0.1 ms …   3.0 ms    1329 runs
```
