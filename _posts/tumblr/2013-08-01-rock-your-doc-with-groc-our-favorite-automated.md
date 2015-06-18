---
layout: post
title: Rock Your Doc With Groc, Our Favorite Automated Frontend Documentation Tool
date: '2013-08-01T16:23:00-04:00'
tags:
- documentation
- automation
- frontend
- javascript
- coffeescript
- handlebars
- css
- less
- grunt
- groc
- dailyjs
tumblr_url: http://tech.gilt.com/post/57089759513/rock-your-doc-with-groc-our-favorite-automated
---
Writing great documentation is not something that comes naturally or easily to many engineers. And many companies don’t have the resources to hire a dedicated document writer. Hence the value of automated documentation, which allows you to generate, at a minimum, a basic amount of documentation quickly and cheaply.
As a front-end engineer working on UI architecture, I write code that ends up being used by most of the front-end engineers at Gilt—which makes automated documentation pretty important to me. A few years ago, some of my colleagues and I checked out a variety of automated documentation tools, including JSDoc, jGrouseDoc, YUIDoc, DocumentJS, and Docco, to see if any of them both looked pretty and supported JavaDoc-style Tags. We liked Docco’s clean look, inclusion of source code alongside documentation, and acceptance of markdown, but it didn’t support Tags. JSDoc supports Tags, but we didn’t like its appearance. We just couldn’t find a tool that gave us everything we needed. In the end, we created our own automated documentation tool by forking Docco and adding (hacking) support for Tags.
More recently, we conducted another survey of automated documentation tools and discovered Groc. Created by Ian MacLeod, Groc is a fork of Docco, and better than the alternatives in many ways. But Groc, like its grandparent Docco, offers no support for Tags—or block comments (WTF?)—so we forked it, too, and added (without hacking, this time) what we felt was missing. The end result is awesome. Go check it out on GitHub.
But wait—what are these “Tags" you speak of?
You’ve probably seen Tags in block comments. They begin with an @ sign—for example, @public or @method foo or @param {String[]} names A list of names.
A doc comment usually has multiple Tags as well as some free-form text, like this:
/**
 * Sets a cookie given a value of any type.
 *
 * @method    set
 * @public
 *
 * @param     {String}   name               The name of the cookie to be set
 * @param     {Mixed}    value              The value to convert to string and set in the cookie
 * @param     {Object}   [options]          Options hash
 * @param     {Mixed}    [options.expires]  The expiration as a number of seconds, or "session", or undefined for one year
 *
 * @return    {Boolean}                     Whether or not the cookie was successfully set
 *
 * @example
 *   cookie.set(''foo'', ''bar'', { expires : 1000000 });
 *   cookie.set(''foo'', [1, 2, 3], { expires : ''session'' });
 *   cookie.set(''foo'', { bar : ''baz'', boom : ''boosh'' });
 */

The structure that Tags provide makes writing documentation much easier for people who aren’t that great at it. The above sample is pretty decent; if we process it using Groc (our fork), we get something even better:

Documenting JavaScript is nice, but what about CSS? Or Handlebars?
Absolutely! I don’t have any amazing examples of CSS/LESS/Handlebars documentation, but it should work exactly the same as for JS/CoffeeScript. Also, you don’t have to use Tags. Here’s a Handlebars template that includes some “ordinary" comments:

OK, that’s cool. But you said something about automating this?
Having a tool that can generate documentation is all well and good, but you have to remember to actually run the tool. There’s always the risk of forgetting this step. So how about we integrate it into your build process?
Let’s assume you use Grunt, a front-end build tool that has a configuration file (Gruntfile.coffee) that enables us to define tasks. (Alternatives to Grunt include Smoosh, Gear and Buildr.) Let’s create a custom task “doc”:
grunt.registerTask ''doc'', ''Generate documentation'', ->
  done = this.async()
  grunt.log.writeln(''Generating Documentation...'')
  require(''child_process'').spawn(''./node_modules/.bin/groc'', [''lib/*.js'', ''README.md'']).on ''exit'', ->
    grunt.log.writeln(''...done!'')
    done()

Let’s integrate doc into our build task:
grunt.registerTask ''build'', [''test'', ''clean:build'', ''doc'', ''concat'', ''replace'', ''uglify'']

Then, whenever you grunt build, you get freshly generated documentation!
How do you guys use this at Gilt?
We have lots of small JS/CSS modules at Gilt, and they all get published to one place: a private npm registry. Our documentation server uses Groc to generate documentation for all modules that get published. Both Groc and this server use the same stylesheets, so they look the same and provides a seamless experience.
Here’s our Table of Contents, generated by our documentation server, which lists all modules, their descriptions, current versions, code coverage percentages (courtesy of Istanbul), dependencies, and has links to the groc-generated documentation (including past versions):

And a sample of what Groc generates:

What comes next?
We’ll continue to tweak Groc to add coverage to our automated documentation tool and show which lines of code have been tested or not. We’d also like to build in a search feature that will access multiple modules. Have some suggestions on other features we should add? Email me at kdavis at gilt dot com. I’d also like to hear from anyone who uses our work—let us know if it was helpful.
Note: Thanks to DailyJS for citing this post.