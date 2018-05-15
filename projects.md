---
layout: page
title: My Projects
---

<div class="case-studies-body">
    <ul class="listing">
        {% assign projects = site.projects | sort: 'listing-priority' %}
        {% for project in projects %}
        <li>
            <h2><a href="{{ project.projectUrl }}">{{ project.title }}</a></h2>
            {{ project.description }}
        </li>
        {% endfor %}
    </ul>
</div>
