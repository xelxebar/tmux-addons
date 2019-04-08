## Motivation

Dealing with nested tmux sessions, under a vanilla setup, is quite painful.
Typically, one either allocates dedicated prefix keys for each session, or
leverages `send-keys` to forward *n* presses of the prefix key to the *n*-th
nested session.

With deep nesting, such measures quickly become unwieldy. This is one solution
to that problem.

## Usage

This plugin provides a workflow whereby you can "focus" on a particular nested
session and operate within its context just like any other tmux session. There
are two special key bindings to move focus up and down the nesting hierarchy.

To use this addon, the current directory should be placed in your home
directory under `.tmux.d/addons/nesting` and source the `conf` file from your
tmux configuration. Naturally, this should be done on all machines intended to
participate in the nesting interaction.

The key bindings for navigating the nesting hierarchy can be configured in that
same `conf` file by setting the following variables:

* `key_focus_up`: moves focus to the next more deeply nested session, and
* `key_focus_reset`: resets focus to the outermost session.

Also, the `key_prefix` variable needs to be set in the files
`states/{focused,blurred}`. Unfortunately, due to a limitation of tmux, we
can't propogate variables between sourced files, so for now we have to manually
set the prefix key in every file that needs it.

Whenever a session loses or gains focus, a hook is called so you can setup
indicators in the status bar, play music, donate money to a good cause, or
whatever works for your setup. The hooks exist in the `hooks` directory and are
as follows:

* `on-focus`: sourced whenever a session gains focus,
* `on-blur`: sourced whenever a session loses focus,
* `init-focused`: to be sourced once when initializing a focused session, and
* `init-blurred`: to be sourced once when initializing a blurred session.

By default, these conain a minimal status bar styling that should be reasonable
for vanilla setups.

## Usage with ssh

When logging into machines that automatically start tmux, say from their
`.profile`, it's nice to have those sessions start in the blurred state. To
that end we can use an environment variable and load the appropriate `init-*`
hook when nested. Here's a minimal example

```
# .profile

if [ -z "${TMUX}" ]; then
	tmux_conf=~/.tmux.d/addons/nesting/hooks/${TMUX_STATE_HINT:-init-focused}
	export TMUX_STATE_HINT=init-blurred

	exec tmux -f "${tmux_conf}"
fi
```

We then need to tell ssh to send `TMUX_STATE_HINT` to the remote machine. This
should be done on every machine---actually, it's optional on any machine that
doesn't expect to parent further nesting:

```
# /etc/ssh/ssh_config
SendEnv TMUX_STATE_HINT
```

And then every nested machine needs to configure the ssh daemon to accept
`TMUX_STATE_HINT`:

```
# /etc/ssh/sshd_config
AcceptEnv TMUX_STATE_HINT
```

Of couse, this requires administrator priviledges to edit those config files.
In case that's not possible, you can just put all nested session in the blurred
state with `key_focus_reset` after login.

## Implementiation Overview

Essentially, this models each session as a simple state machine. States are
modeled by `source-file`ing the appropriate file in the `states` directory, and
sessions communicate by `send-keys`ing a dedicated key to trigger appropriate
state changes.

There are three primary states a session can be in, named by punning on the
word "focus":

* `focused`: session has focus and responds to prefix key,
* `underfocussed`: session is "too far away", *i.e.* more deeply nested than
    the currently focused session, and
* `overfocussed`: session is "too close", *i.e.* less deeply nested than the
    currently focused session.

The `blurred` state is a transient one that `underfocussed` and `overfocussed`
both transition through---essentially, it just factors out the common code
between them.

The primary mechanism works by abusing `bind-key` and `send-keys` in
conjunction. The `send-keys` command acts as a message passing mechanism
between sessions and `bind-keys` allows sessions to transision between states
when receiving a "message". The "message" keys are

* `msg_should_focus`: indicating that focus is moving to a more deeply nested
    session, and
* `msg_should_blur`: indicating that focus is moving to a less deeply nested
    session.

This introduces a couple quirks. We can actually push focus off the end of the
nesting hierarchy. This can be easily reset by pressing `key_focus_reset`, but
while every session is "blurred", the `msg_should_focus` key gets leaked to the
application.

Second, since `send-keys` can only send messages up the nesting hierarchy, we
have no stateless way of decrementing focus down the nesting hierarchy. This is
why whe have `key_focus_reset` instead of a `key_focus_down`.
