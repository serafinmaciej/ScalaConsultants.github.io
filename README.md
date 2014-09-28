#ScalaConsultants.github.io

##Scalac blog

###Overview
This blog uses [Jekyll](http://jekyllrb.com/) with [kramdown](http://kramdown.gettalong.org/syntax.html) parser. Please make sure you know the syntax for [kramdown](http://kramdown.gettalong.org/syntax.html) as there are some differences to the Maruku (parser you're probably used to). 

###To run it locally:

1. Install [Jekyll](http://jekyllrb.com/) for GitHub pages following instructions from here: https://help.github.com/articles/using-jekyll-with-pages
2. `bundle exec jekyll serve` (or `jekyll serve` might also work)
3. browse on http://localhost:4000/

###Mind the Tags
GitHub/Jekyll combo doesn't really allow for making too dynamic pages. The Tags pages have to be re-generated locally and copied to the project everytime you add a new post or change the tags. 

To generate completely fresh set of Tags pages:

1. Clear the _site directory.
2. Clear the tags directory.
3. Run the project as usual e.g. `jekyll serve` (or just build it using `jekyll build`).
4. Copy everything from _site/tags to tags directory.