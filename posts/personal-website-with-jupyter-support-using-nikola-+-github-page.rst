.. title: Personal website with Jupyter support using Nikola and GitHub page
.. slug: personal-website-with-jupyter-support-using-nikola-and-github-page
.. date: 2019-02-17 18:23:21 UTC-05:00
.. tags: Nikola, Jupyter, GitHub
.. category: 
.. link: 
.. description: 
.. type: text

I decide to build my new website and occasionally post blogs on research, software, and computing.

For the blogging tool, basic requirements are:

- It must be a `static site <https://en.wikipedia.org/wiki/Static_web_page>`_ that can be easily version-controlled on GitHub. So, not WordPress.
- I'd like the framework to be written in Python. So, not Jekyll.
- It should support Jupyter notebook format, so I can easily post computing & data analysis results.

The options come down to `Pelican <https://github.com/getpelican/pelican>`_ (the most popular one), `Tinkerer <https://github.com/vladris/tinkerer>`_ (`Sphinx <https://github.com/sphinx-doc/sphinx>`_-extended-for-blog), and `Nikola <https://github.com/getnikola/nikola>`_. I finally settle down on Nikola because it is super intuitive, has a clear and user-friendly documentation, and supports Jupyter natively. It takes me like 10 minutes to set up the desired website layout. Other tools are also very powerful but it takes me longer to get the configuration right.

This post records my setup steps, to help others and the forgetful future me. Building such a website only requires basic knowledge in:

- Git and GitHub
- Python
- Markdown (works fine) or reStructuredText (preferred)

I assume no knowledge in Web front-end stuff (HTML/CSS/JS). This should be typical for many computational & data science students.

.. contents::
.. section-numbering::

Related posts
=============

Just by Googling you can find tons of tutorials on this. Particularly good ones are:

- http://www.jaakkoluttinen.fi/blog/how-to-blog-with-jupyter-ipython-notebook-and-nikola/
- http://louistiao.me/posts/how-i-customized-my-nikola-powered-site/
- https://post2web.github.io/posts/jupyter-blogging/
- http://sampathweb.com/posts/blogging-made-easy.html 

However this post is still useful because I:

- use nikola v8.0.1 as of Feb 2019. Many early tweaks are not necessary now.
- focus on personal website (with additional pages like ``Bio``), not only a blog.
- add additional notes on GitHub deployment and version control.

Installation and initialization
===============================

The `official getting-started guide <https://getnikola.com/getting-started.html>`_ explains this well. I just summarize the steps here.

I come from the scientific side of Python, so use Conda to create a new environment::

    conda env create -n blog python=3.6
    source activate blog
    pip install Nikola[extras]

Making the first demo site is as simple as

- Run ``nikola init`` in a new folder, with default settings.
- Then run ``nikola auto`` to build the page and start a local server for testing.
- Visit ``localhost:8000`` in the web browser.

Adding a new blog post is as simple as:

- Run ``nikola new_post`` (defaults to reStructuredText format, add ``-f markdown`` for ``*.md`` format)
- Write blog contents in the newly generated file (``*.rst`` or ``*.md``) in the ``posts/`` directory.
- Rerun ``nikola auto``. You can also keep this command running so new contents will be automatically updated.

All configurations are managed by a single ``conf.py`` file, which is extremely neat.

Before adding any real contents, you should first configure GitHub deployment, layout, themes, etc.

Github deployment and version control
=====================================

There is an `official deployment guide <https://getnikola.com/handbook.html#deploying-to-github>`_ but I feel that more explanation is necessary.

Most posts mention deployment at the final step, but I think it is good to do so at the very beginning.

We will use `GitHub pages <https://pages.github.com/>`_ to host the website freely. Any GitHub repo can have its own GitHub page, but we will use the special one called `user page <https://help.github.com/articles/user-organization-and-project-pages/#user-and-organization-pages-sites>`_ that only works for a repo named ``[username].github.io``, where ``[username]`` is your GitHub account name (mine is ``jiaweizhuang``). The website URL will be ``https://[username].github.io/``.

You will need to:

1. Create a new GitHub repo named ``[username].github.io``. 
2. Initialize a Git repo (``git init``) in your Nikola project directory.
3. Link to your remote GitHub repo (``git remote add origin https://github.com/[username]/[username].github.io.git``)

So far all standard git practices. The non-standard thing is that you shouldn't manually commit anything to the ``master`` branch. The ``master`` branch is used for storing HTML files to display on the web. It should be handled automatically by the command ``nikola github_deploy``. To version control your source files (``conf.py``, ``*.rst``, ``*.md``), you should create a new branch called ``src``::

    git checkout -b src

My ``.gitignore`` is::

    cache
    .doit.db*
    __pycache__
    output
    .ipynb_checkpoints
    .DS_Store


You can manually commit to this ``src`` branch and push to GitHub.

In ``conf.py``, double-check that branch names are correct:

.. code:: python

    GITHUB_SOURCE_BRANCH = 'src'
    GITHUB_DEPLOY_BRANCH = 'master'
    GITHUB_REMOTE_NAME = 'origin'

I also recommend setting:

.. code:: python

    GITHUB_COMMIT_SOURCE = False

So that the ``nikola github_deploy`` command below won't touch your ``src`` branch.

To deploy the content on ``master``, run::

    nikola github_deploy

This builds the HTML files, commits to the ``master`` branch, and pushes to GitHub. The actual website ``https://[username].github.io/`` will be updated in a few seconds.

You end up having:

- A well version-controlled ``src`` branch, with only source files. You can add meaningful commit messages like for other code projects.
- An automatically generated ``master`` branch, with messy html files which you never need to directly look at. It doesn't have meaningful commit messages, and the commit history is kind of a mess (diff between HTML files).

.. note::
    Remember that ``nikola github_deploy`` will use all the files in the current directory, not the most recent commit in the ``src`` branch! ``master`` and the ``src`` are not necessarily synchronized if you set ``GITHUB_COMMIT_SOURCE = False``.

For all the tweaks later, you can incrementally update the GitHub repo and the website, by manually pushing to ``src`` and using ``nikola github_deploy`` to push to ``master``.

Change theme
============

The default theme looks more like a blog than a personal website. Twitter's `Bootstrap <https://getbootstrap.com/>`_ is an excellent theme and is `built into Nikola <https://themes.getnikola.com/v8/bootstrap4/>`_.  In ``conf.py``, set:

.. code:: python

    THEME = "bootstrap4"

The theme can be further tweaked by `Bootswatch <https://bootswatch.com/>`_ but I find the default theme perfect for me :)

