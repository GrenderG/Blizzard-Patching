# Blizzard-Patching

## Overview

When a player tries to login to the server, the server reads the client version and detects it is out of date and that there is a update available. The server sends a update to the client as part of the login protocol. The update contains a Blizzard Downloader executable. The Blizzard Downloader downloads a Blizzard Updater. The Blizzard Updater installs the MPQ updates and a new WoW executable with a updated build. Now when the player logs in they have the correct version and can play.

## Patching through the WoW client

https://www.dropbox.com/s/fovh9mtrj9tgqd4/Implementing%20in-client%20patching%20for%20World%20of%20Warcraft.pdf?dl=0

## The Blizzard Downloader

The Blizzard Downloader is responsible for downloading the Blizzard Updater. It supports HTTP direct downloads and the bittorrent protocol. We can send the entire update through the WoW client instead of using the Blizzard Downloader but this will stress the authserver and doing it that way loses the peer to peer aspect.

The Blizzard Downloader can be built with the resources under the following repository: https://github.com/stoneharry/Blizzard-Downloader

The downloader can be built using `Bwod.exe`, for example: `.\BwoD.exe compile torrent.torrent MyBackgroundDownloader.exe`. The program compiles the downloader using the `base.exe`, embedding the `skin.mpq`, and embedding the torrent data to use.

The `skin.mpq` file contains the images that are used to style the interface.

The torrent file is the file that the downloader should download and seed.

The torrent file can be created using any bittorrent program. Take note of the split size used, for me I set the split size to 16384 kb. Once you have created your torrent file you can use `BEncode Editor.exe` to edit the data in the torrent file. We can add various properties to the file that the Blizzard Downloader reads to know how to behave.

![Bencode screenshot](https://i.imgur.com/I6tO3Bw.png)

A full list of the possible parameters that I have reverse engineered:
```
download type
url
end
begin
server list
disable p2p
locale
notes url
direct download
choose download
show eula
done label
download label
autolaunch
launch target
creation date
announce
```

To support HTTP direct downloads we need to split the data for the torrent by the file size we defined. I set the size to 16384 kb, so I use the `filesplitter.exe` to split the installer by 16777216 bytes. I can then put this on my HTTP host and use the link for Direct HTTP downloads.

![Example HTTP Direct Download](https://i.imgur.com/vlT4V1V.png)

You need a tracker for the Blizzard Downloader to connect to and use to work out who is seeding and what direct downloads are available. I have tested using a C# library MonoTorrent and also using a very lightweight PHP framework: https://github.com/Hlkz/UpdateTracker

![Example Blizzard Downloader](https://i.imgur.com/z9POmpJ.png)

This works perfectly using HTTP direct downloads. However I have not managed to get the peer-to-peer aspect to work. When debugging my torrent tracker I can see that seeders and leachers are correctly registering with the tracker, so I am not sure the leachers do not start downloading data from the seeders.

When the Blizzard Downloader is run from the WoW directory it creates debug log output at the following directory: `C:\Users\Public\Documents\Blizzard Entertainment\World of Warcraft\Logs`

From my reverse engineering I believe it reads a `BlizzardDownloader.ini` from `C:\Program Files (x86)\Common Files\Blizzard Entertainment`.

Throughout the assembly I can see useful debug log statements made when a "test mode" is enabled. I have not managed to find a way to enable this mode.

Debug mode data:

![Reverse Engineer 1](https://i.imgur.com/ReUPTd5.png)
![Reverse Engineer 2](https://i.imgur.com/Cf1PqSb.png)

We can also see where it appears to read in a bunch of parameters from somewhere:

![Reverse Engineer 3](https://i.imgur.com/d6OJSY1.png)
```
directDownloadURL
nop2p
nohttp
noseeds
onlyblizzpeers
noblizzpeers
trackerless
allowincoming
testmode
```

Considering the Direct Downloader URL is sent from the tracker, I wonder if the other parameters need to be supplied that way somehow:
```php
		// begin response
		$response = 'd8:intervali' . $_SERVER['tracker']['announce_interval'] . 
		$response = strlen($direct_download[0]) ? 'd6:directd3:url' . strlen($direct_download[0]) . ':' . $direct_download[0] . '9:thresholdi10000000ee' : 'd';
		$response .= '8:intervali' . $_SERVER['tracker']['announce_interval'] . 
		            'e12:min intervali' . $_SERVER['tracker']['min_interval'] . 
		            'e5:peers';

```

I have tested trying to set test mode through various properties in the torrent file and have also tried setting it in the `BlizzardDownloader.ini` file it tries to read. Another thing I have noted is that in the logs we can see it trying to connect two hardcoded IP's for the ini file:
`17:55:45.0571 Checking for server side config http://206.16.22.130/update/Downloader.ini`
`15:57:40.6400 Checking for server side config http://12.129.232.131/update/Downloader.ini`
I could not find in the binary where it is defining this IP to try to connect to. It appears to use one of the two each run at random, and doesn't try connecting to both in a single run.

## The Blizzard Updater

https://github.com/stoneharry/Blizzard-Updater
