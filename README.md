This directory contains the Joshua web site, including

- The Joshua decoder main web page
- The Indian language corpora (in indian-parallel-corpora/)
- [Jekyll](https://github.com/mojombo/jekyll/) code for generating the Joshua end user documentation

The main thing you might want to do (assuming you have write access to this repository) is to add
documentation pages.  This can be done in two steps:

1. Write your documentation using Github-supported Markdown or HTML.  Create the file in the current
   directory, using one of the existing files as templates.  The top of the file contains a number of
   lines specifying metadata.  The metadata looks like this:

    ---
    layout: default.html
    title:  My New Page
    ---
    Your content goes here.

   At minimum, you should specify the template to apply (relative to `_layouts`, probably
   default.html) and the page's title.  Everything below the second set of `---` is substituted into
   the template where `{{ content }}` is found.

1. Edit `_layouts/default.html`, which contains the template file used to host user documentation.
   You'll want to add a link to your page from the sidebar.

1. If you also want to edit the main documentation page, you can find that in the file `index.md`.
   This file is transformed by Jekyll and placed in `userdocs/` alongside everything else.

Note that if you're testing on your local machine, you'll need to install Jekyll.  You need to have
ruby installed.  Then type:

    gem install jekyll  # you might need to prepend 'sudo'

You can then type:

    jekyll --pygments --safe

to generate the user pages.  Do this within a web server and point a recent browser at it.  You can
also run your own minimal web server with Jekyll.
[This page](http://net.tutsplus.com/tutorials/other/building-static-sites-with-jekyll/) has a good
Jekyll tutorial.
