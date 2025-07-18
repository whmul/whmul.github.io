---
layout: page
title: About
permalink: /about/
weight: 3
---

# **About Me**

I am **{{ site.author.name }}**, and welcome to this portfolio.<br><br>
I'm passionate when it comes to creating and building cool things.<br>My skills range from software to hardware, back end to front end, design and logic. Full stack, full tilt, full generalist.<br>Always looking for the next skill to learn and project to build. Every project is a chance to push myself, experiment, and bring something new into the world. Never stop learning.

<div class="row">
{% include about/skills.html title="Programming Skills" source=site.data.programming-skills %}
{% include about/skills.html title="Other Skills" source=site.data.other-skills %}
</div>

<div class="row">
{% include about/skills.html title="Other Relevant Experience" source=site.data.experience %}
</div>

<div class="row">
{% include about/timeline.html %}
</div>