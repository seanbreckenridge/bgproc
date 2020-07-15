# bgproc

A bash loop to run tasks in the background. Used as a cron alternative.

Uses a lock file at `/tmp/bgproc.lock` to make sure duplicates aren't running, though it wouldn't make a huge difference since commands are run behind `evry` anyways.

Uses:
  * [evry](https://github.com/seanbreckenridge/evry) to schedule commands/run things periodically
  * [wait-for-internet](https://github.com/seanbreckenridge/wait-for-internet) to make sure the computer has a remote connection before running jobs
  * basic [havecmd](https://sean.fish/d/havecmd?dark) script to test if commands exist

This also runs any other `bash` scripts in this directory that end with `.job`

To daemonize this, I run this at the beginning of my X session. Can also run kill/restart it with `pkill bgproc`/`setsid ./bgproc`
