#ScalaConsultants.github.io

##Scalac blog

###Overview
This blog uses [Jekyll](http://jekyllrb.com/) with [kramdown](http://kramdown.gettalong.org/syntax.html) parser. Please make sure you know the syntax for [kramdown](http://kramdown.gettalong.org/syntax.html) as there are some differences to the Maruku (parser you're probably used to). 

###To run it locally:

1. Install [Jekyll](http://jekyllrb.com/) for GitHub pages following instructions from here: https://help.github.com/articles/using-jekyll-with-pages
2. `bundle exec jekyll serve` (or `jekyll serve` might also work)
3. browse on http://localhost:4000/

###Mind the Tags
GitHub doesn't really allow making too dynamic pages, so the Tags pages have to be generated locally and copied to the project. 

Everytime you add new post or change tags, you have to run the server locally and copy generated _site/tags directory to the main project directory.