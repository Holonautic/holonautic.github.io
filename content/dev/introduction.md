+++
title = "Introduction"
description = "What goes in to a Boom Boom Hamster Doom? An overview of the project."
date = 2024-10-23T12:00:00Z
draft = false
template = "dev/page.html"

[taxonomies]
authors = ["Bryn Dickinson"]

[extra]
lead = "Welcome, everyone. This is the <em>nerdy</em> part of the website."
toc=true
+++

I'm Bryn (aka [canmom](https://canmom.art/)), a programmer, technical artist and animator working on <cite>Boom Boom Hamster Doom</cite>. If you looked at the front page, you might have an idea what this game is about: teams of hamsters with missiles and guns, trying to blast each other into the ocean while making giant holes into everything nearby.

It's a genre of game with [a storied tradition](https://en.wikipedia.org/wiki/Artillery_game), from <cite>War 3</cite> in 1972 to dozens of games called <cite>Artillery</cite>... and of course the one you probably know best, Team17's <cite>Worms</cite> series. But! To date, nobody has made a game of this type for VR (so far as we know)...

<cite>Boom Boom Hamster Doom</cite> is a meeting of the artillery game with tabletop wargaming. The hamsters inhabit an island, floating in front of you. Everything you do is very tactile: you move your hamsters around by pinching your crab claw and pulling on little leashes. You aim the hamsters by pulling back from them like a catapult. You launch an airstrike by throwing it like a paper plane.

## the challenges

This type of game has a *lot* of technical demands. Maybe you know this: in VR, performance is *mandatory*. If you can't maintain 72FPS at all times, even hardy players will rapidly get motion sick. That means every frame has a hard time limit: about 14 milliseconds.

Moreover, we're running on what is essentially mobile hardware: a single ARM chip with limited specs (the Snapdragon XR2 in two different generations), and that imposes pretty severe limits on what is possible graphically. For example, ever noticed that essentially no Quest games have realtime shadows? There's a very specific reason for that, to do with GPU architecture.

Despite this, we've still got a surprisingly large number of toys to play with. The hardware is limited, but it's not *old*, and you've got full-featured vertex and fragment shaders, and a couple of surprisingly zippy CPU cores to play with.

Even so, we are asking a lot of this little device. Here's some of the battles we'll be discussing over the course of this series...

### voxel terrain

We need to blow holes in terrain from just about any angle, and that means voxels. I don't mean <cite>Minecraft</cite>-style cubes here, but data arranged on a grid, a few bytes for each point.

