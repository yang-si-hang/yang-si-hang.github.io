---
layout: about
title: about
permalink: /
subtitle: # <a href='#'>Affiliations</a>. Address. Contacts. Motto. Etc.

profile:
  align: right
  image: 
  image_circular: false # crops the image to make it circular
  more_info: >
    <p>HUAZHONG UNIVERSITY OF SCIENCE AND TECHNOLOGY</p>
    <p>Wuhan, China</p>
    
# <p>555 your office number</p>
# <p>123 your address street</p>
    

selected_papers: true # includes a list of papers marked as "selected={true}"
social: true # includes social icons at the bottom of the page

announcements:
  enabled: false # includes a list of news items
  scrollable: true # adds a vertical scroll bar if there are more than 3 news items
  limit: 5 # leave blank to include all the news in the `_news` folder

latest_posts:
  enabled: false
  scrollable: true # adds a vertical scroll bar if there are more than 3 new posts items
  limit: 3 # leave blank to include all the blog posts
---

I am a Ph.D. candidate in robotics, focusing on robot learning, vision-language-action models, and deformable object manipulation. My research combines imitation learning, differentiable simulation, model-based control, and real-world robotic system integration. Recently, I have been working on VLA post-training and continuous robot execution, with the goal of developing robust and generalizable policies for complex manipulation tasks.

<h2>
  <a href="{{ '/projects/' | relative_url }}" style="color: inherit">projects</a>
</h2>

{% assign selected_projects = site.projects | where: "selected", true | sort: "importance" %}
<div class="projects">
  <div class="row row-cols-1 row-cols-md-3">
    {% for project in selected_projects %}
      {% include projects.liquid %}
    {% endfor %}
  </div>
</div>

<!-- Write your biography here. Tell the world about yourself. Link to your favorite [subreddit](http://reddit.com). You can put a picture in, too. The code is already in, just name your picture `prof_pic.jpg` and put it in the `img/` folder.

Put your address / P.O. box / other info right below your picture. You can also disable any of these elements by editing `profile` property of the YAML header of your `_pages/about.md`. Edit `_bibliography/papers.bib` and Jekyll will render your [publications page](/al-folio/publications/) automatically.

Link to your social media connections, too. This theme is set up to use [Font Awesome icons](https://fontawesome.com/) and [Academicons](https://jpswalsh.github.io/academicons/), like the ones below. Add your Facebook, Twitter, LinkedIn, Google Scholar, or just disable all of them. -->
