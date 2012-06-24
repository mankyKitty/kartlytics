# kartlytics

kartlytics is a web app for browsing records and statistics of recorded Mario
Kart 64 races.

kartvid (the underlying engine for kartlytics) is a facility for processing
Mario Kart 64 still frames from a screen capture video.  The goal is to take a
video of a session and extract information about the courses played, the
characters in each box, each character's position at every point in the race,
and the final lap times.  Future enhancements could include weapon information,
too.

# Running kartlytics

## Recommended prerequisites

kartlytics runs on SmartOS and Mac OS X, and should run anywhere else its
dependencies are available:

- [ffmpeg](http://ffmpeg.org/).  ffmpeg is used to decode screen capture videos.
- libpng, which is used to read individual frames and masks.
- imagemagick, for the command-line "convert" utility.
- Node.js 0.6.x, for the web server.
- cscope for source browsing.

On **SmartOS**, you can install these with:

    pkgin -y install ffmpeg png ImageMagick cscope

I do this in a smartos-1.6.3 zone.

On **MacOS**, I suggest first installing XCode and
[homebrew](http://mxcl.github.com/homebrew/).  Then you can install the
dependencies with:

    brew install ffmpeg
    brew install imagemagick
    brew install cscope

An old version of libpng is shipped with OS X in /usr/X11.

For reference, kartlytics works with ffmpeg versions v0.7.8nb1 (pkgsrc, SmartOS)
and v0.10.3 (built from source, MacOS, but only after using homebrew to try to
install it, which installed its dependencies).

For both SmartOS and MacOS I build Node.js from source.  On SmartOS, you'll want
the scmgit, python27, gcc-compiler, gcc-runtime, and gmake packages to build
it.

## Build and run

Run "make" to build kartvid, generate masks, and install the required Node
packages.  It assumes your compiler, ImageMagick's "convert", and Node's "npm"
are in your path.

You can run `out/kartvid` directly to see its usage information.

You can run the web server with:

    node js/kartlytics.js

To run kartlytics as an SMF service on SmartOS, modify the paths in
smf/kartlytics.json as desired, regenerate the SMF manifest with
[smfgen](https://github.com/davepacheco/smfgen), and then

    svccfg import yournewmanifest.xml

## Working with images

I use Photoshop (but you could use the GIMP or some other image editor) for
creating masks and debugging (see below).  You'll want support for portable
bitmap (PPM or PBM) images.  For PS, you can get that using the [CartaPGM
plugin](http://www.reliefshading.com/software/CartaPGM/CartaPGM.html).

To capture stills and video, I'm using an iGrabber device with the stock
software.


# Approach to data extraction

The basic approach relies on the fact that most objects in the game are either
2D objects (like text) or 3D objects (like characters) rendered using a series
of 2D sprites, which means you only ever see a few different shots them.  The
simple approach we're taking for now is to create "mask" images for every object
we want to detect that consist of only that object, in the precise position
where we want to look for it, and all other pixels black.  The program can then
check all possible masks against a given frame to identify the objects in it.

To create each mask:

1. Capture a high quality still image (in BMP format) of the object on the
   screen.
2. Open it in an image editor.  (I'm using Photoshop CS5.)  Select only the
   object you want (using the magic wand or magnetic lasso tools, for example),
   invert the selection, and delete the selection.
3. Save the image as a portable bitmap (P6) file.  (You can also save it with
   some other lossless format and use the "convert" tool to convert it to PBM.)


## Object variations

Most objects appear in different sizes depending on whether it's a 1P, 2P, or
3P/4P game.  For now, we're focusing only on the 3P/4P case, and we're assuming
that a given object is always the same size.  The only objects that change size
are the characters themselves, depending on whether the player is zoomed in or
not, and we can work around that by checking two masks for each character (one
for each zoom setting).

We also assume a given object only appears in exactly one position on the
screen, which allows us to compute mask matches pretty efficiently (rather than
trying all possible positions on the screen).  This means we need a mask for
each object in each of the 4 boxes on screen.  For characters, these are
automatically generated by the build process based on the Player 1 masks.
Tracks only ever use the Player 1 mask.

Importantly, we know we're analyzing a whole video, not just individual frames.
We don't necessarily need to identify all objects in all frames.  We can get
away with only having a single view of character as long as we know that we'll
always see that view at least once in each race.  We use the back view, and we
assume we'll see that just before the race starts.  We don't bother handling any
of the other views of each character.


## Analyzing a race

We identify the start of a race by looking for a special Lakitu object.  (He's
the guy holding the stoplight at the start of the race.)  When we see him, we
should be able to reliably identify the current track and the players in each
box, which are all pretty fixed at that point.  We can also start the race clock
at this point.

While the race is going on, we only monitor the position and lap number for each
player.  This allows us to see all position changes as well as lap times for
each player, up through the 3rd lap.  And these are relatively easy, since
they're just simple text objects.  (The only problem is that the lap counters
are turned off if the user switches to the alternate view.  If this becomes a
problem, we could look for another way to detect lap completion (perhaps by
looking for another Lakitu object).)

We detect when each player finishes the last lap by looking for the large
flashing number indicating their final position.  From this we have the final
results and the final race times.


## Identifying the track

Obviously, to identify the track, we'll want a mask to represent the way the
track appears in at least one of the boxes.  Since different players can have
different zoom levels, which would cause the track to look different in the
initial screen, it would be impractical to try to match the track based on any
combination of player boxes.  So for simplicity, we'll just create masks for the
1P box's zoomed-in and zoomed-out views.  (The rest of the mask will be black.)
We do this using a track mask generator, which can be applied to a screenshot of
a track in the starting position to generate the track's mask.


# Roadmap

This project is still just a prototype.  Currently, on at least some input, it's
able to identify the start of the race, the characters playing, the positions of
each player during the race, and the end of the race.  It emits both plaintext
and JSON, and reads PNGs, PPMs, and raw videos.  There's also a primitive Node
server that processes video uploads.  Remaining items include:

- finish web client (see TODO in www/resources/js/kart.js)
- try on more types of inputs
- handle aborted races better. (kartvid detects this, but doesn't emit events
  very nicely.)
- detect pause screen.
- Detect weapons.  The most reliable way to detect weapons gotten is probably to
  look at the *last* weapon in the weapon box before the box itself disappears.
  (All the other ideas I've come up with can't really handle the case where a
  player immediately uses the weapon.)  But even then, you have to worry about
  super mushrooms (which flash), triple mushrooms (which become 2 or 1 before
  going away), and people stealing weapons with ghosts.
- Position along the track for detecting "hot spots" where people tend to get
  slowed down.
- Detect lap completion for lap times.
- Billing: spending most of the race in 1st place, but losing due to weapon
  usage late in the race.
- Keithing: going from 1st to 4th within a few seconds.
- Average, median, Nth percentile race and lap times, per player per track (per
  racer?)
- Banana-saves: times when someone saves themselves from a banana spin-out
- Rescues: times when Lakitu has to come fetch you
- Impacts-by-shell: use explosion or "crash" to detect getting hit by something
- Impact of bomb?
- Power slides, boosts, and boost attempts?
