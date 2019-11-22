## nweventwatcher

A macOS utility that watches for changes to the system's network environment and executes user defined commands based on the changes. It can also detect and react to wake-from-sleep events and changs between battery and AC power.

Tested on macOS Mojave and Catalina. May work on earlier versions.

## Documentation
Please see the [User Guide](./UserGuide.md).

## License

You may not use the identified files except in compliance with the Universal Permissive License, Version 1.0 (the "License.")

You may obtain a copy of the License at [https://oss.oracle.com/licenses/upl](https://oss.oracle.com/licenses/upl).  A copy of the license is also reproduced in [LICENSE.md](./LICENSE.md) and [LICENSE.txt](./LICENSE.txt).

## Contact

You may contact the author at chris.jenkins1960@icloud.com 

## Scripts

The package consists of the following scripts:

| Script | Description  |
| :----- | :----------- |
| [new](./new) | The NetWork Event Watcher main script. |
| [getmyip](./getmyip) | Determines the system's IP address, in various flavours. |
| [getwifinetworks](./getwifinetworks) | Determines the names (SSIDs) of any connected WiFi networks. |
| [getnwinterfaces](./getnwinterfaces) | Determines the names of the systems network interfaces, in various flavours. |

## Dependencies

Network Event Watcher requires the following third-party components:

- The 'terminal-notifier' utility which is used to send notifications
  ([https://github.com/julienXX/terminal-notifier](https://github.com/julienXX/terminal-notifier)). An easy way to
  install this is using **Homebrew** ([https://brew.sh](https://brew.sh)):

  **brew install terminal-notifier**

  If 'terminal-notifier' is not installed then the notification
  feature will not work.

- The 'sleepwatcher' utility which is used to detect system wake
  and power events ([https://www.bernhard-baehr.de](https://www.bernhard-baehr.de)). An easy way to
  install this is using **Homebrew** ([https://brew.sh](https://brew.sh)):

  **brew install sleepwatcher**

  You do NOT need to configure 'sleepwatcher' to run automatically.

  If 'sleepwatcher' is not installed then:

  - A less accurate, higher overhead mechanism will be used to detect
    system sleep/wake events and this may result in occasional missed
    events or false positives.

  - Detection of power source changes is not enabled so the '@battery'
    environment will not trigger any events.

