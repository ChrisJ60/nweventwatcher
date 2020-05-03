**NETWORK EVENT WATCHER**

Watch for, and react to, changes of network environment and system wake events.

This utility monitors the system for changes in the network environment, such
as:

-  Network interfaces appearing or disappearing
-  Network interfaces becoming active or inactive
-  Network interface IP addresses changing (IPv4 or IPv6)
-  VPN connections connecting or disconnecting

It also monitors for system wake events (i.e. the system wakes from sleep)
and changes of power source (battery or AC).

When one of the monitored events occurs, the utility can perform a user defined
action to react to the event.

**DEPENDENCIES**

This utility uses:

1.   A 'getNwEnv' script which determines the currently active network
     environment(s) and defines the set of supported environments. This
     script must be provided by the user and must comply with the
     specification (see below).

2.   The 'terminal-notifier' utility which is used to send notifications
     [https://github.com/julienXX/terminal-notifier](https://github.com/julienXX/terminal-notifier). An easy way to
     install this is using HomeBrew [https://brew.sh](https://brew.sh):

     brew install terminal-notifier

     If 'terminal-notifier' is not installed then the notification
     feature will not work.

3.   The 'sleepwatcher' utility which is used to detect system wake
     and power events [https://www.bernhard-baehr.de](https://www.bernhard-baehr.de). An easy way to
     install this is using HomeBrew [https://brew.sh](https://brew.sh):

     brew install sleepwatcher

     You do NOT need to configure 'sleepwatcher' to run automatically.

     If 'sleepwatcher' is not installed then:

     - A less accurate, higher overhead mechanism will be used to detect
       system sleep/wake events and this may result in occasional missed
       events or false positives.

     - Detection of power source changes is not enabled so the '@battery'
       environment will not trigger any events.

**INSTALLATION**

1.  Make sure that your PATH setting includes both /usr/local/bin and
    /usr/local/sbin and that the system is set to include these in the
    default PATH (appropriate files in /etc/paths.d).

2.  Copy the main script to /usr/local/sbin (recommended) or /usr/local/bin.
    The script's default name is 'new' (Network Event Watcher) but you
    can give it any name you like as long as it does not conflict with any
    other executables on your system.

3.  Create (and test) your customised 'getNwEnv' script (see specification
    below) and put that in /usr/local/bin or /usr/local/sbin.

4.  Install the 'terminal-notifier' utility. If you do not install this
    utility then the notification feature will not work but everything
    else will still function correctly.

5.  Install the 'sleepwatcher' utility. If you do not install this
    utility then the '@awake' environment feature will still work but
    it will use a less reliable mechanism to detect system wakes which
    may occasionally result in missed events or false positives.

**CONFIGURATION, STORAGE AND RESTRICTIONS**

The utility stores its configuration and a few other small files in
**~/Library/Application Support/NwEventWatcher** hence all aspects of
configuration and operation are specific to an individual user.
Different users can have different configurations but only one
user's monitor/agent should be running at any one time.

**CONCEPTS**

_Environment_

A specific network environment. The list of supported environments
is that returned by executing **getNwEnv list**. In addition to the
environments determined by that command, the following special
environments are also supported:

**Unknown**  - Matches any environment not specifically recognised
           by 'getNwEnv'.

**@default** - Used to set default actions (see below).

**@awake**   - Used to set an action for system wake events.

**@battery** - System is running on battery power.

**@all**     - Matches all defined environments, including 'Unknown',
          '@default', '@battery' and @awake.

_Event_

An event occurs when a network environment transitions to active (from
being inactive) or vice versa. The current set of active environments
is determined by executing the command **getNwEnv show -all**. Each
individual environment (including **Unknown**) has its own **A** (became
active) and **I** (became inactive) events. Since multiple environments
may be active concurrently this means that in any checking cycle
multiple events may be triggered.

The non-network environment **@awake** corresponds to the system
being awake. Only **A** events are supported for this environment;
i.e. the system changed from being asleep to being awake.

The non-network environment **@battery** corresponds to the
system running on battery power (as opposed to AC power).
The **A** event for this environment occurs when the system switches
from AC power to battery and the **I** event is the reverse of this.

_Action_

A command associated with an event. When the event is triggered the
associated action, if any, is executed.

**EVENT DETECTION MECHANISM**

The monitor task is responsible for detecting and reacting to events
based on the configuration. The monitor can be run directly, either as
a foreground activity, as a background task (preferred) or as a
LaunchAgent (even more preferred). Event detection uses an optimised
algorithm that combines polling with event notification to ensure
timely event detection with minimal system overhead.

Network change events are normally detected using an agent process that
hooks into the macOS System Configuration Framework. This agent gets
notified by macOS whenever any interesting network environment change
occurs and it in turn notifies the monitor task. This is an efficient
mechanism as it avoids the need to frequently poll for changes to the
network environment. If for some reason the SCF agent cannot function
then the monitor falls back to a polling based mechanism.

System wake and power events are normally detected using an agent
process that hooks into the macOS Power Notification mechanism.
This process is the **sleepwatcher** utility which must be installed
separately.

Using **sleepwatcher** is an efficient mechanism as it avoids the need to
frequently check to see if the system has woken from sleep. It is
also more reliable than the polling based mechanism. If **sleepwatcher**
is not available, or cannot run, then the monitor falls back to the
polling mechanism.

When operating in polling mode the monitor regularly checks for changes
to the network environment and for system wake events. The frequency
of these checks is determined by the **-interval** option (see below).
The default interval is 10 seconds and this should be adequate
for most situations.

**RESPONSIVENESS VERSUS RESOURCE USAGE**

Under typical usage conditions the monitor task and associated helper
tasks use very little CPU (roughly 0.5% of a single core on average).
There is a tuneable parameter, the sleep quantum, which can be varied to
reduce the (already low) resource usage at the cost of slightly reduced
responsiveness.

The monitor detects when the system transitions between AC power and battery
power and automatically adapts. When running on AC power it uses a sleep
quantum of 1 second for maximum responsiveness with low overhead, and when
running on battery power it increases the sleep quantum to 3 seconds which
significantly reduces CPU usage (and hence increases battery life) for
only a modest reduction in responsiveness. You can tune the values used
in both modes by using the '-acsq' and '-btsq' parameters of the 'config'
command. Values range between 1 and 5.

**ACTION EXECUTION AND ORDERING**

It is possible that multiple events may occur at the same time (or so close
together that they effectively occur at the same time). In such cases there
is a clearly defined order of execution for their associated actions.

-   Events are triggered for all environments that have transitioned
    from active to inactive (I), then events are triggered for all the
    environments that have transitioned from inactive to active (A).

-   Events are triggered in the order in which the environment names
    are returned by the **getNwEnv list** command. Events for the **@awake**
    environment are always triggered first, followed by events for the
    **@battery** environment and then events for recognised network
    environments. Events for the **Unknown** environment are always
    triggered last.

-   If there are no defined actions for an event then the event is ignored (it     is still logged); you do not need to define actions for every possible event.     You can also create actions for the special environment named **@default**.     These actions will then trigger for any event which does not have its own     defined actions, except for the **@awake** event.

Actions execute synchronously to the monitor. While an action is executing
the monitor will not detect any new events, but in general new events which
occur during action execution will be queued and will be detected/processed
after the current set of actions have finished processing.

**ACTION COMMAND EXECUTION ENVIRONMENT**

Action commands are executed as background tasks. They should make no
assumptions about their environment other than as follows:

- **PATH** will be the system default PATH plus whatever may be defined
  in /etc/paths.d

- **USER** will be the username under which the monitor is running

- **HOME** will be the home directory of **$USER**

Standard input is always redirected to /dev/null. If there is an action
log file defined then standard output and standard error will be redirected
to the log file otherwise they will be redirected to /dev/null. There is
no controlling terminal.

**ACTION COMMANDS**

When assigning commands to actions, or creating scripts to be used
as action commands, bear in mind the following:

-  Action commands run synchronously with the monitor task and so
   should not directly invoke long running operations. Ideally
   an action command should complete its processing within a few
   seconds. If you need to start a long running operation
   from an action command you should ensure that it is started
   as an asynchronous task (for example run it in the background)
   and the action command itself completes quickly.

-  Action commands can be passed two special parameters which they
   can inspect to determine the event that triggered the action.
   See the 'set' command below for details.

-  Action commands should avoid relying on any specific ordering of
   execution and dependencies on other action commands.

-  Generally A events are more useful than I events since they are
   more definitive. Nonetheless I events can sometimes be useful.

**USAGE**

  new help [usage | general | getnwenv | full | <cmd>]...

  new version

  new list {environments [-verbose] | actions [-script] | active}

  new set environment {a | i | b} [-silent] *command* [*param*]...

  new set environment {a | i | b} -notify

  new clear environment {a | i | b} [-verbose]

  new config [-clear | -script | [-warntimeout *w*] [-killtimeout *k*]
             [-actionlog *path*] [-acsq *a*] [-btsq *b*]
             [-maxlogs *l*]]
             
  new install [*agenturl*] [-settle *s*] [-interval *i*] [-notify]

  new uninstall

  new init
  
  

  new refresh [-force]
  
  new rescan

  new status [-verbose] [-asleep]

  new start

  new stop
  
  new pause [*interval*] [-norescan]
  
  new resume

  new cleanup [-verbose]

  new log [-action] [-follow | -clear | -fno *n*]

  new log {-rotate | -info}

  new monitor [-settle *s*] [-interval *i*] [-notify] [-fg]

**help [usage | general | getnwenv | full | *cmd*]...**

Displays help information for the selected topics. If no topic is
specified then displays this help. Topics are:

usage     -  Usage and syntax.

general   -  General information including overview, concepts, instalation,
            dependencies etc.

getnwenv  -  Detailed information on the 'getNwEnv' command that the
             user must provide.

full      -  Full help covering all topics.

*cmd*     -  Detailed help for the specified command.

**version**

Displays version information.
    
**list {environments [-verbose] | actions [-script] | active}**

Displays a list of environment names, defined actions or the currently
active environments (except '@awake'). For actions, if '-script' is
specified the action list is output as a list of commands that will
exactly re-create the existing setup.

For actions, the 'list' command displays the current state of the on disk
configuration which may not be the same as the in-memory copy currently
being used by a running monitor/agent. See 'set', 'clear' and 'refresh'
below.

**set environment {a | i | b} [-silent] *command* [*param*]...**

Creates an action; sets a command to be executed when an event is triggered
for the named environment. Events are:

-  environment becomes active (a)

-  environment becomes inactive (i)

-  either (b)

Note that 'b' is not a separate event; it is simply a shorthand way of
creating an action for both 'a' and 'i' events. Only 'a' events may be
defined for the '@awake' environment.

'*command*' must be an executable file (binary, script, ...) and may be
followed by optional parameters as required. The following special
placeholders will be substituted for the corresponding values when the
command is executed:

  **#E#**- The environment name for the event.

  **#A#** - The new environment state for the event; A = active, I = inactive.

An absolute path should always be used when specifying the command.

If macOS notifications are enabled then a notification will be sent
whenever an event with an associated action is triggered occurs unless
**-silent** was specified when the action was created.

**set environment {a | i | b} -notify**

Sets a 'notify only' action for an event. No command is executed but a
notification is sent if macOS notifications are enabled.

**NOTE:**
    If a monitor or agent is running, changes made with the **set** command will
    not take effect until a **refresh** command has been issued.

**clear environment {a | i | b} [-verbose]**

Removes the specified action(s) for the named environment/event(s).

**NOTE:**
    If a monitor or agent is running, changes made with the **clear** command will
    not take effect until a **refresh** command has been issued.

**config [-clear | -script | [-warntimeout *w*] [-killtimeout *k*] [-actionlog *path*]
       [-acsq *a*] [-btsq *b*] [-maxlogs *l*] [-logrotate *d*]]**

Displays/sets values for the command execution warning timer
(-warntimeout), the command execution kill timer (-killtimeout),
the action log file (-actionlog), the maximum number of retained
log files (-maxlogs), the log file rotate interval in days
(-logrotate), the AC sleep quantum (-acsq) and the battery sleep
quantum (-btsq) or clears all values previously set (-clear).
Individual values can be cleared by specifying a value of '-'.
Any value that does not have an explicitly defined value will use
its default value (if any).

If '-script' is specified, a set of commands are output that will
re-create the current configuration.

Timeout values must be between 0 and 180; a value of 0 for a timeout
means 'no timeout'. Default values are 30 for warnings and 60 for the
kill timeout.

If the warning timeout is > 0 and the execution of any action command
takes longer than the timeout value then a warning message is logged.
If the kill timeout is > 0 and the execution of any action command
takes longer than the timeout value then the command task is killed and
an error message is logged.

If an action log file is defined then additional information about
action execution will be logged there and the standard output and
standard error of action commands will be redirected to the file.
You can disable an existing action log by passing an empty string
for the path (''); the log file is not deleted.

Values for sleep quanta are specified in seconds and must be between
1 and 5.

The 'maxlogs' value specifies the maximum number of log files to retain
when using the log file rotation feature. The value must be between 1
and 10. The default is 5.

The 'logrotate' value specifies how often, in days, a running monitor
should perform an automatic 'log -rotate' operation. The value must be between
0 and 365. The default is 7. If this parameter is set to 0 then automatic
log rotation is not performed.


**NOTE:**
    If a monitor or agent is running, changes made with the 'config' command
    will not take effect until a 'refresh' command has been issued.
    
**install [*agenturl*] [-settle *s*] [-interval *i*] [-notify]**

Installs a user LaunchAgent so that the monitor runs as an agent whenever
you are logged in. The agent must not be currently loaded in order to use
this command. This is the preferred way to use this utility.

The options are:

  **agenturl**
  
  The URL to use for the launch agent. The default is 'com.jenkinsnet.netwatcher'.

  **-settle s**
  
  Wait for 's' seconds before starting to monitor for changes after
  starting or waking from sleep.
  0 <= s <= 300. The default is 15.

  **-interval i**
  
  Check for events every 'i' seconds when using 'polling' mode.
  5 <= i <= 60. The default is 10.

  **-notify**
  
  Enables the macOS notification feature. Requires the
  **terminal-notifier** utility to be installed.

**uninstall**

Uninstalls the user LaunchAgent so that the monitor no longer runs
automatically. The agent must not be loaded in order to use this
command.

**init**

Sets up the configuration from scratch; clears all defined config parameters and
actions and regenerates the list of environments using 'getNwEnv list'.
Any running monitor task or agent must be stopped before using this command.
It is not usually necessary to use this command as an implicit 'init' is
performed the very first time a user runs this utility.

**refresh [-force]**

Tells a running monitor/agent to reload its configuration to pick up any
pending changes. Normally the refresh is only performed if there are pending
changes but you can force it to occur by specifying '-force'. Starting the
monitor/agent using the 'start' or 'monitor' commands performs an implicit
refresh.

**rescan**

Tells a running monitor/agent to rescan the current set of active network
environments and execute any defined A actions for all currently active
environments. This is the same processing that occurs when a monitor or
agent starts up.


**status [-verbose] [-asleep]**

Displays the status of the monitor/agent. If '-verbose' is specified then
status is also displayed for sleepwatcher, netwatcher and pending refresh.

If '-asleep' is specified then sets the exit code as follows:

0   -  An agent or monitor is running, sleepwatcher is running and the
      system is currently asleep (i.e. in a 'dark wake').

1   -  An agent or monitor is running, sleepwatcher is running and the
      system is not currently asleep.

\>1 -  Any of the above conditions are not met, an error occurred, etc.

Normally using '-asleep' does not generate any output messages but you
can get some informative messages by also specifying '-verbose'

**start**

Starts the agent if it is configured and not already running.

**stop**

Stops the monitor/agent if it is running.

**pause [*interval*] [-norescan]**

Pauses the monitor/agent for the specified time interval or forever if
no interval is specified. The interval may be specified as an integer
(number of seconds) or in the format HH:MM:SS. If specified, the value
must be between 15 and 86400 (00:00:15 and 24:00:00).

While paused the monitor/agent will not detect, handle or log any events.

The monitor/agent can be resumed early using the 'resume' command. Stopping
the monitor/agent clears any existing pause state.

Normally a rescan operation is performed when the monitor/agent
resumes but this can be suppressed using '-norescan'.

**resume [-rescan | -norescan]**

Resumes a currently paused monitor/agent. Normally a rescan will be
performed after the monitor/agent resumes but this can be suppressed
by specifying '-norescan' or forced (if it was suppressed at pause
time) by specifying '-rescan'.

**cleanup**

Cleans up any residual helper processes. You should not normally need to
use this option.

**log [-action] [-follow | -clear | -fno *n*]**

Display the current log file (or archived file <n> if '-fno' was
specified) or clears all log files (-clear). Acts on the monitor log
unless '-action' is specified when it acts on the action log, if configured.
Using '-follow' displays new log entries as they are generated; interrupt
the display using Ctrl-C.

**log  {-rotate | -info}**

Provides information about the current log files (-info) or rotates the
log files (-rotate).

Rotating the log renames the current log as <logname>.1 and so on up to a
maximum of the configured value for 'maxlogs' set using the 'config' option.
Logs exceeding 'maxlogs' are removed. This operation applies to both the
monitor log and the action log, if configured. Manually rotating the logs
also updates the timestamp used by the automatic rotation mechanism.

The -rotate command signals the running monitor/agent which then rotates
the log(s), hence this command can only be used when a monitor or agent
is running.

**monitor**

Runs the monitor directly; useful for special purposes.

The options are:

  **-settle s**
  
  Wait for 's' seconds before starting to monitor for changes after
  starting or waking from sleep.
  0 <= s <= 300. The default is 15.

  **-interval i**
  
  Check for events every 'i' seconds.
  5 <= i <= 60. The default is 10.

  **-notify**
  
  Enables the macOS notification feature. Requires the
  'terminal-notifier' utility to be installed.

  **-fg**
  
  Normally the monitor process is run as a separate background task.
  Specifying '-fg' causes it to be run in the foreground (in the
  invoking task).

**THE GETNWENV SCRIPT**

_Overview_

The user is responsible for providing a command (most likely a script) called
'**getNwEnv**' which must be installed in either /usr/local/bin or /usr/local/sbin.

This script is responsible for:

-  Defining the names and descriptions of the set of supported network
   environments.
   
   The definition of what constitutes an 'environment' is left to the user but
   typically it might be things like 'Connected to my Home WiFi', 'Connected to
   my Work VPN', 'Tethered to my iPhone' and so on. 

-  Determining the set of currently active network environments.

   The definition of Active and Inactive is deliberately left to the user
   but typically Active means connected/usable and Inactive means the opposite.
   A good example is a VPN connection; when it is not connected it is Inactive
   and when it is connected it becomes Active. The same example holds true for
   WiFi and wired network connections. Multiple environments may be active
   concurrently.

_Syntax and specification_

The syntax for invoking the script is as follows. This is the minimum syntax;
the script may support additional functionality over and above what is specified
here.

getNwEnv list [-verbose]

Display a list, one per line with no leading or trailing spaces, of all the
network environments that are supported. An environment name may consist of
alphanumeric characters and some special characters such as '-_@+-='. Spaces
are not permitted within an environment name. The following environment
names are reserved and should NOT be returned by this script; Unknown, @all,
@awake and @default. Here is an example of what the output might look like:

    $ getNwEnv list
    Home-LAN
    Home-WiFi
    Work-LAN
    Work-WiFi
    Work-VPN

The order in which the environments are displayed is significant to the
network event watcher utility as it affects the order in which events are
triggered.

If the '-verbose' option is specified then each network name should be
followed by a colon (:) and a textual description of the environment.
For example:

    $ getNwEnv list -verbose
    Home-LAN:Wired connection to the home network
    Home-WiFi:WiFi connection to the home network
    Work-LAN:Wired connection to the office network
    Work-WiFi:WiFi connection to the office network
    Work-VPN:VPN connection to work

The order in which the environments are displayed must be the same as
when the '-verbose' option is not used.

getNwEnv show -all

Display a list, one per line with no leading or trailing spaces, of the
currently active network environments. For example if you are currently
connected to your home WiFi but also using your VPN to access your work
network then the output would look like this:

    $ getNwEnv show -all
    Work-VPN
    Home-WiFi

The order in which the active environments are displayed is not significant.

_Exit codes_

The 'list' command should always return a zero exit code unless it is unable to
successfully generate the list, in which case the exit code should be > 0.

The 'show' command should return 0 if at least one of the supported environments
is active. If none of the supported environments is active it should not return
any textual output and the exit code should be 1. An exit code > 1 indicates
that an error occurred and the command was not able to determine the environment
status.

