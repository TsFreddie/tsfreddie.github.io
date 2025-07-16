---
title: "The Brutal Optimization of RPG Maker MV"
excerpt_separator: "<!--more-->"
categories:
  - Blog
tags:
  - RPG Maker
  - Profiling
  - Debugging
  - JavaScript
  - Optimization
---

When a game you desperately want to play runs at 15fps and stutters constantly, you have two choices: give up, or dive deep into the engine's guts with a profiler and some determination. This is how I managed to optimize a borderline unplayable game to a buttery smooth 144hz experience.

<!--more-->

> NSFW Warning: Although this post does not describe any game content, the game mentioned contains mature, homosexual, and furry content.

> You can find the final optimization script [here](https://github.com/TsFreddie/rmmv_optimization).

## It's not RPG

Despite the engine's name, the game I am trying to optimize is far from being a traditional turn-based RPG game. Instead, it is a side-scrolling action game. Intuitively, this game is more heavy on scripts than a traditional RPG game.

If it wasn't obvious, RPG Maker MV is just a JavaScript web game engine using PixiJS as the renderer and NW.js as the browser runtime. This, along with the fact that the engine code is not obfuscated, makes debugging it trivial in theory. However, reality hits you in the face almost immediately before you can even get started.

## Unusable DevTool

The target game of my choice is "Polidog Patrol" which is the game I always wanted to play, but the framerate can drop as low as 15fps sometimes and the constant stuttering on my machine makes it unplayable.

Since NW.js is basically chromium, the chromium devtool has some incredible features for diagnosing JavaScript performance. RPG Maker MV actually blocks you from using the devtool by disabling the context menu and blocking F12 key events. However it is simple to comment out the event handling code:

```js
// rpg_managers.js
case 123: //F12
    if (Utils.isNwjs()) {
        event.preventDefault();
    }
    break;
```

After that, I can simply open devtool by pressing F12... **_Oh no_**.

Upon opening the devtool, the game immediately drops to 5fps. The devtool itself is also almost unresponsive, clicking anything takes about 5 seconds to register. Despite the slowdown, it is still possible to navigate to the performance tab and record in the profiler. I just have to be patient. So I hit the record button, and let it record for 5 seconds. Recording that 5 seconds of gameplay took the devtool about 2 minutes to finish processing.

After the initial recording of the title scene, it is quite obvious which functions are the main culprit: `Game_Interpreter.command355` and `Game_Interpreter.command111`.

![Initial Profiler Result](/assets/imgs/2025-07-16-optimizing-rmmv/image_1.png)

Let's take a look at what this `command355` is doing:

```js
// rpg_objects.js
Game_Interpreter.prototype.command355 = function () {
  var script = this.currentCommand().parameters[0] + "\n";
  while (this.nextEventCode() === 655) {
    this._index++;
    script += this.currentCommand().parameters[0] + "\n";
  }
  eval(script);
  return true;
};
```

After a chuckle to myself, the `eval` call immediately jumped out. Let's remember that the game does not actually run at the blazingly fast 5fps before we open the devtool. My hypothesis is that the `eval` call is somehow blocking the devtool due to some kind of chromium wizardry, and the sheer amount of `eval` functions being called per frame is choking the devtool.

To confirm my hypothesis, I need to somehow "eliminate" all the eval calls.

## "Pre-compiling" the Scripts

After taking a closer look at `Game_Interpreter`, I found that `command355` is basically the custom script block in the game's script. It might be funny to you that the game's "interpreter" is just calling `eval` functions with provided JavaScript code. But the actual interpreter does more than just running JavaScript code - it handles its own branching and looping (foreshadowing), and due to how the game script is structured, every `command355` block only runs one line of code. Apparently the game's heavy usage of `command355` is making it very slow.

The game's script/code is loaded by the engine from a series of JSON files (JSON as a programming language?). The interpreter then goes through a list in the structure line by line, or more accurately, item by item. I can't really eliminate the `eval` call since the strings in the JSON file do need to be executed. However, I can wrap the code as a function and `eval` the function once on load, then call the function in place of every `eval` while the game runs.

```js
// Instead of running this every call
eval(script);

// I can run this once when loading the JSON file
const func = eval(`(__)=>{\n${script}\n}`);

// Then run this instead in place of original eval
func();
```

I quickly whipped out a modification that intercepts the various loading methods and runs my custom "pre-compile" routine and stores all the functions in the interpreter instance.

This solved the eval problem by running all the required `eval` calls during load time. However, profiling revealed another performance bottleneck in the interpreter's control flow.

## Jumping

The engine's `Game_Interpreter` also does branching and looping, which requires the interpreter to move to another part of the script list to continue execution. Here is how the interpreter handles the `else` statement:

```js
// rpg_objects.js
Game_Interpreter.prototype.skipBranch = function () {
  while (this._list[this._index + 1].indent > this._indent) {
    this._index++;
  }
};

// ...

Game_Interpreter.prototype.command411 = function () {
  if (this._branch[this._indent] !== false) {
    this.skipBranch();
  }
  return true;
};
```

All `skipBranch` does is iterate through the list until it finds a command with a different indent level. Needless to say, this is also quite slow. However, it is possible to precompute the jump index for every index in the list. When `skipBranch` is needed, I can just set `this._index` to the precomputed value. Additionally, `command113` (Break Loop) does a different kind of jumping. And `command108` (Comment) is just comment blocks that the interpreter needs to skip through. I precomputed all the jump indices upon loading the JSON file and modified all the relevant commands to use the precomputed value.

After optimizing both the "script" calls and "jumping" commands, I returned to the trusty profiler to see whether improvements had been made. After opening devtool, it is obvious that the devtool is now very responsive. Here's the new recording:

![Second Profiler Result](/assets/imgs/2025-07-16-optimizing-rmmv/image_2.png)

**Is that a 35x improvement I see?!** Now the game runs at full speed. **_But wait_**, full speed it sure is! The game is running so fast that the gameplay is running at 2x speed now. After a bit of digging, I found that the game uses a `FpsSyncOption` plugin. By looking at the code and the way it is set up, it forces the game to update at every `requestAnimationFrame`. Since RPG Maker MV is designed to only run at 60fps, the engine logic is also framerate dependent. Due to my 144hz monitor, the game updates 144 times per second instead of 60. Turning off the `FpsSyncOption` plugin fixes the issue and the game runs beautifully at 60fps.

## FAKEFRAMES™

The game running at its intended speed and framerate - what more could I want? Well, if I did have a 60hz monitor, or even a 120hz monitor, I probably would be very happy to be able to play the game at buttery smooth 60fps. However, since I have a 144hz monitor, rendering at 60fps actually introduces terrible frame pacing. Since the game is a side scroller, the jittery motion of the camera is very distracting.

The game I am trying to optimize is particularly interesting as an RPG Maker game - it doesn't actually use much of the engine's features. There are no "characters", "battles" or any of the RPG elements. Essentially the game bypasses most of the engine's gameplay features. Instead, the game opts to use lots of custom scripts and only "pictures" as its sprite system and "tilemap" as its map.

Now, since I want to render the game at high refresh rate, and the only thing I need to worry about is "picture" objects, I thought I can just intercept the picture rendering system and interpolate every picture's position between current and last game ticks. Sure enough, it is a fine idea. I quickly implemented a picture interpolation system, and confirmed that it does work _almost_ flawlessly. There are some quirks:

- Some sprites teleport. I added a distance check for this - if a picture moves too far across the screen, it resets the interpolation state and lets it teleport.
- Infinite scrolling sprites are done by teleporting back to the beginning, this introduces one frame of jitter when it teleports, making it not so seamless. I identified all the scrolling sprites and marked them as scrolling. Then every time they teleport, I also teleport the last frame's position back as well to make it more seamless.

This does introduce a tiny bit of input latency, but I've played through the game and it is not really noticeable. Most people are concerned about input latency when it comes to any type of interpolation. By interpolating from previous game update to current game update, you can say there is 1 frame of input latency. In reality, it is harder to define, since realistically, after the game processes the input, it changes from seeing the result of the frame to seeing the beginning of movement of that frame - you surely would notice your input being accepted even halfway through interpolation, so some might even say it's only 0.5 frame of latency. In any case it still feels very responsive.

## Moving the Goalposts

Now the game renders at high refresh rate with the power of FAKEFRAMES™. A new problem becomes immediately noticeable. The game can feel stuttery even when the framerate seems high enough. Well, since you can now see render updates at 144hz, our game logic's CPU budget plummets from 16ms down to just under 7ms. Even if the game only runs the update 60 times per frame, reaching the budget limit of 7ms still delays the next interpolated frame, which introduces stutters.

In actual video game engines, if your game runs at 60 ticks per second with interpolation of some kind, you can get away with updating the game logic while rendering out the interpolated frames. Obviously, in a browser environment with single-threaded JavaScript, this is not possible. So we need to actually optimize the game as if it is targeting 144hz to squash the frame time to be consistently under 7ms.

With that goal in mind, I played the game normally, observed the behavior, and profiled the stuttering parts to identify further optimization opportunities. Here are the main bottlenecks I discovered and how I addressed each one:

### Excessive DOM Elements

After each battle, there is a result overlay of the battle. I noticed sometimes when the overlay is shown, the game slows down quite significantly, but it doesn't happen every time. There is a text sprite plugin `DTextPicture` - when the overlay is shown, the game creates multiple RPG Maker's `Bitmap` instances each frame. Each frame contains an `HTMLCanvasElement`. I'm not entirely sure whether creating an excessive amount of canvas instances would slow down the browser, but after modifying the plugin to pool the `Bitmap` instances and reuse them as much as possible, the slowdown is gone.

### Minimap Rendering Performance

When the minimap is open, frame rate drops whenever the player moves. Profiling shows `Sprite_Minimap.update` takes up to 4ms. This is a custom sprite in the plugin `KMS_Minimap`. The plugin retrieves information about the tilemap and draws the minimap pixel by pixel by calling `Bitmap.fillRect` which essentially calls canvas context's `fillRect`. Calling `fillRect` for every pixel seems to be slow. So I modified the plugin to have two different `Bitmap` instances - one for the current frame, and one for the previous frame. Whenever the player moves, I swap the Bitmap instances and clear the current one, calculate the overlapped area of the minimap, and blit the previous frame's bitmap onto the current one. Then update the rest of the minimap as usual. I also added dirty flags to each tile of the tilemap in case the overlapping area also needs to be updated somehow. This way we only need to update the changed area of the minimap, instead of the entire thing every time it updates.

### Decryption Bottleneck

Whenever the game loads an image or audio file, the engine also needs to decrypt them. For audio files specifically, whenever audio plays, the game actually loads the audio from scratch again. Chromium is pretty good at caching the file access, but after the file loads, the engine decrypts the audio file synchronously. This can be very expensive whenever there is more than one audio playing at the same time. I modified the game to use a decryption worker to decrypt the audio file on a separate thread. This way the decryption is done asynchronously although the worker still only does one decryption at a time. But the decryption routine no longer blocks the main thread.

### Sprite Tinting Cache

RPG Maker MV can tint a sprite by creating a canvas and running various filters then reading back the canvas data to use as image data. This only happens once when the sprite is tinted. However, in the late game, there are flicker effects for game character sprites done by updating the tint of an image each frame. When multiple flickering characters are on screen, multiple tinting routines run every frame. I modified the tinting routine to cache the tinted image data after it is tinted by their tint parameters. This way, the tinting routine only needs to run once for each sprite, and can be retrieved back from the cache whenever the same sprite with the same tint color is used.

### Optimizing Picture Management

Probably for ease of managing sprites by assigning them large IDs with actual meanings (which is common in game development), the game requested the engine to provide a picture bank of 10,000 pictures. Each frame, the game iterates through all 10,000 picture IDs to update them. Obviously the game doesn't actually render 5 digits worth of sprites. It usually only has about 100 pictures shown at any time. So I modified the game picture system to put active pictures into a separate container, then only update the active pictures. I also updated the FAKEFRAMES™ to only iterate through active pictures too.

### Z-Index Sorting Optimization

The game uses a `PictureZIndex` plugin to assign pictures with a z-index value. Without this plugin, the game renders from picture 1 to picture 10,000 in order, therefore putting pictures with higher IDs on top. The game uses the plugin to dynamically adjust the z-index of pictures. However, the plugin sorts the picture container every time a picture's z-index is changed. In some cases, the z-index is set multiple times per frame, and, remember, the game is sorting 10,000 pictures every time. Given the simplicity of the plugin, I nuked the plugin and rewrote it entirely by assigning the z-index and managing a dirty flag to make sure the pictures are sorted correctly right before the engine renders them. Making sure the game sorts at most once per frame.

## Conclusion

I've completed the game and the game is also optimized to the best of my ability. Using RPG Maker MV to make a non-RPG is no small feat. The game engine and its plugins are perfectly suited for turn-based RPGs where logic and graphical changes usually happen only every so often. But they ultimately fall short when it comes to making anything else that requires a more intensive game loop.

These optimizations are very game-specific, and I cannot promise they will work for other games. I have sectioned the optimization script with headers to explain their purpose. If you also use RPG Maker MV and want to optimize your game, hopefully this write-up can give you some ideas.

Ironically, going back and forth between playing and optimizing is not really a good way to play video games. I hope this optimization work will benefit other players more than myself. But I also hope someday I can completely forget about the game so I might be able to enjoy the buttery smooth game in 144hz like a normal person.

Here's the final profiler record after everything:

![Final Result](/assets/imgs/2025-07-16-optimizing-rmmv/image_3.png)
