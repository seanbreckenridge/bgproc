# bgproc

A `bash` loop to run tasks in the background. Used as a `anacron` alternative.

This uses [`evry`](https://github.com/seanbreckenridge/evry) to schedule commands/run things periodically. `evry` saves persistent files with timestamps to the computer for each job, which means this follows `anacron`s philosophy - the computer doesn't have to be running 24 x 7. `evry` checks when tasks were last run, and if that duration has elapsed (e.g. `2 days`), it runs the task.

You can see my jobs/bgproc wrapper [here](https://github.com/seanbreckenridge/dotfiles/tree/master/.local/scripts/supervisor), which I split into jobs specific to linux/mac.

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

## Usage

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

### Logs

If you want to save logs somewhere else, you can set the `BGPROC_LOGFILE` environment variable to a different location. Defaults to saving temporary logs at `/tmp/bgproc.log`

Logs are very basic, just saves the timestamp and the message passed like:

```
1613892690:Starting loop...
1613892693:updaterss:updated RSS feeds:0
```

Both the [`printlog` and `send-error`](https://github.com/seanbreckenridge/bgproc/blob/2b4a2a021bd0ccf0d7ea8d2557e8c5c816e05b49/bgproc#L34-L54) functions are exported into the bash environment, so they're accessible from any bash scripts `bgproc` runs. Both of those accept one argument - the text to print. `send-error` sends a OS notification if possible, using `notify-send` on linux and `osascript` (AppleScript) on mac.

For reference, my jobs often follow a structure like this:

```bash
#!/usr/bin/env bash

evry 2 hours -somecommand {
  printlog "some command: running..."  # saves timestamp to logfile
  somecommand || send-error "some command failed..."  # notifies me if this fails
}
```

### Configuration

This waits for `60 seconds` between running jobs, if you want to increase/change that, you can set the `BGPROC_SLEEPTIME` environment variable. To wait for 10 minutes between trying to run jobs:

`BGPROC_SLEEPTIME=600 bgproc`

If you want to run multiple `bgproc` instances for different directories/jobs, put `bgproc` on your `$PATH`, and set the `BGPROC_LOCKFILE` environment variable to allow multiple instances of `bgproc` to run at the same time:

```
BGPROC_LOCKFILE=/tmp/personal_jobs.lock BGPROC_LOGFILE=/tmp/personal_logs bgproc /some/other/directory
```

---

This doesn't offer a way to run this automatically, thats should be handled by you. To daemonize this, I run this with [`supervisor`](https://github.com/Supervisor/supervisor) (since it being cross platform means my background processes are platform agnostic) at the beginning of my X session on linux, and [check whenever I open a terminal on mac](https://github.com/seanbreckenridge/dotfiles/blob/master/.config/zsh/mac.zsh).

Could potentially use a `systemd` service (on linux flavors that have that) or an [Automator script](https://stackoverflow.com/questions/6442364/running-script-upon-login-mac) on macOS.

### Performance

Despite this running `evry` all the time to check if times have elapsed, it doesn't cause much of a footprint, each call takes about `1ms`:

```
$ hyperfine --warmup 3 -i 'evry 1 day -run_benchmark'
Benchmark #1: evry 1 day -run_benchmark
  Time (mean ± σ):       1.0 ms ±   0.7 ms    [User: 0.6 ms, System: 0.6 ms]
  Range (min … max):     0.1 ms …   3.0 ms    1329 runs
```
