---
title: All About Little Martian!
date: 2022-07-24 10:53:00 +/-0800
categories: [writing, game-dev]
tags: [little-martian, tutorial]
author: mwertman
---

Breif introduction on how I found one of my favorite games to both play and develop!

## Origin Story
Little Martian is a open world survival craft game (as of now, just a demo). All made within the Construct3 engine by Charlie, Jack, and  Craig Smith. I think it's a beautifully-crafted game and I sure have been having my fun with it. Both gameplay-wise and the development side. No, I did not join the development team (though I have interacted with Craig, the lead developer, and he is very nice :)), but I have been taking a deeper look into what makes this game tick. I have never fully developed a game before, but this one immediately caught my attention when I first saw it. Obviously , this type of game has been made before but I like a lot of what this one does in paricular. 

However, the main thing that caught my attention was that it's all web-based! It all takes place in the browser essentially. I was very curious about this and instantly wanted to learn more.

## Technologies Behind The Game

[Construct3](https://www.construct.net/en) is a HTML5 2D game engine aimed towards non and beginner programmers. Contruct offers a lot of cool features like cross-platform capabilities. Since it is all made with web technoligies (HTML, CSS, and JS), these games built within the engine can be play on anything that can run a browser.

[NW.js/node-webkit](https://nwjs.io/) is the main tool that helps make this cross-platform feature possible. Similar to [Electron](https://www.electronjs.org/), you can write or turn your web applications to native standalone applications using chromium and nodejs.

## How Does Contruct3/NW.js Pack Your Game?

On the [NW.js docs](https://nwjs.readthedocs.io/en/latest/For%20Users/Package%20and%20Distribute/#package-your-app), it details two ways that it can package your app files:
1. Plain Files - Put all your files in the same directory as the NW.js binaries

2. Zip File - Package your app files into a zip file named 'package.nw'

Not sure if this is just the default way the Contruct does it, but Little Martian uses the second method. We will get into this in the next section.

## Accessing The Game Files
> WARNING: Spoiler warning for the game if you decide to look at the game files following along with this section.
{: .prompt-warning}

With the history out of the way, I want to get into how exactly I got access to the source files for the game. It was surprising easy! You can even follow along with this section and start exploring the files yourself. 

Firstly, however, I do recommend playing the game first as this might ruin the game for you.

Lets start by getting the game! Simply go the [Little Martian](https://little-martian.dev/demo/)'s website and download the version for your current platform. At the time of this post, the game is not fully out yet and only offers a playable demo right now. The current version is `0.6.2`.

### If Version 0.6.2 is Not Available
Don't tell anyone I told you this, but you may still be able to get version 0.6.2 if it's unavailable on the website. All you time travellers, LISTEN UP!

Go to `https://little-martian-builds.s3.eu-west-2.amazonaws.com/0.6.2/little-martian-0.6.2-<platform>.zip` and it should automatically download the 0.6.2 version from their build server!

If you want, you can also validate your download with the following md5 checksums:
```
c608056861e43850db2dc5f28059694c  little-martian-0.6.2-linux.zip
53db2b5f537415e934b93969489764db  little-martian-0.6.2-mac.zip
b33c23ca781e0f5a73d2928a17c3b72a  little-martian-0.6.2-windows.zip
```

> I will be working with the linux version as I am currently on Pop!_OS. So the following tree of files may be different for you depending on the version for your platform.
{: .prompt-warning}

After you extract the zip archive, you will see a new directory `little-martian-0.6.2-<your-platform>`{: .filepath}. 
The following is the structure of the **linux** version of the game. 
```
└── little-martian-0.6.2-linux
    ├── icudtl.dat
    ├── lib
    │   ├── libEGL.so
    │   ├── libffmpeg.so
    │   ├── libGLESv2.so
    │   ├── libnode.so
    │   └── libnw.so
    ├── little-martian
    ├── locales
    │   ├── en-US.pak
    │   └── en-US.pak.info
    ├── nacl_helper
    ├── nacl_helper_bootstrap
    ├── nw_100_percent.pak
    ├── nw_200_percent.pak
    ├── package.nw
    ├── resources.pak
    ├── swiftshader
    │   ├── libEGL.so
    │   └── libGLESv2.so
    └── v8_context_snapshot.bin
```
*There is another directory `__MACOSX`{: .filepath}, but this can be safely ignored/deleted.*

The key file to be aware of is the `package.nw`{: .filepath} archive or `Contents/Resources/app.nw`{: .filepath} on Mac OS as this is where all of the game files are stored! In here is all the engine code (!) and user scripts for the game. 

And it is just as simple as extracting the `package.nw`{: .filepath} archive as any other zip archive!

## Game Extractor Script
As a bonus, I will include a script to help extract the game. Its nothing fancy!
```bash
#!/bin/bash
# game_extractor -- extracts little-martian game

# get unzip and perl
unzip="$(which unzip)"
perl="$(which perl) -pe"


os="linux"
version="0.6.2"
package="package.nw"

function format_version_number ()
{
    v="$1"
    length=${#1}
    if [[ length -eq 2 ]]; then
        v="0$1"
    fi
    if [[ length -eq 1 ]]; then
        v="00$1"
    fi
    echo $v | $perl 's/(\d)(?=\d)/\1\./g'
}

function safe_cd ()
{
    cd "$1" || { echo "$0: Error: unable to change directory to $1" >&2; exit 1; }
}

OPTIND=1
while getopts ":o:v:a" opt; do
    case "$opt" in
        "o") os=$OPTARG; ;;
        "v") version=$(format_version_number "$OPTARG"); ;;
        \? ) echo "$0: Error: Invalid Parameter -$OPTARG."; print_usage; exit 1 ;;
         : ) echo "$0: Error: $OPTARG requires an agrument." >&2; exit 1 ;;
    esac
done
shift $(($OPTIND - 1))

location="little-martian-${version}-${os}/"
$unzip little-martian-${version}-${os}.zip -d $location
rm -rf "$location/__MACOSX"
if [[ $os == "mac" ]]; then
    location="little-martian-${version}-${os}/Little Martian (${version}).app/Contents/Resources/"
    package="app.nw"
else
    location="little-martian-${version}-${os}/little-martian-${version}-${os}/"
fi
safe_cd "$location"
$unzip $package
exit 0

```
{: file="game_extractor"}

To use, run the following: `./game_extractor -o <platform> -v <game-version>`

## Closing Notes

Feel free to try on your own! You are now able to explore and even get into modifing the files.

Something to play with is this [randomizer script](https://gist.github.com/MatthewWertman/d3385c4f50c09585d15639fbcdcdc9d6) that I wrote in Python. It will randomize the starting items
that you get from your ship! *Make sure to have the game extracted before running the randomizer.*

## Links
[Little Martian Website](https://little-martian.dev/demo/)

[Little Martian Twitter](https://twitter.com/MartiansGame)
