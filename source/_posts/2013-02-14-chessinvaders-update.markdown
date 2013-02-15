---
layout: post
title: "chessInvaders update"
date: 2013-02-14 01:40
comments: true
categories: Log 
tags: [chess, game, iOS, education]
---

ChessInvaders is submitted to the Apple App store and is pending review!  I have a [limited functionality web port up](/chessinvaders/webplayer/). Some stuff doesn't work due to it being on the web rather than on iOS.  To pinch the screen hold down the option key. 

Turns out that porting to the web wasn't straightforward at all. The Unity web player runs in a partial trust environment which ends up affecting Coroutines in a weird way.  If you have some classes where you are overriding virtual IEnumerator or Coroutine methods but still want to call the base class versions then those trigger a ValidationException.  All my GUI code is built out of smaller components which all have virtual Coroutine TurnOn/TurnOff methods, blam, that won't work on the web.  I hacked it up to work for now, but some functionality isn't there (highscores) and its not worth it to try to redesign my GUI code right now.

Regardless, hopefully soon it'll be coming to the app store on an iOS device near you!  Also it'll be free. 


