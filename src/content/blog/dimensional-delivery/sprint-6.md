---
title: "Dimensional Delivery Sprint 6"
description: "QoL"
pubDate: "Nov 24 2024"
heroImage: "../../../assets/dimensional-delivery-backdrop.gif"
project: "dimensional-delivery"
---

Sprint six is done and over with and with it came the beta build of our game. 17 cards were completed this sprint which is the exact same amount from last sprint. The main focus of this sprint was quality of life features and the title screen.

Title screens are the first thing the player sees when they boot up your game, and thus it needs to stand out. What we went with is a very simple display of crates in transit going across a conveyor belt into a portal. I believe that conveys the design of the game perfectly and makes the goal of the game clear before even getting to the gameplay. I opted to make the animation using scripts as I wanted the flexibility to increase or decrease crate amount, crate speed, conveyor speed, and more.

Before getting into the QoL features I want to talk about the two shaders that were done during this sprint. One was a simple animated shader done in HLSL to animate the conveyor belt. The other shader was a quick optimization shader for the portals that allowed better portal graphics whilst also cutting down on resource cost as using the built-in material caused artifacts and was overall awful to work with. Before this shader, two materials had to be applied which cost a lot of resources.

Quality of life is a very crucial component of games and is something that I hold to a very high standard which is a double-edged sword. On one hand it means that I value player enjoyment and streamlining the gaming process; however, this makes me obses
s over every minute detail leading to wasted time and energy. The biggest QoL change made in this build was the introduction of a zoom in feature when you are placing portals which allows you to see where you are placing no matter what. This was very successful as one of our biggest complaints in previous playtests was not being able to see portal placement; however, in this playtest no-one brought it up at all and even said they had much more enjoyment. Other smaller QoL features include faster level loading, more tutorials, audio, render textures on portals (ability to see through them), and the ability to select which sub-level from a level to play.

Some bugs that were fixed during this sprint were portals not visually lining up in preview mode (when placed they were correct) and the end level screen not working properly. The preview mode glitch was completely visual and happens because the system gets confused when the player switches portal placement type in the settings. The end level screen had to be pretty much reworked completely to make it work in any scene instead of just the original creator's scene. I also redid the design of it to fit more in theme with the current UI as it looked a bit out of place.

Other work completed this sprint included displaying how many stars are earned on a certain level (and by extension diamond stars if all sub-levels are gold) and applying textures to levels. The display for how many stars has been earned was just making the data that is already there pop up on the UI with a nice to look at style.

This sprint went really well, and I will even say that even though we are just at beta if this is the final product, I'm perfectly content with it. More content would be nice and that is the main focus on sprint seven. Some sneak peeks into what I'm currently working on is writing an Android Library in Java to expose Android specific functions like vibration to get better control over a device's haptic motor. Another big thing will be the level editor, currently in its infancy stage, it will allow the level designer to design levels with more ease instead of using Unity's default tools. That is all for this sprint!
