### Motivation

Dealing with nested tmux sessions, under a vanilla setup, is quite painful.
Typically, one either allocates dedicated prefix keys for each session, or
leverages `send-keys` to forward *n* presses of the prefix key to the *n*-th
nested session.

With deep nesting, such measures quickly become unwieldy. This is one solution
to that problem.

### Usage

This plugin provides a workflow whereby you can "focus" on a particular nested
session and operate within its context just like any other tmux session. There
are two special key bindings to move focus up and down the nesting hierarchy.

To use this addon, the current directory should be placed in your home
directory under `.tmux.d/addons/nesting` and source the `conf` file from your
tmux configuration.

The key bindings for navigating the nesting hierarchy can be configured in that
same `conf` file by setting the following variables:

* `key_focus_up`: moves focus to the next more deeply nested session, and
* `key_focus_down`: resets focus to the outermost session.

Whenever a session loses or gains focus, a hook is called so you can setup
indicators in the status bar, play music, donate money to a good cause, or
whatever works for your setup. The hooks exist in the `hooks` directory and are
as follows:

* `on-focus`: sourced whenever a session gains focus, and
* `on-blur`: sourced whenever a session loses focus.

By default, these conain a minimal status bar styling that should be reasonable
for vanilla setups.

### Implementiation Overview

Essentially, this models each session as a simple state machine. States are
modeled by `source-file`ing the appropriate file in the `states` directory, and
sessions communicate by `send-keys`ing a dedicated key to trigger appropriate
state changes.

There are three primary states a session can be in, named by punning on the
word "focus":

* `focused`: session has the focus and responds as expected,
* `myopic`: session is "too far away", *i.e.* more deeply nested than the
    currently focused session, and
* `hyperopic`: session is "too close", *i.e.* less deeply nested than the
    currently focused session.

There are also `init-*` states intended for one-time sourcing a session
startup. The `blurred` state is a transient one that `myopic` and `hyperopic`
both transition through---essentially, it just factors out the common code
between those two states.

The primary mechanism works by abusing `bind-key` and `send-keys` in
conjunction. The `send-keys` command acts as a message passing mechanism
between sessions and `bind-keys` allows sessions to transision between states
when receiving a "message". The "message" keys are

* `msg_should_focus`: indicating that focus is moving to a more deeply nested
    session, and
* `msg_should_blur`: indicating that focus is moving to a less deeply nested
    session.

This introduces a couple quirks. We can actually push focus off the end of the
nesting hierarchy. This can be easily reset by pressing `key_focus_down`, but
while every session is "blurred", the `msg_should_focus` key gets leaked to the
application.

Second, since `send-keys` can only send messages up the nesting hierarchy, we
have no stateless way of decrementing focus down the nesting hierarchy. This is
why `key_focus_down` resets focus instead of doing the more intuitive thing.
