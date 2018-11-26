## Overview

For some long-running, low-interaction processes---*e.g.* music players---we
mostly want them to run in the background and only occassionally peek at what's
going on.

This addon implements a little panel that can peek at named contents.

## Usage

Source `conf` from your main tmux configuration.

To create new named content, use `key_new`. This will open a new window where
you can start whatever process you wish. Then simply kill said window (or use
`key_unpeek`) and the process will continue running in the background.

To peek at the contents, simply use `key_peek`, and small "drawer panel" will
appear at the top of the current window. This "drawer" can then be closed with
`key_unpeek`.
