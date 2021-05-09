+++
title = "Writing effective service files in Linux for your Application"
date = 2020-11-06T23:08:58+05:30
images = []
tags = []
categories = ["linux", "systemd", "service_files", "supervisord"]
draft = false
+++
![Service](/posts/effective-service-files/gears.png)

## TLDR;

If you would like to write a basic Systemd service file or a Supervior conf, follow the links [here (systemd)](https://gist.github.com/beingadityak/ab2e46988cccc0a5e6dd289065551d8d) and [here (supervisor)](https://gist.github.com/beingadityak/c71e157ae79822c3c3587ee427498317). These contain a basic template of the service file. You can modify the parts of it as per your requirements. Once done, reload the daemons (both supervisor & systemd) and the services will be running.

## Context

Whenever you had a requirement of deploying your application on the server (be it Amazon EC2, Linode instances etc.) the thought of the deployment process always comes in late. You generally tend to use some kind of a hacky solution, such as using **tmux** for running your application. Granted that it is a great tool but it has a purpose of allowing you to run multiple windows/panels of your terminal. It can also be used for running some background tasks, but it is _not really scalable_. You would either run the tmux session manually when you try to horizontally scale your application or come up with a script that automates this for you - which is unnecessary engineering to me.

If you're running on Linux instances, well you're in luck! You have 2 options to run your application. You can either use the Linux-native process manager (here I'll be talking about Ubuntu & Debian related one - Systemd) or you use [Supervisor](http://supervisord.org/) which allows you to use a single conf file across Linux systems, as long as the system is having the Supervisor daemon installed (`supervisord`)

## Pre-requisites

1. Your application should be executable as a binary or it should have a shell script for starting the appication.
2. The script/binary should be accessible via the root user
3. (Optional) You can have an additional user whose purpose is to execute the application/script

## Process - the SystemD way

### 1. Create the service file

You can write the service file (systemd format) for your application with this template:

```
[Unit]
Description=<Application/service description>
[Service]
Type=simple
Restart=always
RestartSec=5s
ExecStart=<Binary/shell script location>
[Install]
WantedBy=multi-user.target
```

This file has multiple parts to be understood. Mainly, any service file is having 3 main sections: `[Unit]`, `[Service]` and `[Install]`. These define the following:

1. `[Unit]`: This section helps to create a description of the service, specify the order (using `Before` and `After` fields), specify the requirements (`Requires` field) as well as descibe conflicts for the service (using `Conflicts` field.)
2. `[Service]`: This section is the main part of a service file. This describes the type; the command to run when starting, stopping and reloading the service.
3. `[Install]`: You can define the alias, define the required services to start before this as well as define the services to enable/disable when the user enables/disables the service.

We'll now go over the parts mentioned in the template.

- Following are the fields which are the part of `[Unit]` section in the service file:

    - `Description` : The description of the service for which we want to control as a process.

- Following are the fields which are the part of `[Service]` section in the service file:

    - `Type`: Here we define the type of the service. For the simplicity purposes we keep it as `simple`.
    - `Restart`: Here we specify whether to restart whenever the application exits with a zero or a non-zero code. We're keeping it to `always`.
    - `RestartSec`: Here we specify the interval for sleeping before restarting a service. This can be a unit value in seconds or a timespan value such as `1min 20s`. Useful when used in conjuction with `Restart`.
    - `ExecStart`: The name of the script/binary to execute when the service is started.

- Following are the fields which are the part of `[Install]` section in the service file:

    - `WantedBy`: The current service will be started after the listed service is started. To keep things simple, we're keeping it as `multi-user.target`.

**If you want to deep-dive into the sections and if you have a more complex requirement with the service file, you can use the [man page for systemd.service](https://www.freedesktop.org/software/systemd/man/systemd.service.html) as a reference.**

### 2. Install and start the service

- After filling the necessary details for your application in the template shown above, you need to save the above file as `<your-app-name>.service` under `/lib/systemd/system/`

- Once the file is saved, you need to reload the daemon with `sudo systemctl daemon-reload` and then start the service with `sudo systemctl start <your-app-name>.service`

- (Optional) If you want to keep your service each time the system reboots, run `sudo systemctl enable <your-app-name>.service`.


## Process - The Supervisord way

Supervisord is an awesome way to write service files for your application. This enables you to write a single configuration file which is compatible across multiple OS distributions (Ubuntu, Debian, Fedora, CentOS etc.) as well as Windows! (**[although you'll need Cygwin, as described in this SO answer](https://stackoverflow.com/questions/7629813/is-there-windows-analog-to-supervisord)**)

You also get an internet socket so that you can view and control the processes on a server through a web UI. This opens up opportunities to develop custom UIs for your processes.

### 1. Install the Supervisord daemon in your OS

For Linux-based OSes, it's a straightforward process. You'll need to install supervisord with any of the commands:

```
apt-get install supervisor -y # For Ubuntu, Debian based OS

yum install -y supervisor # For CentOS, RHEL based OS

pip install supervisor # for anything else
```

### 2. Generate the initial supervisor config

For most of the installations, you'll have a default config file present in `/etc/supervisor/supervisord.conf` for starting and configuring the Supervisord service. If it's not present, you can generate the conf wtih the following:

```
mkdir -p /etc/supervisor
echo_supervisord_conf > /etc/supervisor/supervisord.conf
```

### 3. Create the config for your application

You can write the config for your application using the following template:

```
[program:<app-name>]
command=<Binary/shell-script location>
autostart=true
autorestart=true
stderr_logfile=/var/log/<app-name>.err.log
stdout_logfile=/var/log/<app-name>.out.log
```

**Note**: This particular config is only for a program that you might want to keep it running in the background. However there are multiple options available if you want to use supervisor as something else (grouping programs, FastCGI programs, event listeners etc.). More information is available from the [docs](http://supervisord.org/configuration.html)

This template has a few fields which are needed to be understood first. We'll be going over the parts mentioned:

- `command`: This is the shell script/binary that you want the service to run.
- `autostart`: This means that the program will start automatically whenever supervisord is started.
- `autorestart`: This specifies whether supervisord should restart a process if it exits when it is in the `RUNNING` state.
- `stderr_logfile`: The log file location which will be used for logging errors from the application. If the file does not exists, supervisord will create this file automatically
- `stdout_logfile`: The log file location which will be used for logging outputs from the application. If the file does not exists, supervisord will create this file automatically.

If you want to have additional capabilities such as log file rotation, custom user for starting/stopping the process, additional environment variables etc., you can take a look at the [full example documented by supervisor](http://supervisord.org/configuration.html#program-x-section-example).

### 4. Install and start the service

- After modifying the above template for your use-case, save the file at `/etc/supervisor/conf.d/<app-name>.conf`.
- Once the file is saved, reload supervisord with `sudo supervisorctl reread`
- To start the service, run `sudo supervisorctl update`

Once the above steps are completed, you can check the status of your service with `sudo supervisorctl status all`. You can control the lifecycle of a service with the following:

- `supervisorctl restart <app-name>` - Restart your application
- `supervisorctl start/stop <app-name>` - Start/stop your application


## Conclusion

With the above mentioned methods, you can now free yourself from managing any tmux/screen sessions for your application and write a service file for the same as well as manually starting applications everytime when the server is rebooted or a new server is created.