---
layout: "post"
title: "Exploring Rails 5.1 with Webpacker"
tags: []
---

I just had a heck of a time trying to figure out how to setup [Rails 5.1 with
Webpacker](http://weblog.rubyonrails.org/2017/2/23/Rails-5-1-beta1/). As of
March 25th, 2017, here's how I got up and running:

I went ahead and started my repo, `little_library`, with the latest release. As
of today, that is Rails 5.0.2. I found [this blog
post](https://medium.com/statuscode/introducing-webpacker-7136d66cddfb#.rowi31wov)
to have instructions that looked pretty clever, getting up and running with
edge-rails, but it didn't quite work for me. So instead of debugging I decided
to just ram my skull into the problem as hard as I could. (This occassionally
works for me.) Still, the explanations in the the aforementioned blog post are
worth reading.

So I just went in, `rails new`ed me a new Rails 5.0.2 application, and updated the
Gemfile:

```
gem 'rails', github: 'rails/rails'
```

If you're following along, go ahead and add `gem 'foreman'` and `gem
'webpacker'` to the Gemfile as well. Foreman will come in handy later.

Adding Webpacker now will prevent you from facing the same ten minutes of
irritation and subsequent embarrassed epiphany that I had, wherein I tried to
run `rails webpacker:install` and kept getting yelled at by my console. Rules
don't change just because the tools do, K?

Ahem.

I also ended up having to remove `gem 'sass-rails'` and `gem 'coffee-rails'` due
to version incompatibilities that I didn't really feel the need to work too hard
to fight through. I'm not 100% convinced I want to get rid of the asset pipeline
in this project, but if I decide I really don't want to and I need to use sass
and coffee there, I'll fight that battle when it comes.

`bundle install` is the most sane command to run next, so do that.

Then run `rails webpacker:install`.

This should create an 'app/javascript/packs' directory structure, as well as
some binstubs and a handful of config files in a new directory 'config/webpack'.
Then it'll install some JavaScript dependencies.

With this simple setup, you can go ahead and see how the structure works. As
mentioned in the [Webpacker documentation](https://github.com/rails/webpacker),
you can set up a component using the following structure:

```javascript
// app/javascript/packs/calendar.js
require('calendar')
```

```
app/javascript/calendar/index.js // gets loaded by require('calendar')
app/javascript/calendar/components/grid.jsx
app/javascript/calendar/styles/grid.sass
app/javascript/calendar/models/month.js
```

```html
// app/views/layout/application.html.erb
<%= javascript_pack_tag 'calendar' %>
<%= stylehseet_pack_tag 'calendar'' %>
```

I found this structure easy enough to replicate. Here's what I have:

```javascript
// app/javascript/packs/shelves.js
require ('shelves')
```

```javascript
// app/javascript/shelves/index.js
import './stylesheets/shelves.sass';
import shelves from './shelves';

document.addEventListener('DOMContentLoaded', () => {
  shelves.initialize();
):'
```

```javascript
// app/javascript/shelves/shelves.js
const shelves = {
  initialize() {
    console.log("code to run on initialization")
  }
}

export default shelves;
```

```sass
// app/javascript/shelves/stylesheets/shelves.sass
h1
  color: red
```

Then I have a `ShelvesController` with an `index` page.

```html
-# app/views/shelves/index.html.erb
<h1>Hello, World!</h1>

<%= javascript_pack_tag 'shelves' %>
<%= stylesheet_pack_tag 'shelves' %>
```

With Foreman installed, my Procfile-dev file looks like this:

```
web: bundle exec rails s
webpack: ./bin/webpack-dev-server
```

Running `bundle exec foreman start -f Procfile-dev` then starts up my Rails
server (at port 5000, instead of the usual 3000) and a listener that will catch
any updates made to files in app/javascript.

Navigate to localhost:5000, open up the web inspector, and you should see:

1. 'code to run on initialization' in the console
2. A red header that says 'Hello, World!' on the page itself

Go ahead and change `color: red` to `color: blue` in
app/javascript/shelves/stylesheets/shelves.sass, and after a couple of moments
that page should live reload and turn your header blue!

Yay!

From here, of course, there's only room for more exploration.

Happy coding!
