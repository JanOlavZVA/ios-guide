
**setNeedsLayout** will layout subviews
Call this method on your application’s main thread when you want to adjust the layout of a view’s subviews.

**setNeedsDisplay** will call for a redraw of your view (drawRect:, etc).
You can use this method or the setNeedsDisplayInRect: to notify the system that your view’s contents need to be redrawn.



# Implicit and explicit constraints

Explicit constraints are the constraints you create explicitly. You can add constraints in code or in Interface Builder. 
Implicit constraints are constraints Auto Layout creates for you.

Auto Layout translates the intrinsic content size of a view to a set of implicit constraints, one constraint per dimension.
It is important to know that not every view has an intrinsic content size. A vanilla UIView instance, for example, does not have an intrinsic content size. This means that you need to add four explicit constraints to describe the size and position of a UIView instance in a user interface.

For example: The intrinsic content size of a label is equal to the size of the label's text.

**The intrinsic content size is dynamic and calculated at runtime.**