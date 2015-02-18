---
title: Cursing Windows
date: 2015-02-18 14:40 UTC
tags: ruby, ncurses, curses
---
## Cursing Windows
### AKA 'How On earth do I make a window?!'

This post uses [ncurses-ruby](https://github.com/eclubb/ncurses-ruby) I'd
recommend looking through some of the examples with the caveat that they
are not quite idiomatic Ruby. They are helpful enough to get started.

```ruby
require 'ncurses'

Ncurses::initscr()
Ncurses::start_color()
Ncurses::noecho()
Ncurses::cbreak()
Ncurses::keypad(Ncurses.stdscr, true)

begin
  win = Ncurses::WINDOW.new(10, 10, 10, 10)
  win.printw("lol")
  win.getch()
rescue => e
  raise e
ensure
  Ncurses::endwin()
end
```

First off we're starting ncurses which has a top level window that other things
will be drawn into.

Then we create a sub window with `Ncurses::WINDOW.new(height, width, y, x)` and use
`win.printw("some_string")` to print into it, and we wait for user input
by using `getch`.

**Note:** x and y refer to the top left corner of the window, also they're
**backwards** from what you would expect in a normal coordinate. This is
repeated in nCurses and has bitten me more than once ;-)

I wrapped up the main window drawing stuff in a begin/ensure
because if something exits with an error without ending/cleaning up the main window
noecho/cbreak it will bork your terminal! :D. Let's push this setup into a utility/wrapper
so we don't have to think about it.

If you want to read more about this initial setup/options I mostly just fiddled
with what I found at [The Linux Documentation Project](http://tldp.org/HOWTO/NCURSES-Programming-HOWTO/init.html)
init section.

```ruby
require 'ncurses'
class SwearWords

  def start_cursing(&block)
    set_options
    begin
      block.call
    rescue => exception
      raise exception
    ensure
      Ncurses::endwin()
    end
  end

  private

  def set_options
    Ncurses::initscr()
    Ncurses::start_color()
    Ncurses::noecho()
    Ncurses::cbreak()
    Ncurses::keypad(Ncurses.stdscr, true)
  end
end
```

Later we can maybe come back and setup some way of handing in
setup options but for now this will do.

```ruby
SwearWords.new.start_cursing do
  win = Ncurses::WINDOW.new(10, 10, 10, 10)
  win.printw("lol")
  win.getch
end
```

We have a window! Super useful right? Let's see if we can move it around.

```ruby
SwearWords.new.start_cursing do
  win = Ncurses::WINDOW.new(10, 10, 10, 10)
  win.printw("lol")
  win.getch
  win.mvwin(15,15)
  win.getch
end
```

This redraws the window in a new location but the old one is still floating
around :'(. Turns out what we need here is a call to `Ncurses.refresh`

```ruby
SwearWords.new.start_cursing do
  win = Ncurses::WINDOW.new(10, 10, 10, 10)
  win.printw("lol")
  win.getch
  win.mvwin(15,15)
  Ncurses.refresh
  win.getch
end
```

Currently the text is being written to the `(0,0)` location within
the window. To change this we can move the cursor with `win.move(y, x)` or
we could do it in one step with `win.mvaddstr(y, x, "some_string")`.

```ruby
SwearWords.new.start_cursing do
  win = Ncurses::WINDOW.new(10, 10, 10, 10)
  win.mvaddstr(5,2, "lol")
  win.getch
  win.mvwin(15,15)
  Ncurses.refresh
  win.getch
end
```

The last thing I wanted to try was adding borders around the window. This
is easily done by passing 8 `0`s as an argument to `win.border()`

```ruby
  idk = Array.new(8) { 0 }
  win.border(*idk)
```

It's not ideal, and I have no idea why it works, but this is enough to get started
playing with windows. I'll leave the fiddling with the borders as an exercise for
a later post. :-)
