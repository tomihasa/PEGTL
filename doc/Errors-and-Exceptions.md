# Errors and Exceptions

In the most basic case, the result of a parsing run is a `bool` that indicates whether a given input adheres to a given grammar or parsing rule.

Usually it is necessary to obtain a more precise error message than `false`, one that indicates what went wrong and where.

## Contents

* [Failure](#failure)
* [Error Messages](#error-messages)

## Failure

The PEGTL offers facilities to define error points in the grammar where a return value of `false`, in PEGTL terminology a "local failure", should be converted into an exception, a "global failure".

These global failures prevent back-tracking to proceed beyond the error points in order to

1) throw the parse error at a well-defined point where the parsing run was unable to proceed with matching, and
2) enable certain assumptions to be made in other branches of the grammar.

In this section we will only look at the first item.

For example consider the following parsing rule for a backslash-escape-sequence, simplified from what is allowed in C++ string literals.

```c++
   using namespace tao::pegtl;
   struct esc : seq< one< '\\' >, one< 'n', 'r', 't' > > {};
```

#### Must

Usually in a string literal, the backslash *must* be followed by one of a certain set of characters.

If the following input character is not valid, we do not want to allow back-tracking via a local failure.
We rather want to generate a global failure *at that point* by throwing an exception.

The PEGTL contains the `must< R... >` combinator which defines such an error point.
It converts local failure into global failure, thereby expressing that, when this point was reached, the input *must* match the given rules, or else the parsing run immediately fails with an exception and without back-tracking.

The task of actually throwing the exception is delegated to the [control class'](Control-Hooks.md) `raise()`-method.
The exception thrown by the default control class `tao::pegtl::normal` contains the demangled name of the failed parsing rule, and the position in the input at which the rule was attempted to match.

Given the `must<>`-combinator we can rewrite the example above with the correct semantics that globally fails a parsing run when the backslash is not followed by a valid character.

```c++
   using namespace tao::pegtl;
   struct esc : seq< one< '\\' >, must< one< 'n', 'r', 't' > > > {};
```

Other [convenience rules](Rule-Reference.md#convenience) that define an error point in the grammar with an implicit `must<>` all contain `must` in their name.
One of these is `if_must<>`, which allows us to shorten our example equivalently.

```c++
   using namespace tao::pegtl;
   struct esc : if_must< one< '\\' >, one< 'n', 'r', 't' > > {};
```

Since the demangled name of the rule that failed to match is used in the error message,
the example can be improved further by choosing a more descriptive name for the error point,
the rule that could trigger a global failure.

```c++
   using namespace tao::pegtl;
   struct escaped_char : one< 'n', 'r', 't' > {};
   struct esc : if_must< one< '\\' >, escaped_char > {};
```

#### Raise

It is also possible to generate a global failure via the `raise< T >` parsing rule.

A `raise< T >`-rule unconditionally calls `Control< T >::raise()` on the current control class template instantiated for `T`.

In other words, `must< Rule >` is equivalent to `sor< Rule, raise< Rule > >`, however `raise<>` is slightly more flexible:

The template argument to `raise<>` can be *any* type `T`, not just a parsing rule in the grammar, as long as the control class template can be instantiated for `T`.

## Error Messages

Once the error points are defined in a grammar the next task is to customise the error messages to make them more readable.

By default, when using any `must<>` error points, the exceptions generated by the PEGTL use the demangled name of the failed parsing rule as descriptive part of the error message.

The most powerful, and cumbersome, way to customise error messages is to specialise the control class for each rule that occurs as error point and implement a different `raise()`-method in each case.

When the exception class is always the same it is possible to simplify the implementation of custom error messages to not require control class specialisations.

This is done with a custom control class template whose `raise()`-method uses a static string as error message.

```c++
template< typename Rule >
struct my_control
   : tao::pegtl::normal< Rule >
{
   static const std::string error_message;

   template< typename Input, typename... States >
   static void raise( const Input& in, States&&... )
   {
      throw tao::pegtl::parse_error( error_message, in );
   }
};
```

Now only the `error_message` string needs to be specialised, rather than the whole class, as follows.

```c++
template<>
const std::string my_control< MyRule >::error_message =
   "expected ...";
```

Since the `raise()`-method is only instantiated for those rules for which `must<>` could trigger an exception, it is sufficient to provide specialisations of the error message string for those rules.
Furthermore, there will be a linker error for all rules for which the specialisation was forgotten although `raise()` could be called.
For an example of this method see `examples/json_errors.hpp`.

It is also possible to provide a default error message that will be chosen by the compiler in the absence of a specialised one as follows.

```c++
template< typename T >
const std::string my_control< T >::error_message =
   "parse error matching " + tao::pegtl::internal::demangle< T >();
```

It is advisable to choose the error points in the grammar with prudence.
This choice becoming particularly cumbersome and/or resulting in a large number of error points might be an indication of the grammar needing some kind of simplification or restructuring.

Copyright (c) 2014-2017 Dr. Colin Hirsch and Daniel Frey
