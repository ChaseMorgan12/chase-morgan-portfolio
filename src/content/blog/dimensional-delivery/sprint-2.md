---
title: "Dimensional Delivery Sprint 2"
description: "Gel Systems"
pubDate: "Sept 28 2024"
heroImage: "../../../assets/dimensional-delivery-backdrop.gif"
project: "dimensional-delivery"
---

Sprint two has passed and it is time to reflect on how well this sprint has gone. This sprint went fairly well; however, one card during the sprint proved to be quite troublesome. That card was developing the underlying system to allow gel to be placed across the levels. As the lead programmer, this sprint was mostly focused on developing the second most substantial element for our game, gel.

Gel proved to be harder than I thought it would be. The general idea of placing planes on top of other planes is not hard, but having to account for numerous cases outside of just placing it on top of planes is. Those other cases include wall placement, teleportation, collision, getting rid of gel that is in the same area, and gel combining. Each element adds complexity which turned a relatively easy card into a three point one with many late-night programming sessions. It also did not help the numerous times it had to be rewritten completely as the way it was being programmed would never be able to allow required elements. Although, what came out of it was a fairly robust gel placement system that takes into account many things that the levels will throw at it.

To implement the base gel system, I had to completely rewrite the base plane projection system that was used originally for the portal placement system. Doing this allows me to use the same system for anything that needs to be placed on any surface. Redesigning it allowed me to take in account way more cases that the gel would need to take in account in order to place properly. Overall, it turned out very well and it is way more robust which means it doesn't break as often.

Other than the base gel mechanic taking a while to implement this sprint went very well with most cards assigned being completed by the end of the sprint. The two cards that weren't finished were not high priority in the sprint, so they were left untouched as time ran out before being able to complete them. Eight cards were completed with a total point count of ten.

Other than gel, the other thing that was worked on during this sprint was cube projection for portals. This opens up the opportunity to give the level designers more freedom in level creation instead of only having the walls available to them for portals. This involved a lot of reworking the original portal projection scripts but was not too hard to implement in a timely manner.

The base gel mechanic was not all that was done with the gel during this sprint of which most functionality was actually implemented during sprint two. The two gels that were programmed during the sprint were bouncy and slippery gel. Bouncy gel will invert the velocity (with some total velocity loss) and make the crate bounce to clear larger gaps in non-portal areas. Slippery gel will make the crate lose all friction with the world which will allow level designers to make levels with more restrictive portal placement. Another thing that was added was the ability to apply any gel to the crate itself which can give more creative freedom to the level designers.

Overall, this sprint went very well, and the game is almost ready for its first internal playtest. The next sprint will mostly be used for adding miscellaneous features that are needed to make the game feel smoother and the rest for fixing issues with the current gel and portal system.
