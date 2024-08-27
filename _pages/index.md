---
layout: page
title: Home
id: home
permalink: /
---

# Welcome!

This is the <em>n</em>th, and maybe final (ha!), iteration of my personal website. (I am not related to [George Pollard,](https://en.wikipedia.org/wiki/George_Pollard_Jr.) the captain of the ill-fated whaleship <cite>Essex</cite>, but I have read <cite>Moby Dick</cite>.)

This time around my personal site takes the form of a collection of notes rather than attempting (and failing) to function as a blog.


Some entry points:

- <a class="internal-link" href="interests">My (Current) Interests</a>
- <a class="internal-link" href="publications">My (Few) Publications</a>

## Elsewhere

My other sites are:

- [Ways to Play](https://games.porg.es/), a site about traditional board and card games

You can also find me on:

- Bluesky: [@porg.es](https://bsky.app/profile/porg.es)
- [Goodreads](https://www.goodreads.com/user/show/4220815-george-pollard)
- Mastodon: [@porges@toot.cafe](https://toot.cafe/@porges)
  - note that I do not check this very often

### Previously

I was previously on these sites but no longer use them:

- Twitter: [@porges](https://twitter.com/porges)

## Recently updated

<ul>
  {% assign recent_notes = site.notes | sort: "last_modified_at_timestamp" | reverse %}
  {% for note in recent_notes limit: 5 %}
    <li>
      {{ note.last_modified_at | date: "%Y-%m-%d" }} — <a class="internal-link" href="{{ site.baseurl }}{{ note.url }}">{{ note.title }}</a>
    </li>
  {% endfor %}
</ul>

<style>
  .wrapper {
    max-width: 46em;
  }
</style>
