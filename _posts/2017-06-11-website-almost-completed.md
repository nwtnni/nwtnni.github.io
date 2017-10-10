---
layout: post
title: Creating a Website
date: 2017-06-11
description: A brief overview of how I created my website.
category: [project, site]
---

Aside from filling out the Contact page and deciding how
to structure and display projects, this website is almost
done! I've learned a lot about Jekyll and CSS in the process.
Thank goodness for Chrome developer tools.

I've been meaning to start a personal website for a while now,
but haven't had the time to learn web development until this
summer. Github Pages seemed like a good place to host mine, despite
its limited Jekyll plugins and inherently static nature.

##### **CSS**

Instead of using [Bootstrap](https://v4-alpha.getbootstrap.com/),
I opted for a much lighter CSS framework called 
[Skeleton](http://getskeleton.com/). I figured I wouldn't need
most of Bootstrap's fancier features, and Skeleton offered a
solid foundation for me to build on. I could tack on my own .css
files without worrying (for the most part) about removing 
complicated existing styles.

Then came a whole lot of research as I delved into CSS style guides
and SASS tutorials. [AirBnB's style guide](https://github.com/airbnb/css)
was useful, as was looking at the main Skeleton.css file. 
Most of it was familiar: code reuse, consistent
formatting, modular design. I followed 
[this guide](http://thesassway.com/beginner/how-to-structure-a-sass-project)
for structuring my directories--which was maybe a little excessive
for such a small-scale project. Here's my main stylesheet directory:

```
_sass
├── modules
│   ├── _all.scss
│   ├── _colors.scss
│   ├── _media.scss
│   └── _variables.scss
├── partials
│   ├── _base.scss
│   ├── _hero.scss
│   ├── _navbar.scss
│   ├── _post.scss
│   └── _utility.scss
└── vendor
    ├── normalize.scss
    └── skeleton.scss
```


Most of my files are tiny. 
But at least everything's extensible, right? And the cool thing about
SASS is that it processes all of the partials into one main file.

##### **HTML**

Armed with some basic knowledge of CSS, I set about writing the HTML.
Github Pages uses [Jekyll](https://jekyllrb.com/) to generate static sites, 
which in turn uses [Liquid](https://shopify.github.io/liquid/) to process 
template files. I had to refer to the documentation quite a bit.

In essence, Jekyll turns content-only files like this one into complete
web pages, which is perfect for blogging since you have a lot of content
(i.e. posts) which all have the same page layout. You just have to write
the corresponding templates, and Jekyll processes the posts into full pages.

So I wrote one default template that `includes` files for the head,
header, and footer. The body is filled in with a separate template for 
each page layout:

```
_layouts/
├── about.html
├── contact.html
├── default.html
├── home.html
├── post.html
├── posts.html
├── project.html
└── projects.html
```

All with pretty self-explanatory filenames. Because I'm not sure how I'll
display projects yet, the code is essentially a duplicate of the post-related
pages. That may or may not change in the future.

##### **Styling**

Basically a lot of referring to [MDN](https://developer.mozilla.org/en-US/)
and poking around with Chrome's CSS inspector. Since I don't have any 
graphic design experience, I stuck to trial and error, iterating until I had
something that looked presentable.

##### **Responsive Design**

Since Skeleton was created with mobile devices in mind, I had to write
very little intentionally responsive CSS. I used 
[media queries](https://developer.mozilla.org/en-US/docs/Web/CSS/Media_Queries/Using_media_queries)
in three places to deal with small screen sizes:

- Social media buttons collapse to icon-only
- Post date displays go from right-aligned to left-aligned
- Profile background image disappears

Which are all pretty small changes, but I guess it doesn't hurt
to start good habits.

##### **Category Sorting**

Since Github Pages doesn't really support dynamic sites, I couldn't implement
a search bar or category filter on the Posts page. However, I did find
[this post](https://www.chrisanthropic.com/blog/2014/jekyll-themed-category-pages-without-plugins/),
which details a roundabout method of getting Jekyll to generate
category-specific pages. So I adjusted the code a little and
incorporated it into this site.

##### **Conclusion**

This has been a very brief description of how I built the site, but I hope
it'll suffice. It's not quite done yet--even so, it's been a great learning
experience, and hopefully a forerunner to many more projects in the future.
