---
layout: default
title: Georgi Kostov
header_title: Georgi Kostov
description: "Senior Unity developer, XR specialist, educator, and technical writer building games, simulations, digital twins, and AI-powered products."
image: /assets/images/social/georgi-kostov-social.png
---

<div class="profile-container">
  <picture>
    <source srcset="{{ '/assets/images/me-400.webp' | relative_url }} 1x, {{ '/assets/images/me-800.webp' | relative_url }} 2x" type="image/webp">
    <img src="{{ '/assets/images/me.jpeg' | relative_url }}" alt="Georgi Kostov" class="profile-image" width="200" height="200" fetchpriority="high" decoding="async">
  </picture>

  <div class="profile-info">
    <h1 class="profile-name">Georgi Kostov</h1>
    <p class="profile-tagline">Senior Unity Developer · XR Specialist · Educator</p>

    <div class="hero-actions">
      <a href="{{ '/pages/projects/' | relative_url }}" class="primary-action">View selected work</a>
      <a href="mailto:georgikostov1337@gmail.com" class="secondary-action">Get in touch</a>
    </div>

    <div class="social-links" aria-label="Social profiles">
      <a href="https://www.linkedin.com/in/kostovg/" target="_blank" rel="noopener noreferrer" aria-label="LinkedIn">
        <svg class="social-icon" viewBox="0 0 24 24" aria-hidden="true"><path d="M20.45 20.45h-3.56v-5.57c0-1.33-.02-3.04-1.85-3.04-1.85 0-2.14 1.45-2.14 2.94v5.67H9.34V9h3.42v1.56h.05c.47-.9 1.64-1.85 3.37-1.85 3.61 0 4.27 2.37 4.27 5.46v6.28ZM5.32 7.43a2.07 2.07 0 1 1 0-4.13 2.07 2.07 0 0 1 0 4.13ZM7.1 20.45H3.54V9H7.1v11.45Z"/></svg>
      </a>
      <a href="https://x.com/KostovSolutions" target="_blank" rel="noopener noreferrer" aria-label="X">
        <svg class="social-icon" viewBox="0 0 24 24" aria-hidden="true"><path d="M18.24 2.25h3.31l-7.23 8.26 8.5 11.24h-6.66l-5.21-6.82-5.97 6.82H1.67l7.73-8.84L1.25 2.25h6.99l4.71 6.23 5.29-6.23Zm-1.16 17.52h1.83L7.24 4.13H5.28l11.8 15.64Z"/></svg>
      </a>
    </div>
  </div>
</div>

Since 2015, I have worked on games and simulations across XR, education, climate change, agriculture, logistics, digital twins, city planning, sport, health and civil courage. I am a senior Unity developer with experience across every stage of development—concept, design, programming, UX, graphics, publishing and analytics. I also lecture and mentor several student projects each year.

<div class="link-cards">
  <a href="{{ '/pages/projects/' | relative_url }}" class="link-card">
    <h3>Projects</h3>
    <p>Explore my work in XR, games, and simulations.</p>
  </a>
  <a href="{{ '/pages/blog/' | relative_url }}" class="link-card">
    <h3>Blog</h3>
    <p>Read articles and insights on technology and design.</p>
  </a>
  <a href="{{ '/pages/publications/' | relative_url }}" class="link-card">
    <h3>Publications</h3>
    <p>Discover my research and academic contributions.</p>
  </a>
</div>

<p class="about-note">Outside work, I enjoy traveling, cycling, hiking, swimming, games and film criticism—interests that regularly find their way back into my projects.</p>

---

## Professional Timeline
<ul class="timeline">
  {% for item in site.data.timeline limit:4 %}
    <li class="timeline-item">
      <p class="timeline-date">{{ item.duration }}</p>
      <div class="timeline-content">
        <h3>{{ item.role }}</h3>
        <p><strong>{{ item.title }}</strong></p>
        <ul>
          {% for point in item.description %}
            <li>{{ point | strip }}</li>
          {% endfor %}
        </ul>
      </div>
    </li>
  {% endfor %}
</ul>

<p><a href="{{ '/output/pdf/Georgi_Kostov_CV_2026.pdf' | relative_url }}" class="text-action">View the full CV <span aria-hidden="true">→</span></a></p>

---

## Teaching
I teach courses on games with a purpose, mixed reality and location-based technologies at the [University of Applied Sciences Upper Austria, Campus Hagenberg](https://fh-ooe.at/campus-hagenberg). The courses include a theoretical component, where core concepts from XR and game design are explored, and a practical component, where students develop projects throughout the semester under my mentorship. I also lead a making games with Unity workshop for the Master’s program in Interface Cultures at the [University of Art and Design Linz](https://www.kunstuni-linz.at/en/studies/degree-programmes/interface-cultures/master-programme/courses).

### Student Projects 2023/2024
{% include youtube-lite.html id="11QtNfz-3rc" title="Student games project showcase" %}
