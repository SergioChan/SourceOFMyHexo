title: How to build a simple RESTful service using Github and your Mac mini
date: 2016-11-20 12:00:57
tags: [NONSENSE]
---



## Recent Work

[EarthQuick](https://earthquick.github.io/) 

![](https://ooo.0o0.ooo/2016/11/19/582fcb8440a7f.png)

## Point Of Interests

I built this service through Github repo, having not used any cloud server, cloud database or even a single domain name. And I can assume that some day, some one may write a blog which telling us how to build a fully-functional backend service through Github .. :-) It may cost some time and work on both client and server side, maybe SDK or something, while simple write operation can be wrapped into simple Git operation, and use HTTPS to transmit operation to server or something ..

Ok, back to reality, what I gonna make is just a small and simple data fetching RESTful API: you can retrieve well formatted data from various different source with different format. Well, I run a script periodly on my **useless** Mac mini to execute the Python extractor and data processor every Five minutes in order to make sure the data update in time, then through **git commit & push** operation upload the constructed data to Github. Yes, every piece of data you get from this API is pre-processed on my Mac mini, and what you get is kind of a image of the data. So this is it - render locally, access remotely. (If you are willing to do more work on client side, it's possible)

### How to use launchctl run your script periodly on macOS

- Create your Shell script **xx.sh** in your work directory **/Users/xxx**
- Change permission of your script **sudo chmod 777 xx.sh**
- There are five destinations for you `launch.plist` file :

​	**~/Library/LaunchAgents**  per-user agents provided by specific user

​	**/Library/LaunchAgents** per-user agents provided by the administrator 

​	**/Library/LaunchDeamons** Deamons provided by the administrator

​	**/System/Library/LaunchDeamons** per-user agents provided by OSX

​	**/System/Library/LaunchDeamons** Deamons provided by OSX

In which `agents` means application or script will only be running when specific user has logged in. When `agents` is provided by superior users, it means no matter user A or user B has logged in, this script will always be executed by this superior users. On the contrary, `deamons` means scripts can also be executed even when system has not been logged in. Here, the provider means you have to set the permission of the  `plist` file in these directory to support **only** the specific user group can operate. For example, if you place your `plist` file in **/Library/LaunchDeamons**, you should 

```
sudo chmod 600 sergio.chan.cron.plist
```

- Detailed description of parameters in `launchd.plist` can be found [here](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man5/launchd.plist.5.html)
- Finshed `plist` file according to the documents, for example :

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>Label</key>
	<string>sergio.chan.cron.plist</string>
    <key>ProgramArguments</key>
    <array>
  		<string>/Users/xx/xx.sh</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>StartInterval</key>
    <integer>300</integer>
    <key>StandardOutPath</key>
	<string>/../../Log.log</string>
    <key>StandardErrorPath</key>
	<string>/../../Error.err</string>
</dict>
</plist>
```

- Using `launchctl` command to control the script

```
launchctl load   /Library/LaunchDeamons/sergio.chan.cron.plist
launchctl unload /Library/LaunchDeamons/sergio.chan.cron.plist
launchctl start  /Library/LaunchDeamons/sergio.chan.cron.plist
launchctl stop   /Library/LaunchDeamons/sergio.chan.cron.plist
launchctl list
```

Notice that whenever you make any changes to the `plist` file, you should run `unload` first, then `load` again to make the changes effective. Rememer to enter **full path** of your `plist` file to guarantee the executed file is correct.

## EarthQuick on macOS

Will come out sooooon.