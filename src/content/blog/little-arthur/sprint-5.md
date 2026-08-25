---
title: "Little Arthur Sprint 5"
description: "Animation and User Interface"
pubDate: "Apr 09 2026"
heroImage: "../../../assets/little-arthur-image.png"
---

Sprint five is complete and with it came some nice additions to the project. This sprint had a bit less work complete; however, our game is in a really good state right now and I was given far less work to complete. During this sprint I completed eight cards which most related either to animation or UI updates.

The first thing that was done this sprint was the shop. The shop as it stands is just a simple boolean flag on an item spawner that instead creates the item as a shop item that costs gold instead of an item that the player can just pick up. The next thing after that was making a respawn anchor to allow players to revive other players. It basically just acts a simple interactable, but when it is activated, it takes 50% of the player's health to give to a randomly revived ally. That ally will then get teleported near the anchor itself, so they are in frame.

The last thing that wasn't animations or UI for this sprint was creating a border around the camera, so the players are always in frame. For the system I made it uses current screen width and height so all aspect ratios will work and basically just has four cubes that get positioned at each of the camera borders (one for forward, back, left, and right).

After this a bunch of animation work was completed. A lot of this stuff was more or less just importing the animation and then setting it up in the animator window; however, a couple animations like attacking and idle breaking did require some extra programming work. The attack animation needed some extra work as it needed to be converted to being event driven. For this I basically fire an event on the animator controller with an animation event so that when the swing looks like it has contact then the attack would actually fire. For idle breakers a similar, but simpler system was made where the animation event on the idle iterates an integer on the controller and when the number is greater than the desired idle break, it will play the idle break animation.

The last thing other than bug fixing that was done during this sprint was creating the user interface. I was kind of added onto completing the UI late because another team member was struggling with making it all work. For the UI I made the normal player HUD and the popup system. All the UI elements are fairly simple with just some data bindings to help keep things simple.

Overall, this sprint was a lot slower; however, I struggled greatly to receive cards as my designer and producer were fairly happy with where we are now. For the next sprint all that is lined up is sound implementation and bug fixing! Thanks for reading!
