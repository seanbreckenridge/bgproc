# bgproc

A `bash` loop to run tasks in the background. Used as a `anacron` alternative.

This uses [`evry`](https://github.com/seanbreckenridge/evry) to schedule commands/run things periodically. `evry` saves persistent files with timestamps to the computer for each job, which means this follows `anacron`s philosophy - the computer doesn't have to be running 24 x 7. `evry` checks when tasks were last run, and if that duration has elapsed (e.g. `2 days`), it runs the task.

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
Usage: bgproc [-h] [-nodpqjJ] [DIR...]
Runs tasks in the background. Run without flags to start the background loop
	-n	Don't search directories recursively (add -maxdepth 1)
	-o	Runs the task loop once
	-d	Runs the task loop once, in debug mode
	-p	Runs the task loop thrice, to pretty print debug info
	-q	Quiet mode, silences any logs
	-j	Print paths of all jobs, then exit
	-J	Print paths of all job directories, then exit
Any additional arguments should be directories which contain '.job' files
If no directories are provided, searches from the current directory recursively
See https://github.com/seanbreckenridge/bgproc for more info
```

See [here](https://gist.github.com/seanbreckenridge/e7ad77320c065d96f282f6d45deaa842) for example debug output.

This offers a few ways to run the task loop once, `-o` (once), `-d` (with lots of debug information) or `-p`, in pretty mode, which tells you when each job will run next:

```bash
$ bgproc -p ~/.local/scripts/supervisor_jobs/linux
glue-update-cubing-json - 5 days, 1 hour, 49 minutes, 47 seconds
warn_mailsync - 9 minutes, 27 seconds
bg-my-feed-index - 34 minutes, 48 seconds
copy_images - 4 minutes, 27 seconds
linkmusic - 46 minutes, 13 seconds
guestbook_comments - 10 minutes, 18 seconds
updaterss - 2 minutes, 45 seconds
```

### Install

Copy the `bgproc` script onto your `$PATH` somewhere and make it executable. To automate, could use:

- `sinister`: `sh <(curl -sSL http://git.io/sinister) -u 'https://raw.githubusercontent.com/seanbreckenridge/bgproc/master/bgproc'`
- [`basher`](https://github.com/basherpm/basher): `basher install seanbreckenridge/bgproc`

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

evry 2 hours -somecommand && {
  printlog "some command: running..."  # saves timestamp to logfile
  somecommand || send-error "some command failed..."  # notifies me if this fails
}
```

### Configuration

This waits for `60 seconds` between running jobs, if you want to increase/change that, you can set the `BGPROC_SLEEPTIME` environment variable. To wait for 10 minutes between trying to run jobs: `BGPROC_SLEEPTIME=600 bgproc`

If you want to run multiple `bgproc` instances for different directories/jobs, put `bgproc` on your `$PATH`, and set the `BGPROC_LOCKFILE` environment variable to allow multiple instances of `bgproc` to run at the same time:

```bash
BGPROC_LOCKFILE=/tmp/personal_jobs.lock BGPROC_LOGFILE=/tmp/personal_logs bgproc /some/other/directory
```

To change the date format, you can set the `BGPROC_DATE_FMT` environment variable, that is passed to the `date` command, for more info see `man date`:

```bash
BGPROC_DATE_FMT='+%Y-%m-%dT%H-%M-%S' ./bgproc -o ./jobs
```

## bgproc_on_machine

I use `bgproc` on all of my machines and my phone, so `bgproc_on_machine` handles the task of figuring out which machine I'm currently on, so the correct background `.job`s can run. That uses [`on_machine`](https://github.com/seanbreckenridge/on_machine) internally, which generates a unique hash, like: `linux_arch` or `android_termux`.

After setting the `$BGPROC_PATH` environment variable:

```bash
$ export BGPROC_PATH="${HPIDATA}/jobs:${HOME}/.local/scripts/supervisor_jobs:${REPOS}/HPI-personal/jobs"
$ bgproc_on_machine -o
1655993990:Searching for jobs in:
1655993990:/home/sean/data/jobs/all
1655993990:/home/sean/data/jobs/linux
1655993990:/home/sean/.local/scripts/supervisor_jobs/all
1655993990:/home/sean/.local/scripts/supervisor_jobs/linux
1655993990:/home/sean/Repos/HPI-personal/jobs/all
1655993990:/home/sean/Repos/HPI-personal/jobs/linux
```

You can see examples of those directory structures [in my dotfiles](https://github.com/seanbreckenridge/dotfiles/tree/master/.local/scripts/supervisor_jobs) and [in my personal HPI repo](https://github.com/seanbreckenridge/HPI-personal/tree/master/jobs):

```
jobs
├── all
│   ├── backup_bash.job
│   ├── backup_browser_history.job
│   ├── backup_ipython.job
│   ├── backup_zsh_history.job
│   ├── doctor_snapshot.job
│   ├── minecraft_advancements.job
│   └── runelite_screenshots.job
├── android
├── linux
│   ├── backup_albums.job
│   ├── backup_bash_server_history.job
│   ├── backup_chess.job
│   ├── backup_garmin.job.disabled
│   ├── backup_ghexport.job
│   ├── backup_git_doc_history.job
│   ├── backup_listenbrainz.job
│   ├── backup_rexport.job
│   ├── backup_stexport.job
│   ├── backup_trakt.job
│   └── mint.job
└── mac
    ├── backup_imessages.job
    └── backup_safari_history.job
```

#### Background Service

This doesn't offer a way to run this automatically, thats should be handled by you. Could potentially use a `systemd` service (on linux flavors that have that) or an [Automator script](https://stackoverflow.com/questions/6442364/running-script-upon-login-mac) on macOS.

Personally I run this with [`supervisor`](https://github.com/Supervisor/supervisor) (since it being cross platform means my background processes are platform agnostic) at the beginning of my X session on linux, and [check whenever I open a terminal on mac](https://github.com/seanbreckenridge/dotfiles/blob/master/.config/zsh/mac.zsh) - see [here](https://github.com/seanbreckenridge/dotfiles/tree/master/.local/scripts)

On [`termux`](https://termux.com/) (android), where handling background tasks is a bit more complicated, instead of running this in the background, I use the `-o` flag to run the loop once a day, [checking when I open a terminal](https://github.com/seanbreckenridge/dotfiles/blob/master/.config/zsh/android.zsh). In my shell profile, I put something like:

`evry 1 day -run_android_jobs && bgproc -o ~/.local/scripts/supervisor_jobs`

### Performance

Despite this running `evry` all the time to check if times have elapsed, it doesn't cause much of a footprint, each call takes about `1ms`:

```
$ hyperfine --warmup 3 -i 'evry 1 day -run_benchmark'
Benchmark #1: evry 1 day -run_benchmark
  Time (mean ± σ):       1.0 ms ±   0.7 ms    [User: 0.6 ms, System: 0.6 ms]
  Range (min … max):     0.1 ms …   3.0 ms    1329 runs
```
