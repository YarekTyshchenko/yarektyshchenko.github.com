---
published: true
layout: post
comments: true
---


## Complexity

This is something that I have seen argued often and without a successful definition. When choosing a solution or an implementation there is an inherent draw towards something thats simpler.


Most people associate simplicity instinctively with several different things, two of the most common are size/quantity and understanding. The latter being rather ambiguous, difficult to define because level of understanding might be different for people with differing experiences.


When me and a colleague decided to sit down and distil what makes systems in general simpler, here is what we came up with, Excuse my poor drawing skills.


![Complexity Triangle](https://www.lucidchart.com/publicSegments/view/b3dc33b9-1b1f-4f4a-998c-ff02f00fffac/image.png)
What we call "Tyshchenko-Thumes Complexity Triangle". Hey, someones has to name this shit.


Failure modes per artifact: How many different ways communication with the interface of that artifact could fail? A function call can throw an exception. A TCP network call could fail, timeout, hang indefinitely, succeed right after a timeout, return unexpected garbage. A file read could block on IO-wait, etc.


Number of responsibilities per artifact: How many things does each artifact do? e.g. Have you accidentally baked a load balancer into your API Client?


Total number of artifacts: How many of instances there are in total. Perhaps you require several instances of your artifact to achieve HA. Or you choose another piece of software to provide a service instead of baking it into your code.


Looking at a bounded context holistically: A system that has many moving pieces, but all the pieces do only one thing and have limited number of failure modes each would in my opinion be simpler than a system made up of a few complicated "Master" services.


Conversely, a monolith that does only one thing, that unfortunately has to integrate with some awful API gateway that could fail in many unpredictable ways would be considered simple. All its failure modes should be easily testable and accounted for in a big list.

