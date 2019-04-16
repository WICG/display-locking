# Background: User-agent's Update Phases

When processing DOM changes, the user-agent typically goes through several
stages, which we will call *update phases*. In order to understand how display
locking proposal is going to work, we will briefly discuss the major update
phases.

**Note to the reader**: these phases are covered in greater detail in the
[rendering event
loop](https://github.com/chrishtr/rendering/blob/master/rendering-event-loop.md).
Here, we briefly go over the main update phases that happen in a typical
user-agent.

## Script
During the script update phase, the user-agent executes script requested by the
page. At this time, the script on the page is free to mutate the DOM in any way.
It can do things like update element style, position elements based on a custom
animation, add and remove elements, and many other things. Synchronous
javascript functions that are invoked here will finish to completion. This means
that if a function runs for a long time, other update phases cannot proceed and
the page janks. Script, however, cannot be interrupted by the user-agent.
Because of this, developers should always be mindful of long running script.

After the script phase finishes, the DOM structure and style for the next visual
update is finalized.

## Layout and paint
During the layout and paint phase, the user-agent goes through the DOM and
updates any properties that need to be updated. Specifically, it can apply new
style to elements, position and size the elements based on style, as well as
generate draw commands needed to visually display the content to the screen.
Note that it is wasteful to do these steps on every element, so the user-agent
typically maintains a set of elements that have changed, or could have been
changed by updates to style or elements which may have happened during script
execution. This is typically done using some sort of dirty bits, or a list of
dirty elements.

For the elements that do need updates, the user-agent processes them in some
defined order. This work finishes to completion. This means that depending on
the complexity of the DOM, and the number of changes required, this process can
take a long time. This, again, causes jank since other update phases do not
happen while layout and paint is executing.

The output of the layout and paint phase is a sequence of draw commands which,
when executed, will draw the current visual update.

## Compositing
In order to avoid doing repeated work, modern user-agents also employ
compositing. Compositing is a process of splitting up draw commands into
separate layers, each with its own backing, in order to avoid re-rasterizing
content in other layers.

As an example, if a div is animating and moving across the screen using a css
animation, then it might make sense to separate the div and its draw commands
into a separate layer. This would mean that the content of the div can be
rasterized once, and the animation would simply move the backing across the
screen. This is typically significantly faster than invalidating content,
discarding the previously rasterized content, and rasterizing new content in a
new location.

Compositing is typically determined by the user-agent based on heuristics. It
isn’t perfect, but it can help by reducing the amount of time spent rasterizing
content. On top of that, the developer can provide hints to the user-agent
indicating that an element, and its subtree, are likely to move together. For
example, “will-change: transform” is one such property that the user-agent may
use to promote the affected element to a separate layer.

The output of the compositing update phase is a set of layers with either draw
commands, or rasterized textures.

## Presentation
The final update phase is presentation to the screen. By this time, the
user-agent has generated draw commands, or possibly even rasterized those into
separate layers, and arranged the layers in the final locations. With GPU
acceleration, this update phase issues GPU commands to the driver to display the
layers and its contents.

At the end of the presentation phase, the content is finally visible to the
user.