Set non-blog layout
===================

`Official non-blog guide <https://getnikola.com/creating-a-site-not-a-blog-with-nikola.html>`_ explains this well. I just summarize the steps here.

Nikola defines two types of contents:

- "Posts" generated by ``nikola new_post``. It is just the blog post and will be automatically added to the main web page whenever a new post is created.
- "Pages" generated by ``nikola new_page``. It is a standalone page that will not be automatically added to the main site. This is the building block for a non-blog site. 

In ``conf.py``, bring Pages to root level:

.. code:: python

    POSTS = (
        ("posts/*.rst", "blog", "post.tmpl"),
        ("posts/*.md", "blog", "post.tmpl"),
        ("posts/*.txt", "blog", "post.tmpl"),
        ("posts/*.html", "blog", "post.tmpl"),
    )
    PAGES = (
        ("pages/*.rst", "", "page.tmpl"),  # notice the second argument
        ("pages/*.md", "", "page.tmpl"),
        ("pages/*.txt", "", "page.tmpl"),
        ("pages/*.html", "", "page.tmpl"),
    )

    INDEX_PATH = "blog"

Generate your new index page (the entry of you website)::

    $ nikola new_page
    Creating New Page
    -----------------

    Title: index

To add more pages to the top navigation bar::

    $ nikola new_page
    Creating New Page
    -----------------

    Title: Bio

And then add it to ``conf.py``:

.. code:: python

    NAVIGATION_LINKS = {
        DEFAULT_LANG: (
            ("/index.html", "Home"),
            ("/bio/index.html", "Bio"),
            ...
        ),



Enable Jupyter notebook format
==============================

Just add ``*.ipynb`` as recognizable formats:

.. code:: python

    POSTS = (
        ("posts/*.rst", "blog", "post.tmpl"),
        ("posts/*.md", "blog", "post.tmpl"),
        ("posts/*.txt", "blog", "post.tmpl"),
        ("posts/*.html", "blog", "post.tmpl"),
        ("posts/*.ipynb", "blog", "post.tmpl"), # new line
    )
    PAGES = (
        ("pages/*.rst", "", "page.tmpl"),
        ("pages/*.md", "", "page.tmpl"),
        ("pages/*.txt", "", "page.tmpl"),
        ("pages/*.html", "", "page.tmpl"),
        ("pages/*.ipynb", "", "page.tmpl"), # new line
    )

With the current version (v8), that's all you need to do!

Create a new blog in notebook format::

    $ nikola new_post -f ipynb

Add social media button
=======================

You might want to add buttons for other sites like GitHub and Twitter, or any icons from `Font Awesome <https://fontawesome.com/>`_.

Taken from `this post by Jaakko Luttinen <http://www.jaakkoluttinen.fi/blog/how-to-blog-with-jupyter-ipython-notebook-and-nikola/>`_, the minimal example is (only one GitHub button):

.. code:: python

    EXTRA_HEAD_DATA = '<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/font-awesome/latest/css/font-awesome.min.css">'

    CONTENT_FOOTER = '''
    <div class="text-center">
    <p>
    <span class="fa-stack fa-2x">
    <a href="https://github.com/[username]">
        <i class="fa fa-circle fa-stack-2x"></i>
        <i class="fa fa-github fa-inverse fa-stack-1x"></i>
    </a>
    </span>
    </p>
    </div>
    '''

Various tweaks
==============


Add table of content
--------------------

For ``*.rst`` posts, simple add::

    .. contents::

Or optionally with numbering::

    .. contents::
    .. section-numbering::

Tweak Archive format
--------------------

To avoid grouping posts by years:

.. code:: python

    CREATE_SINGLE_ARCHIVE = True

The creation time of each blog post is displayed down to minutes by default. Only showing the date seems enough:

.. code:: python

    DATE_FORMAT = 'YYYY-MM-dd'

Enable comment system
---------------------

Because static sites do not have databases, you need to use a thiry-party comment system as documented on the `official doc <https://getnikola.com/handbook.html#comments>`_. The steps are:

1. Sign up for an account on https://disqus.com/.

2. On Disqus, select "Create a new site" (or visit https://disqus.com/admin/create/). 

3. During configuration, take note on the "Shortname" you use. Other configs are not very important.

4. At "Select a plan", choosing the basic free plan is enough.

5. At "Select Platform", just skip the instructions. No need to insert the "Universal Code" manually, as it is built into Nikola. Keep all default and finish the configuration.

In ``conf.py``, add your Disqus shortname:

.. code:: python

    COMMENT_SYSTEM = "disqus"
    COMMENT_SYSTEM_ID = "[disqus-shortname]"

Deploy to GitHub and the comment system should be enabled.
