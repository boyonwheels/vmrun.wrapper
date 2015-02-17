WHAT IS THIS THING?
===

This script is a wrapper for VMWare Fusion's vmrun commandline tool. It was inspired by Macport's daemondo. It let's launchd start, stop, and restart VMs like any other system daemon. The wrapper script performs the following functions:

* Checks that a VM (specified by it's vmx file) is not already running.
* Starts the VM without a gui so that it can run without a logged in user account.
* Polls the VM. If it has died the script will exit so launchd will start it again.
* If the VM specified has already been started it will begin to monitor it as if it started it itself. 

Logs are written to stdout. Use launchd.plist(5)'s StandardOutPath and StandardErrorPath to log these messages to a file if you're having issues with starting or stopping a VM.

An annotated example launchd.plist is also provided.


REQUIREMENTS
---

* VMware Fusion -- vmrun should be in the system PATH.
* A basic understanding of launchd. See the references at the bottom of this document for thorough launchd documentation.

Optionally, you will also want a dedicated user and group account to run your VMs.


HOW DO I USE THIS THING?
===

Edit the example launchd.plist to be unique and put it in /Library/LaunchDaemons/ with the correct permissions (root:wheel 644). Then enable/disable and start/stop the VM with the launchctl commands you know and love to hate:

	% sudo launchctl load -w /Library/LaunchDaemons/com.odyssey.vm.hal9000.plist

This script should also work as a launchagent for a particular user's account if you prefer to have the VM start and stop with user login. Note, however, that it will cause pandemonium as-is with system level launchagents since a VM cannot be run multiple times. Currently, if you were to do it this way you would have multiple launchagents monitoring the same VM playing the greatest game of launchd Whac-a-Mole ever!


WHY USE THIS THING?
===

Short Version
---

Two reasons.

1. So you can run VMs on a headless OS X system without the gui or an account auto-login.
2. Make launchd restart a VM if it fails unexpectantly.

Long Version
---

Apple's launchd requires that processes it manages behave in a certain manner. The key requirement is that a daemon should not fork or exit:

> You must not daemonize your process. This includes calling the daemon function, calling fork followed by exec, or calling fork followed by exit. If you do, launchd thinks your process has died. Depending on your property list key settings, launchd will either keep trying to relaunch your process until it gives up (with a “respawning too fast” error message) or will be unable to restart it if it really does die.

> SOURCE: [Daemons and Services Programming Guide](https://developer.apple.com/library/mac/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html)

VMware's vmrun does exactly what launchd doesn't like: it exits when it's done. This incompatibility between vmrun's behavior and launchd requires a piece of middleware to manage the dysfunction between the two programs. This is similiar to what daemondo does for the MacPort's project. daemondo was originally written as a go-between for rc.d style unix programs that exit or fork and launchd. daemondo helps older programs work with launchd by running in the foreground. It then runs start, stop, and restart commands on behalf of launchd as well as exiting under certain conditions so that launchd can restart it.


HINTS AND TIPS
===

You can continue to use vmrun to interact with your VMs. But, you must execute vmrun with the user account that the running VM process belong's to for commands to work correctly. For example, if you run a VM as root but run the command `vmrun list` as your user account you won't see the VM in the return results:

	% vmrun list
	Total running VMs: 0
	% sudo vmrun list
	Total running VMs: 1
	/Users/Shared/Test Machine.vmwarevm/Test Machine.vmx

VMWare Fusion's VNC settings allow you to connect to the console session of a VM. This is a great way to troubleshoot VMs that are freezing in the middle of OS boot before their own remote access daemon is available. Note that the port number you connect to is on the VMware Fusion host, not the VM. By default VMware selects port 5900 but if you have Screen Sharing enabled on your Mac that port is already taken.

If launchd has issues starting a VM it will log the issue to syslog and record the exit code. You can see the exit code with `launchctl list |grep $name_of_plist`. 

% sudo launchctl list |grep omni                                                                                                         
16432	2	com.omnigroup.vm.test

The script logs to stdout. Use launchd.plist(5)'s StandardOutPath and StandardErrorPath to log these messages if you're having issues with starting or stopping a VM.

The script originally parsed process information from the filesystem. VMware Fusion creates symbolic links in /var/run/vmware/ for every running virtual machine. The last part of the symbolic link's pathname has the PID in it. So, for example this VM's PID is 64956:

/var/run/vmware/c06b2ef271c7ae3b3c55a87a3b105c42@ -> /var/run/vmware/root_0/1423866455738667_64956

The path also tells you the username_uid (root_0) of the account that owns the running vm and the epoch time that the machine was started (1423866455738667)


KNOWN ISSUES
===

* None at this time.


TODO
===

* Improve the Check_PID() function. It's going to burn someone at some point. I know it!

REFERENCES AND DOCUMENTATION
===

* [Daemons and Services Programming Guide](https://developer.apple.com/library/mac/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/Introduction.html#//apple_ref/doc/uid/10000172i-SW1-SW1)
* [MacPorts StartupItems](https://guide.macports.org/chunked/reference.startupitems.html)
* [More Explanation in the Guide for StartupItems](https://trac.macports.org/ticket/16930)
* [Daemondo Source Code](https://trac.macports.org/browser/trunk/base/src/programs/daemondo/main.c)


SEE ALSO
===

Joseph Chilcote (@chilcote) has built a nice python tool called [vserv](https://github.com/chilcote/vserv) also based on the ideas of this shell script. His tool is a little more comprehensive then this shell script. You should take a look at it, too! 
