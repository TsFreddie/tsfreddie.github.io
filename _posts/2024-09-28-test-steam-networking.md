---
title: "How to Test or Debug Steam Networking"
excerpt_separator: "<!--more-->"
categories:
  - Blog
tags:
  - Steam
  - Networking
  - Publishing
  - Debugging
---

I have worked at several companies who develop multiplayer video game projects. Anyone who's been through the process of developing multiplayer games probably shares the pain of juggling multiple running games at the same time. Just to find out the game still breaks when three or more players are connected during testing.

Regardless of the difficulty of testing multiplayer games. The initial development still involve launching multiple instances just to see if they connects. For all my past experiences, this stage is usually done locally on my workstation. Having one socket connect to another local socket involves very little effort: hardcoding "localhost" as the address and click the magical "host" and "connect" test buttons (which you coded yourself of course).

However, integrating with target platforms like Steam make things a little bit more complicated.

<!--more-->

> TL;DR:
>
> 1. Launch Steam with `-master_ipc_name_override ipc_name -userchooser` to launch a new Steam instance with a different IPC name and sign in with another Steam account.
> 2. Launch your game with `steam_master_ipc_name_override` environment variable set to the name you used (e.g. `ipc_name`) before initializing Steamworks SDK.
> 3. Do this for every new instance with different IPC name.

## Steam is (not actually) single instanced

By default, you can only open one Steam on your computer. You can open multiple instances of your game and initialize Steamworks SDK on all of them just fine. But once you started trying to send a message to the other instance through Steam Networking, you'll realize Steamworks SDK uses Steam ID as the identifier of the connection source and destination (Example: [SendMessageToUser](https://partner.steamgames.com/doc/api/ISteamNetworkingMessages#SendMessageToUser)). You might realize the two instances shares the same Steam ID as they are both logged in with the same account.

Now, if we could launch two Steam with different accounts, this wouldn't be a problem anymore. Let's figure out how to do exactly that.

Launching a second Steam is actually easier than you thought. Steam communicates with it's SDK via a IPC mechanism, each IPC is named with a unique string. If you override the IPC name, Steam will launch a new instance for that name. We can do that by passing the following command line arguments while opening Steam:

```bash
-master_ipc_name_override testing -userchooser
```

You can replace `testing` with any name you want. The `-userchooser` argument is to make sure Steam opens the user chooser dialog instead of automatically logging in with your main account _again_.

## Make your game recognize the second Steam

If you open two game instances from your game engine or project directory, you'll notice that they are still both logged in with your main account. Now you have three choices:

1. Upload your game to a Steam Depot and download it to the second Steam. Open the game via Steam as usual.

2. Add your game as a non-Steam game to the second Steam. Then you can launch it that way.

3. My favorite choice: set a `steam_master_ipc_name_override` environment variable before initializing Steamworks SDK. You can do this before launching your game or during your game's initialization, as long as it is set before you call [`SteamAPI_Init`](https://partner.steamgames.com/doc/api/steam_api#SteamAPI_Init). This way you can launch your game directly without opening it through Steam.

You might have guessed it already: all three methods above are actually the same. Steam just set the environment variable for you if you launch it through the store's interface.

## Why is this not documented anywhere?

Believe me, I already tried to do exactly this before. However, searching "testing steam networking" usually leads you to believe you need multiple machines to test your game.

I only found this method after one day I just randomly decided  to search "opening multiple steam" instead. There are quite a few guides on how to do that with the perspective of players who are trying to loggin their game with multiple accounts. Retroactively searching `master_ipc_name_override` and `steam_master_ipc_name_override` in the Steamworks official documentation also give you zero results.

Anyway, I actually can't believe that it took me three years to figure this out. I've been testing our games on multiple machines as informed, or even worse - by calling my friends to help then disappoint them when the new update simply doesn't work. I hope this post can save some friendship in the future like it might have saved mine.

