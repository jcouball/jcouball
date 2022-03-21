# Defining Class Variables and Constants in an Anonymous Class

## Class Variables

If you were creating a class using the `class` keyword, you could simply
do the following to create a class variable for the class:

```Ruby
class MyClass
  @@class_var = :foo
end
```

However, when creating an anonymous class using `Class.new`, the following
will define the class variable on `Object` instead of on the new anonymous
class (this actually fails on Ruby 3.0.0 or later with a RuntimeError "class
variable access from toplevel":

```Ruby
MyClass = Class.new do
  @@class_var = :foo
end
```

This is because class variable references are lexically scoped. In the case
of anonymous classes, `Class.new` evaluates the given block with `class_eval`
which DOES NOT change the lexical scope for the block. This means that the lexical
scope inside the block is the same as the lexical scope outside the block.

To get the class variable to be defined on `MyClass` instead of `Object`, you
need to use the `class_variable_set` method:

```Ruby
MyClass = Class.new do
  self.class_variable_set(:@@class_var, :foo)
end
```

Using `self` in the method invokation `self.class_variable_set` is redundant.
I included it in this example to make the point that `class_eval` DOES set
`self` as expected and that the solution to defining a class variable for
the anonymous classs is to explicitly call `class_variable_set` in the anonymous
class initialization block.

Here is another example that avoids problems with "class variable access
from toplevel" RuntimeError in Ruby 3.x. Here is how class variables work
using the `class` keyword:

```Ruby
class OuterClass
  class InnerClass
    @@class_var = :foo
  end
end

# The results are as expected:
OuterClass::InnerClass.class_variables #=> [:@@class_var]
OuterClass.class_variables #=> []
```

The incorrect solution where InnerClass is an anonymous class:

```Ruby
class OuterClass
  InnerClass = Class.new do
    @@class_var = :foo
  end
end

# Oops, the class variable `@@class_var` is defined on OuterClass
# instead of InnerClass:
OuterClass::InnerClass.class_variables #=> []
OuterClass.class_variables #=> [:@@class_var]
```

And the corrected solution where InnerClass is an anonymous class:

```Ruby
class OuterClass
  InnerClass = Class.new do
    class_variable_set(:@@class_var, :foo)
  end
end

# The results are (once again) as expected:
OuterClass::InnerClass.class_variables #=> [:@@class_var]
OuterClass.class_variables #=> []
```

## Constants

The same problem exists for constants and is solved in the same way.

Here is how class variables work using the `class` keyword:

```Ruby
class OuterClass
  class InnerClass
    CONSTANT = :foo
  end
end

# The results are as expected:
OuterClass::InnerClass.constants #=> [:CONSTANT]
OuterClass.constants #=> [:InnerClass]
```

The incorrect solution where InnerClass is an anonymous class:

```Ruby
class OuterClass
  InnerClass = Class.new do
    CONSTANT = :foo
  end
end

# Oops, the constant :CONSTANT is defined on OuterClass instead of InnerClass:
OuterClass::InnerClass.constants #=> []
OuterClass.constants #=> [:CONSTANT, :InnerClass]
```

And the corrected solution where InnerClass is an anonymous class:

```Ruby
class OuterClass
  InnerClass = Class.new do
    const_set(:CONSTANT, :foo)
  end
end

# The results are (once again) as expected:
OuterClass::InnerClass.constants #=> [:CONSTANT]
OuterClass.constants #=> [:InnerClass]
```
