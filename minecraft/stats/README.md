# MinecraftStats

_MinecraftStats_ is a web browser application for the [statistics][1] that Minecraft servers collect about players.

The presentation is done by giving __awards__ to players for certain achievements. For example, the player who played on the server for the longest total time receives the _Addict_ award. Every award has a viewable ranking associated to it with __medals__ - the award holder gets the gold medal, the second the silver medal and the third the bronze medal for the award. Each medal gives players a __crown score__ (1 for every bronze medal, 2 for every silver, 4 for every gold medal), which is displayed in a server __hall of fame__.

The system is highly customizable. All the awards are defined in Python modules that can be altered, added or removed to fit your needs. Additionally to simply reading Minecraft's original statistics, there are some awards that are combinations of various statistics.

A live demo of _MinecraftStats_ in action is available here: [DVG Snapshot Stats][2]

## Setup Guide
This section describes how to set up _MinecraftStats_ to work on your server.

### Compatibility
Since version 2.0, ___MinecraftStats_ is compatible only to Minecraft 1.13 or later__ (more precisely: snapshot [17w47a][5] or later). This is because starting with snapshot 17w47a, Minecraft stores statistics very differently from before. In the process of updating _MinecraftStats_ for 1.13, the complete infrastructure has been rewritten and is in no way compatible to older servers.

If your server runs Minecraft 1.12.2 or prior, please use [_MinecraftStats 1.0_][3] and do not follow this guide any further, but use the old guide.

I am trying to keep the statistics up to date with Minecraft snapshots. Since Mojang often decide to rename entity IDs, this may mean that compatibility to a prior release version of Minecraft breaks. If you notice such a case, please don't hesitate to [open an issue][4] about that!

#### Migrating
If you used _MinecraftStats_ 1.0 before, the only way to migrate is delete it and use this new version instead.

Any customizations (awards or other extensions) won't be compatible and need to be ported manually, because _MinecraftStats_ 2.0 is a complete rewrite, much like the way Mojang apparently rewrote their statistics completely (more structured, but quite some stats have vanished).

### Requirements
_Python 3.4_ or later is required to feed _MinecraftStats_ with your server's data.

### Installation
Simply copy all files (or check out this repository) somewhere under your webserver's document root (e.g. `htdocs/MinecraftStats`).

The web application is simply `index.html` - so you'll simply need to point your players to the URL corresponding to the installation path.

### Feeding the data
The heart of _MinecraftStats_ is the `update.py` script, which compiles the Minecraft server's statistics into a database that is used by the web application.

Simply change into the installation directory and pass the path to your Minecraft server to the update script like so:
```python3 update.py -s /path/to/server```

You may encounter the following messages during the update:
* `updating profile for <player> ...` - the updater needs to download the player's skin URL every so often using Mojang's web API ,so that the browser won't have to do it later and slow the web application down. __If this fails__, make sure that Python is able to open `https` connections to `sessionserver.mojang.com`.
* `unsupported data version 0 for <player>` - this means that `<player>` has not logged into your server for a while and his data format is still from Minecraft 1.12.2 or earlier.

In case you encounter any error messages and can't find an explanation, don't hesistate to [open an issue][4].

After the update, you will have a `data` directory that contains everything the web application needs. Fore more information about what it contains exactly, see below.

#### Automatic updates
_MinecraftStats_ does not include any means for automatic updates - you need to take care of this yourself. The most common way to do it on Linux servers is by creating a cronjob that starts the update script regularly, e.g., every 10 minutes.

If you're using Windows to run your server... shame on you, figure something out!

#### FTP
In case you use FTP to transfer the JSON files to another machine before updating, please note that _MinecraftStats_ uses a JSON file's last modified date in order to determine a player's last play time. Therefore, in order for it to function correctly, the last modified timestamps of the files need to be retained.

#### Options

The `update.py` script accepts the following command-line options (and some more unimportant ones, try passing `--help`):

* `-s <server>` - the path to your Minecraft server. This is the only __required__ option.
* `-w <world>` - if your server's main world (the one that contains the `stats` directory) is not named "world", pass its alternate name here.
* `-d <path>` - where to store the database ("data" per default). Note that the web application will only work if the database is in a directory called `data` next to `index.html`. You should not need this option unless you don't run the updater from within the web directory.
* `--server-name <name>` - specify the server's name displayed in the web app's heading. Minecraft color codes are supported! By default, the updater will read your `server.properties` file and use the `motd` setting, i.e., the same name that players see in the game's server browser.
* `--inactive-days <days>` - if a player does not join the server for more than `<days>` days (default: 7), then he is no longer eligible for any awards.
* `--min-playtime <minutes>` - only players who have played at least `<minutes>` minues (default: 0) on the server are eligible for awards.

#### Database structure
The `data` directory will contain the following:
* `db.json.gz` - This is the database index used by the web application and for future updates. It is a simple JSON file compressed using `gzip` to reduce traffic by the web application. If you also need the uncompressed file for any reason, pass the `--store-uncompressed` option to the updater.
* `server-icon.png` - if your server has an icon, this will be a copy of it. Otherwise, a default icon will be used.
* `rankings` - this directory contains one JSON file for every award. These are dynamically loaded by the web application as needed.
* `playerdata` - this directory contains one JSON file for every player. These are dynamically loaded by the web application as needed.

# Customizing Awards
I assume here that you have some very basic knowledge of Python, however, you may also get away without any.

`update.py` imports all modules from the `mcstats/stats` directory. Here you will find many `.py` files that define the awards in a pretty straightforward way.

The next time `update.py` is executed, changes here will be in effect.

## Adding new awards
For adding a new award, follow these steps:

1. Create a new module in `mcstats/stats` and register your `MinecraftStat` instances (see below).
2. Place an icon for the award in `img/award-icons`. If your award ID is `my_award`, the icon's file name needs to be `my_award.png`.

### The MinecraftStat class
A `MinecraftStat` object consists of an _ID_, some _meta information_ (_title_, _description_ and a _unit_ for statistic values) and a _StatReader_.

The _ID_ is simply the award's internal identifier. It is used for the database and the web application also uses it to locate the award icon (`img/award-icons/<id>.png`).

The _meta information_ is for display in the web application. The following units are supported:
* `int` - a simple integer number with no unit.
* `ticks` - time statistics are usually measured in ticks. The web application will convert this into a human readable duration (seconds, minutes, hours, ...).
* `cm` - Minecraft measures distances in centimeters. The web application will convert this to a suitable metric unit.

The _StatReader_ is responsible for calculating the displayed and ranked statistic value. Most commonly, this simply reads one single entry from a player's statistics JSON, but more complex calculations are possible (e.g., summing up various statistics like for the `mine_stone`.

There are various readers, the usage of which is best explained by having a close look to the existing awards. If you require new types of calculations, dig in the `mcstats/mcstats.py` file and expand upon what's there.

## Removing awards
In order to remove an award, find the corresponding module and delete or modify it to suit your needs.

[1]:http://minecraft.gamepedia.com/Statistics
[2]:http://mine3.dvgaming.com/
[3]:https://github.com/pdinklag/MinecraftStats/releases/tag/1.0
[4]:https://github.com/pdinklag/MinecraftStats/issues
[5]:https://minecraft.gamepedia.com/17w47a
