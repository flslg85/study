## Method Naming Conventions

Prefix | Method Type | Use
-- | -- | --
of | static factory | Creates an instance where the factory is primarily validating the input parameters, not converting them.
from | static factory | Converts the input parameters to an instance of the target class, which may involve losing information from the input.
parse | static factory | Parses the input string to produce an instance of the target class.
format | instance | Uses the specified formatter to format the values in the temporal object to produce a string.
get | instance | Returns a part of the state of the target object.
is | instance | Queries the state of the target object.
with | instance | Returns a copy of the target object with one element changed; this is the immutable equivalent to a set method on a JavaBean.
plus | instance | Returns a copy of the target object with an amount of time added.
minus | instance | Returns a copy of the target object with an amount of time subtracted.
to | instance | Converts this object to another type.
at | instance | Combines this object with another.
