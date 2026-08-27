---
title: "Little Arthur Sprint 6"
description: "User Experience"
pubDate: "Apr 23 2026"
heroImage: "../../../assets/little-arthur-image.png"
project: "little-arthur"
---

Sprint six is over and with it came much more content added to the game! This sprint I completed 13 cards with a total of 15 points. A lot of bug cards were also moved as well getting ready for the beta release of the game. This sprint was mostly focused on quality-of-life changes along with sound implementation.

The first thing completed this sprint was programming a small intro for the game that immerses the player through a drawbridge dropping down when all players are ready. It is a very simple trick that just combines the Unity animator with the custom script that I made a couple sprints ago to allow players to ready up.

The next half of this sprint was focused on implementing all of the sound assets we have accumulated over the last few sprints as our sound engineer was not available to integrate them with Wwise yet. Most if not all sound implementations were just as simple as defining a Wwise event reference and posting it when the general event is passed. I mostly focused on gameplay sounds as our other programmer focused on UI sounds. Playing the game with most sounds implemented completely changes the immersion the player experiences.

The next thing worked on, and the coolest in my opinion, was the shader to cull a circle around the player if an object is in the way. It extends the base Unity Lit shader with the added functionality of culling. It takes an array of player positions and loops through that to determine whether or not the current UV world space position would be blocking the camera. If it is, we will cull that pixel using the handy cull() function call in HLSL. After all UV positions are checked I then apply a nice little border around the circular hole the culling shader just culled. What is neat about this shader is that since it just extends the base Lit shader, all that needs to change is the base shader of the material and all bound textures are automatically rebound to the correct parameter. Since this uses UV positions it looks perfectly fine, but if you just slightly rotate the camera, it will look offset. This shader was something that I proposed to my lead designer around sprint 2, so I am very glad that I got the opportunity to work on it!

The next thing worked on was general bug fixing to get ready for our beta release. Overall, the game is in a really good state but there are definitely a few bugs that needed to be ironed out, especially ones that were found from our alpha build which was a lot! During this sprint I was able to fix all bugs on the bug board along with one bug that has evaded us for four sprints!

The last thing worked on this sprint was some quality-of-life features. I noticed that our tooltip system didn't support accurate displaying of keybinds. This is a problem as labelling a keybind as "button west" is not acceptable and it will take players more time to figure out what button west means in the context of this game. This did require a complete system rewrite to dynamically build UI elements at runtime (instead of just editing a text element).

Overall, this sprint went very well! I think if we just add a bit more polish in the upcoming sprint 7, we will have an incredible game to show off! Thank you for reading!
