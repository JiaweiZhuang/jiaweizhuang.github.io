.. title: Personal website with Jupyter support using Nikola and GitHub page
.. slug: personal-website-with-jupyter-support-using-nikola-and-github-page
.. date: 2019-02-17 18:23:21 UTC-05:00
.. tags: Nikola, Jupyter, GitHub
.. category: 
.. link: 
.. description: 
.. type: text

.. contents::
.. section-numbering::

Related posts
=============


Uses nikola v8 as of Feb 2019::

    $ nikola --version
    Nikola v8.0.1

Choosing the software
=====================

Compare with dynamic site: WordPress

Compare with Other languages: Ruby. 

Compare with other python tools: Pelican, [Tinkerer](http://tinkerer.me/index.html)

I find Nikola very east to learn, has user-friendly documentation, has native Jupyter support.


Installation and initialization
===============================

``nikola init``

``nikola auto``

``nikola clean``


Github deployment and version control
=====================================

Most posts mention deployment at the last but I feel it necessary to mention it at the very beginning.

- A well version-controlled ``src`` branch, with only core files and meaningful commit messages.
- A automatically generated ``master`` branch for deployment. With messy html files but you never need to care. With only `auto commit` messages and comprehensible commit history.

Caveat: The deployment branch will use all the files on the current directory, not the most recent commit in the ``src`` branch!

Change theme
============

Use Twitter's well-known boostrap4. Can tweak theme with bootwathc but I find the default theme perfect for me.


Set non-blog layout
===================

Well-documented in official tutorial.

Emphasize the difference between `post` and `page`: Unlike `new_post`, `nikola new_page` will not be indexed unless you add it]
Move to blog posts to `blog` folder

Add social link button
======================

Steal from



Enable Jupyter notebook format
==============================



Various tweaks
==============

Add table of content
--------------------


Enable comment system
---------------------


Archive list format
-------------------