More precisely, our ground is an implicit surface, defined by the values of a 3D grid of voxel data, which we mesh using [marching cubes](https://en.wikipedia.org/wiki/Marching_cubes). This is the technique used in PC games such as <cite>Deep Rock Galactic</cite>, <cite>No Man's Sky</cite> and <cite>Space Engineers</cite>... but it's rarely been used in standalone VR.

The upshot of this is that we have a *lot* of data to crunch, very fast, whenever anything blows up.

Besides that is the question of creating the island 

To pull this off in Unity, we needed to call on the might of the Burst compiler and Job system, and be smart with chunking the data and spreading the calculation across multiple frames. After that, shading the mesh has its own challenges---smoothing vertices, optimising draw calls, and texturing and lighting the procedurally generated terrain.

### networked physics

Our game is a physics-based game, and a physics-based game that needs to maintain network state between as many as twelve different players. It also needs to be smooth and responsive enough to work in VR. That means rollback netcode.

BBHD is server-authoritative, meaning that one of the players (the host or server) has the final word on game state. <dfn>Prediction/rollback netcode</dfn> means that the game runs a few frames ahead of the latest information from the server, applying the most recent inputs from the player to minimise latency. But that means it can make mistakes! For various reasons: the actions of another player will *always* be different than expected since they consist of real-time 3D tracking, and on top of that, small divergences in physics calculations can gradually magnify into big differences.

So whenever the client receives an update, if there's a mistake (and there is nearly always a mistake), it has to resimulate from that point up to the current frame (a 'rollback'). This means it may have to calculate quite a large number of physics steps in one frame---a problem that compounds as latency between players increases.

The framework we eventually settled on was Unity DOTS Netcode, since this gave us the most robust and reliable network state. This means our game must be built in Unity's Entity-Component-System framework, something which brought with it as many headaches as advantages...

### DOTS performance woes

Our game is built in DOTS: the '[Data Oriented Technology Stack](https://unity.com/dots)'! Unity has been hard at work on DOTS for a long time, but most of the people who originally designed the system have left after... tumultuous times at Unity, so it presently exists in a strange kind of limbo. DOTS is supposed to offer 'performance by default': organise your code according to its patterns and you can expect things to run *much* faster than the old-fashioned way of MonoBehaviours and GameObjects. DOTS demos show impressive scenes of thousands and thousands of entities, thanks to cache-optimised memory layouts and efficient paralellisation and so on...

...but it turns out this comes with a huge asterisk. Because for everything we gain in Burst compilation and so forth, we lose a great deal in overheads, particularly around system and job scheduling. Things that should take almost no time at all, like schedulling simple systems, might take as long as 0.1ms... which might not sound like a lot, but with all the things that the game has to do in a frame, it adds up *fast*.

Our efforts to solve this led us to try rebuilding the game in a different framework, but after that had its own problems (we'll get into that later, don't worry), we came back to DOTS and found a kludgey solution to sidestep the job scheduling woes.

### rendering

What's the proper shader model for a stylised game like this? Somewhere colourful, between cel-shading and photorealistic rendering. How do we render fur and grass? How do we keep the colours harmonised? How do we make bright lights pop without bloom? How can we get around the lack of shadows? What about fancy effects, like parallax cracks inside the icebergs of the Ice biome?

Every shader in this game is custom, and there's a lot to talk about. Especially...

### rendering water

You want a nice looking ocean? I really, *really* did. Games like <cite>Sea of Thieves</cite> have been wowing us with the state of the art FFT-based water simulation, creating incredibly natural looking oceans which respond naturally to boats and similar. We had to find another way, by precalculating the ocean in Blender and baking it to a flipbook texture---and shading it well is another matter again.

### visual effects

It's a game about blowing things up, so of course we want nice explosions. But with everything else our game is doing, our overhead for particle simulations is *very* limited. Nevertheless, we have nice rocket trails, clouds of smoke and burning debris. Which requires finding ways to harmonise DOTS with the world of GameObjects, and that led to all sorts of strange problems...

My colleague Alfon came up with all sorts of ingenious approaches to VFX, which we'll cover in the VFX article.

### animation

We are a very small team, but our hamsters must express all the charm a hamster should, as they swing about a huge variety of weapons. I *love* animation, and I wanted our animations to be expressive through all the animation principles: weight, squash and stretch, overshoot and settle, etc. etc.

But more than that, I strongly wanted physical continuity: the weapons should feel like objects with weight, that don't just magically appear and disappear, or magnetically cling to a hamster's back. At the same time, all these animations had to feel snappy, and not get in the way of gameplay---not to mention that our hamsters would have to run over uneven and unpredictable terrain. In an upcoming article I'll go through the different iterations of animations we hit along the way, and the system we finally settled on.

Also it had to run in DOTS, which *still* doesn't have a proper, first-party animation system (and won't until after Unity 6). Luckily, this is where third-party developers stepped up. We'll go into the different DOTS animation systems that currently exist.

### UI/UX

Or, how do you build a game for crabs?

How do you pick weapons for a hamster, lead her about the map, aim his shots and know which team they're on? How do you inform the players about game state, when your world is a 3D volume that could be viewed from any angle? What does a game setup screen look like? A weapon unlock?

Although VR games have been around for a good number of years now and conventions have emerged, the particular tactile style of our game demanded we find something a little new...

(We'll also talk about the calculations involved in creating a <cite>Demeo</cite>-like locomotion system, since that has rapidly become the standard for 'tabletop-style' VR games.)

### bots

Sometimes, nobody's online. Or you want a practice game. So the game has a full-featured AI player thanks to my colleague Bas, which can choose the best weapon to use, manoeuvre hamsters, shoot across the map and generally present a seriously challenging game to rival a human player---while also now and then making a charming mistake.

What does a fast entity-component-system AI look like? We'll get into the design in this article.

### and more...?

I mean, maybe more. We've got a lot to cover and we're still working on the game. But I'm seriously proud of what we've put together here, and I'd love to share some of what's going on under the hood of this thing. So, read on...