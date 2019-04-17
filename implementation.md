# Implementation description

Note to the reader: this describes the sample prototype implementation.
The design choices here are not meant to represent final design decisions, but
act as a proof of concept implementation in Blink.

To implement the co-operative update phases, we need a
way to prevent the update phases in the locked subtree from blocking other
phases from occurring for other parts of the tree. The co-operative updates part
of the API can be implemented in several ways. Here, we describe one possible
implementation, which is the basis for our prototype implementation. We call
this approach budgeted update phases.

##### Budgeted update phases

In the budgeted update phases approach, we introduce a new concept to the
user-agent, called an update budget. This budget dictates how much of a locked
subtree should be processed. As an example, consider the following the following
pseudo code for laying out an element:

```cpp
void LayoutElement::UpdateLayout() {
  UpdateOwnLayout();
  ClearDirtyBitForSelfLayout();

  for (auto* child : GetChildren()) {
    if (child->NeedsLayoutForSelfOrChildren())
      child->UpdateLayout();
  }
  ClearDirtyBitForChildrenLayout();
}
```

Here the code goes through the layout tree walk. At each step, the element
updates its own layout, then iterates over all of its children. If a child, or a
sub-element of the child needs a layout, then it recursively descends into the
call. Also note that every time an element completes its layout, it clears the
dirty bit for itself and after processing its children, it also clears the dirty
bit that indicates its children need processing.

With budgeted update phases, the recursive call gets a context which tracks the
current budget. Furthermore, it checks the current budget to see if it expired.
If so, it aborts doing any future work. When it comes to dirty bits, if the
element processed its own layout, it clears its dirty bit. Also, if all of the
children and, by extension, their subtrees successfully complete layout, it also
clears the children dirty bit.

```cpp
void LayoutElement::UpdateLayout(DisplayLockContext& context) {
  auto scoped_visitor = context.VisitObject(this);
  if (context.BudgetExpired()) {
    context.DidSkipObject();
    return;
  }
  UpdateOwnLayout();
  ClearDirtyBitForSelfLayout();
  context.DidProcessObject();

  for (auto* child : GetChildren()) {
    if (child->NeedsLayoutForSelfOrChildren())
      child->UpdateLayout(context);
  }
  if (!context.HadSkippedDescendants())
    ClearDirtyBitForChildrenLayout();
}
```

Here, the budget is generated if context is visited with a locked element. That
is, if `this` specified in `VisitObject` is locked, then the context generates a
budget which will be used for all subtree calculations. In cases that the
context is not working in a locked subtree, `BudgetExpired()` always returns
false, meaning that work can proceed.

By adding similar code to all update phases, we achieve co-operative processing
of locked subtrees. Note that if a budget expired on during one of the phases,
such as layout, then we won’t process the subtree at all for the future phases,
such as paint. The reason for this is that the budget that we generate is stored
on a display lock which is associated with an element. Hence, if we have an
expired budget at an earlier phase, then it is guaranteed to be expired at a
later phase. As the code processes the DOM tree, it essentially aborts doing
work on a locked subtree if an earlier phase has not finished its work on that
subtree. All other parts of the subtree, like the rAF spinner in the explainer's
example, are still processed as normal.

For completeness, the budget can consist of several things:

* Time budget: this tracks the time that has elapsed since work started and
  expires after a predefined period.
* Element count budget: this tracks the number of elements that have been
  processed, and expires after a fixed number was completed.
* A combination of the two: this tracks the number of elements that have been
  processed, and after a fixed number is complete, it checks if the time has
  expired. If it did, the budget expires. Otherwise, it continues to process the
  next batch of elements.

##### Other approaches

Budgeted update phases is only one of the ways to achieve co-operative work.
Other approaches are possible and may yield different behavior depending on
implementation:

* Threaded update phase: locked subtrees can be processed on a separate thread.
* Fixed phase updates: locked subtrees could be processed one update phase at a
  time. For example, layout finishes first and skips the other phases. On the
  next frame, paint finishes, and skips the remainder. Etc.

It’s possible that other approaches could have better behavior than budgeted
update phases, but any approach is feasible as long as it implements
non-janking phase updates.
