## How to create a Cronjob with Auto-Scaling-Group 

A Cron Job in Linux is a tool for scheduling tasks, like running scripts or programs at specific times. It works by setting up commands and schedules in a table called the crontab. The cron daemon (crond) checks this table and executes the commands according to the set schedule.

## Cron Job Schedule Syntax
A basic crontab entry looks something like this, with the cron job schedule first, followed by the command to run:

```bash
    *    *    *    *    *   /home/user/bin/somecommand.sh
    |    |    |    |    |            |
    |    |    |    |    |    Command or Script to execute
    |    |    |    |    |
    |    |    |    | Day of week(0-6 | Sun-Sat)
    |    |    |    |
    |    |    |  Month(1-12)
    |    |    |
    |    |  Day of Month(1-31)
    |    |
    |   Hour(0-23)
    |
    Min(0-59)
```