---
layout: default
title: Home
---

<div class="profile-container">
  <img src="assets/me.jpeg" alt="Georgi Kostov" class="profile-image" />

  <div class="profile-info">
    <h1 class="profile-name">Georgi Kostov</h1>
    <p class="profile-tagline">Unity Developer | Lecturer | XR Enthusiast</p>

    <div class="social-links">
      <a href="https://www.linkedin.com/in/kostovg/" target="_blank" aria-label="LinkedIn">
        <img src="assets/icons/linkedin.png" alt="LinkedIn" class="social-icon" />
      </a>
      <a href="https://x.com/KostovSolutions" target="_blank" aria-label="Twitter">
        <img src="assets/icons/twitter.png" alt="Twitter" class="social-icon" />
      </a>
      <a href="mailto:georgikostov1337@gmail.com?subject=Project%20Inquiry" aria-label="Mail">
        <img src="assets/icons/mail.png" alt="Mail" class="social-icon" />
      </a>
    </div>
  </div>
</div>

In my professional experience I have worked on games and simulations in a variety of fields: XR, education, climate change, agriculture, logistics, digital twins, city planning, sport, health and civil courage. I am an experienced Unity3D developer with 10 years of practice in every stage of game development – concept, design, programming, UX, graphics, publishing, analytics. In addition, I have recently embraced the role of lecturer, guiding and mentoring several student projects each year.

In my free time I love traveling, cycling, hiking, swimming, spending time with friends and family. I am an avid gamer, sometimes write film critique and always aim to learn something new about the world, which I can reflect in my projects.

<a href="/pages/projects/" class="button">Explore Projects</a> <a href="/pages/blog/" class="button">Read My Blog</a> <a href="/pages/publications/" class="button">View Publications</a> 

---

## Professional Timeline
<div class="timeline">
  {% for item in site.data.timeline %}
    <div class="timeline-item">
      <div class="timeline-dot">
        <span class="timeline-time">{{ item.duration }}</span>
      </div>
      <div class="timeline-details">
        <h3>{{ item.title }}</h3>
        <p><strong>{{ item.role }}</strong></p>
        <ul>
          {% for point in item.description | split: '\n' %}
            <li>{{ point }}</li>
          {% endfor %}
        </ul>
      </div>
    </div>
  {% endfor %}
</div>


## Teaching


I teach courses on games with a purpose, mixed reality, and location-based technologies at [University of Applied Sciences Upper Austria, Campus Hagenberg](https://fh-ooe.at/campus-hagenberg). The courses include a theoretical component, where core concepts from XR and game design are explored, and a practical component, where students develop projects throughout the semester under my mentorship. I also lead an introduction to making games with Unity workshop for the Master’s program in Interface Cultures at the [University of Art and Design Linz](https://www.kunstuni-linz.at/en/studies/degree-programmes/interface-cultures/master-programme/courses).

### Student Projects 2023/2024
<iframe width="560" height="315" src="https://www.youtube.com/embed/11QtNfz-3rc?si=LaHVU8-6pop8pJYK" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>