---
layout: page
title: Projects
icon: fas fa-code
order: 5
permalink: /projects/
---

A mix of research platforms, open-source tools, and side experiments.

<div class="projects-grid">
  {% for project in site.data.projects %}
  <div class="project-card">
    <div class="project-card-header" style="background: {{ project.color }};">
      <i class="fas {{ project.icon }}" aria-hidden="true"></i>
      <span class="domain-badge">{{ project.domain }}</span>
    </div>
    <div class="project-card-body">
      <h3 class="project-title"><a href="{{ project.url }}">{{ project.title }}</a></h3>
      <p>{{ project.description }}</p>
      <div class="project-links">
        {% for link in project.links %}
          {% unless forloop.first %} · {% endunless %}<a href="{{ link.url }}">{{ link.label }}</a>
        {% endfor %}
      </div>
    </div>
  </div>
  {% endfor %}
</div>
