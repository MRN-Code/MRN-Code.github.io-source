title: 'Launching a Console in VMware Web Client on Windows 10, Chrome 42+'
date: 2015-07-04 11:26:21
tags:
  - VMware
  - vSphere
  - Chrome
authorId: dwood
---
Maybe it is just me, but I had a difficult time launching the console of my
guest VMs using the VMware Web Client. Here is how I eventually got it working
on Windows 10 and Chrome 43.x.

{% asset_img launch.png Launch VMWare Web Client %}

## Background

I am a huge fan of VMware's plan to replace their Windows-only vSphere desktop
client with a web client. Using the web client, I am able to perform
*most* tasks directly from my Mac, thus saving the time of booting a Windows VM.
The only task which cannot be performed in the web client from OS-X is launching
the guest OS console.

In order to open the console in the web client, it is necessary to install the
*VMware Client Integration Plugin*, which VMWare claims is only available for
64/32-bit Windows and 32-bit Linux. I was unable to get the Client Integration
Plugin to install on Ubuntu 14.04, so it looks like I am still stuck using a
Windows VM to manage our VMware cluster.

Even on Windows, it took some time to get things configured such that I could
access a guest VM's console via the web client. Here is how I eventually did it.

## Environment
* OS: Windows 10 Evaluation Copy
* Browser: Google Chrome 43.0.2357.130 (Tried IE Edge and Firefox with no luck)

## Install Client Integration Plugin
1. Navigate to your VMware Web Client login page in your browser. **Do not log in.**
1. Click the link at the bottom left of the page entitled 'Download Client Integration Plugin'.
1. Run the downloaded installer, accepting all defaults.

## Enable NPAPI plugins:
1. Paste `chrome://flags/#enable-npapi` into your address bar and press return.
1. Click `Enable` below *Enable NPAPI*.
1. **Click `Relaunch Now` at the bottom left of the page**.

## Allow VMware plugins to run
1. Paste `chrome://plugins/` into the address bar and press return.
1. Check the box next to 'Always allow to run' below **both** VMware plugins.

## Verify plugins
1. Restart the windows machine for good measure.
1. Open Chrome and navigate back to your VMware Web Client login page.
You should see two notifications from chrome at the top of the page (see image below).
These notifications can be disregarded (for now, see discussion further below).
{% asset_img plugin_warnings.png %}

## Still does not work?
If you do not see the warnings from Chrome, try this:
1. Navigate to chrome://settings/content
1. Scroll to 'Unsandboxed Plugins'
1. Select 'Allow all sites ...'
1. Click 'Done'
1. Repeat the *Verify plugins* steps above.

## Open the console for a VM
1. Log in to the VMware Web Client
1. Locate a VM whos console you want to open
1. Click the `Settings` tab
1. Near the top of the center pane, you should see a black square with the text `Launch Console` beneath it. If you see a link to `Download plugin` instead, something
is wrong. Try repeating the steps above.

## Discussion about NPAPI plugin support
Google has promised to completely remove NPAPI plugin support from Chrome with
version 45. Given the approximate 5-week release schedule that Google has been on,
this means that you will only be able to use the most recent version of Chrome
with the VMware Client Integration Plugin for another couple of months.

With this in mind, I am going to keep my vSphere desktop application installed.
Hopefully, VMware has already begun work on a truly cross platform Web Client
that supports launching a console.
