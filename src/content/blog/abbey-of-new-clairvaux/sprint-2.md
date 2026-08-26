---
title: "Abbey Of New Clairvaux AR App Tour Sprint 2"
description: "Setting up project"
pubDate: "Mar 29 2026"
heroImage: "../../../assets/ar-tour-backdrop.jpg"
---

Sprint two is over and with it came a fair amount of work on the project. Unfortunately, this sprint was only one week in length total so there was less time to complete work than other sprints. This sprint I completed around 6 cards all related to the POI system for the project.

The POI system is a relatively straight forward state system that holds data related to tasks and is spawned at runtime from pre-defined data assets that the designer creates. It automatically will bind to its UI Toolkit element using the built-in binding system that UI Toolkit offers. This way whenever the POI updates its state, the UI matches to reflect that! Another thing completed this sprint related to the POI system was the tracking and info display feature.

<figure class="gif-card">
  <img src="/gifs/Abbey_POI_Completion.gif" alt="POI Completion" />
  <figcaption>POI Completion</figcaption>
</figure>

When a user clicks on the POI it should open up and display a nice little menu showing the POI and the current task completion compared to the total task amount. It is tracked using UI Toolkit bindings and the menu itself is a child of the POI container as we need the menu to follow wherever the POI goes so this would be the fastest method to do so! Now when the user double clicks the POI should be tracked which marks it as blue.

None of what was created unfortunately has any way of working as it is just data bindings under the hood that would need actual implementation to work in the future. Overall, this sprint did go fairly slowly and if we want a product we will have to put in some extra hours to get more done. Thanks for reading!

<figure class="gif-card">
  <img src="/gifs/Abbey_POI_Tracking.gif" alt="POI Completion" />
  <figcaption>POI Tracking</figcaption>
</figure>
