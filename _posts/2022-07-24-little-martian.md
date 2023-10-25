---
title: All About Little Martian!
date: 2022-07-24 10:53:00 +/-0800
categories: [writing, game-dev]
tags: [little-martian, tutorial]
author: mwertman
---

Brief introduction on how I found one of my favorite games to both play and develop!

## Origin Story
Little Martian is a open world survival craft game (as of now, just a demo) created by Charlie, Jack, and Craig Smith. I think it's a beautifully-crafted game and I sure have been having my fun with it. Not only have I been playing the game, but taking a deeper dive into the files to see what makes this game tick. Game development has always been a curiosity of mine. I have never fully developed a game before or even know where to begin.

My plan was to essentially try and find a candidate that was easy to understand with my current limited knowledge of game-dev AND was able to be broken-down to it's source files. Effectively, instead of building a game, reverse-engineering an existing one. At the time, I was also helping out on an open-source project centered around de-compiling/reverse-engineering a little known GameCube game called Gladius by Lucus Arts. A lot of it was over my head, so I didn't work on the main tools, but rather creating helpful scripts/docs for people that were modding the game. At the root, this is where this idea came from. I wanted to work on my own project (something that I can actually understand!) on a little bit of a smaller scale.

I ended up finding Little Martian, which was the perfect candidate for this! A simple 2D sprite-based game written in 100% JavaScript.

## Technologies Behind The Game

[Construct3](https://www.construct.net/en) is a HTML5 2D game engine aimed towards non and beginner programmers. Construct3 offers a lot of cool features like easy-to-use web editor, 3D-like capabilities, and cross-platform support for any device. Since it is all made with web technologies (HTML, CSS, and JS), these games built within the engine can be play on anything that can run a browser.

[NW.js/node-webkit](https://nwjs.io/) is the main tool that helps make this cross-platform feature possible. Similar to [Electron](https://www.electronjs.org/), you can write or turn your web applications to native standalone applications using chromium and nodejs. This is was Construct3 uses behind the scenes to bundle/distribute your game!

## How Does Contruct3/NW.js Pack Your Game?

On the [NW.js docs](https://nwjs.readthedocs.io/en/latest/For%20Users/Package%20and%20Distribute/#package-your-app), it details two ways that it can package your app files:
1. Plain Files - Put all your files in the same directory as the NW.js binaries

2. Zip File - Package your app files into a zip file named 'package.nw'

The second method can be seen in Little Martian. Looking in the root directory, you will see an 'package.nw' like mentioned in the docs. I have not personally used Construct3 and am not sure if this is something that the developer can control or if it just the default settings set in the engine.

## Accessing The Game Files
> WARNING: The following section showcases how to get access to the source files of Little Martian and therefore can ruin the experience of the game. Even though it is only a demo, some spoilers may be exposed.
{: .prompt-warning}

With the history out of the way, I want to get into how exactly I got access to the source files for the game. It was surprising easy! You can even follow along with this section and start exploring the files yourself.

Firstly, however, I do recommend playing the game first as this might ruin the game for you.

Lets start by getting the game! Simply go the [Little Martian](https://little-martian.dev/demo/)'s website and download for your current platform. At the time of this post, the game is not fully out yet and only offers a playable demo right now. The current version is `0.6.2`.

If you are reading in the future and the above version is not the current version any longer, don't worry! The following content should work no matter what version. Just download what ever the latest version is. Or maybe the game is finally out on [Steam](https://store.steampowered.com/app/1455610/Little_Martian/).

> Depending on the platform, the following directory tree may look quite a bit different. Linux and Windows have a pretty similar structure. Mac OS's content mainly takes place in `Contents/Resources/`{: .filepath}.
As you will see, I will be working with the linux build.
{: .prompt-warning}

After you extract the zip archive, you will see a new directory `little-martian-0.6.2-<your-platform>`{: .filepath}.
The following is the structure of the **Linux** version of the game.
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
*There is another directory, `__MACOSX`{: .filepath}, but this can be safely ignored/deleted.*

The key file to be aware of is the `package.nw`{: .filepath} archive or `Contents/Resources/app.nw`{: .filepath} on Mac OS as this is where all of the game files are stored! In here is all the engine code (!) and user scripts for the game.

And it is just as simple as extracting the `package.nw`{: .filepath} archive as any other zip archive!

Here is the inner `package.nw`{: .filepath} project structure:
```
├── cursor.css
├── data.json
├── icons
│   ├── ...
├── images
│   ├── ...
├── index.html
├── media
│   ├── ...
├── package.json
├── scripts
│   ├── c3runtime.js
│   ├── dispatchworker.js
│   ├── jobworker.js
│   ├── main.js
│   └── supportcheck.js
├── style.css
```

## Game Extractor Script
As a bonus, I will include a helpful shell script that does just this.
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

Feel free to try on your own! You are now able to explore and even get into modifying the files.

Something to play with is this [randomizer script](https://gist.github.com/MatthewWertman/d3385c4f50c09585d15639fbcdcdc9d6) that I wrote in Python. It will randomize the starting items
that you get from your ship! *Make sure to have the game extracted before running the randomizer.*

One last thing, I really want to thank the devs for their amazing game! I am excited to see the finished product and have become a fan of their work. I have interacted with Craig, the lead developer, and he was really nice and was telling me of some cool ideas that might get added! Tune into their Twitter or Discord server for updates with the links below. Show them some love!

## Links
[Little Martian website](https://little-martian.dev/)

[Little Martian Twitter](https://twitter.com/MartiansGame)

[Little Martian Discord Invite](http://discord.gg/fH7agQaHPx)
