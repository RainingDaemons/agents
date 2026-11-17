# Coding Style

> Source: <https://sqlite.org/zipvfs/doc/trunk/www/style.wiki> (retrieved 2026-11-17)
> The original checklist numbering (10s, 20s, 30s, …) is preserved so the
> items can be cited by number.

## General points

- **10.** No line of code exceeds 80 characters in length.
- **11.** There are no tab characters.
- **12.** 2-space indentation is used.
- **13.** Comments contain no spelling or grammatical errors. (Abbreviations
  and sentence fragments are acceptable when trying to fit a comment on a
  single line as long as the meaning is clear.)
- **14.** The tone of comments is professional and courteous. Comments contain
  no profanity, obscenity, or inuendo.
- **15.** All C-code conforms to ANSI C-89.

## C preprocessor macros

- **20.** The purpose of every preprocessor macros is clearly explained in a
  comment associated with its definition.
- **21.** Every preprocessor macro is used at least once.
- **22.** The names of preprocessor macros clearly reflect their use.
- **23.** Assumptions about the relative values of related macros are verified
  by asserts. Example: `assert(READ_LOCK+1==WRITE_LOCK);`

## Function header comments

- **30.** Every function has a header comment describing the purpose and use of
  the function.
- **31.** Function header comment defines the behavior of the function in
  sufficient detail to allow the function to be reimplemented from scratch
  without reference to the original code.
- **32.** Functions that perform dynamic memory allocation (either directly or
  indirectly via subfunctions) say so in their header comments. The memory pool
  used for dynamic memory allocation is clearly stated.
- **33.** The behavior of functions in an OOM situation is clearly stated in the
  header comment.
- **34.** The behavior of functions during an I/O error is clearly stated in the
  header comment.
- **35.** The behavior of functions when faced with a corrupt database file is
  clearly stated in the header comment.
- **36.** The behavior of the function during all other exception conditions
  (conditions other than OOM, I/O error, or corruption) is clearly stated.

## Function bodies

- **40.** The name of a function clearly reflects its purpose.
- **41.** Exception conditions do not cause memory or other resource leaks.
- **42.** Exception conditions leave the system in a consistent and well-defined
  state.
- **43.** The maximum size of a memory allocation is proportional to the the
  uncompressed database page size.
- **44.** Automatic variables are small, not large objects or arrays. Avoid
  excessive stack usage.
- **45.** The check-list items for functions also apply to major subsections
  within a function.
- **46.** Bounded recursion.
- **47.** All code subblocks are enclosed in `{...}`.
- **48.** `assert()` macros are used as follows:
  - **a.** Function preconditions are clearly stated and verified by asserts.
  - **b.** Assumptions about which thread locks are held are explicitly stated
    and tested by asserts.
  - **c.** Invariants are identified by asserts.
- **49.** `testcase()` macros are used as follows:
  - **a.** Sufficient `testcase()` macros and/or code paths exist to verify that
    all OOM scenarios have been tested.
  - **b.** Sufficient `testcase()` macros and/or code paths exist to verify that
    all I/O error scenarios have been tested.
  - **c.** Sufficient `testcase()` macros and/or code paths exist to verify that
    all corrupt database file scenarios have been tested.
  - **d.** Sufficient `testcase()` macros and/or code paths exist to verify that
    all exception conditions have been tested.
  - **e.** Sufficient `testcase()` macros and/or code paths exist to verify that
    all boundary conditions have been tested.
  - **f.** `testcase()` macros exist to prove that all cases of a switch
    statement are tested.
  - **g.** `testcase()` macros exist to prove that all cases of bit-wise
    conditional are tested.

## Class (struct) declarations

- **50.** The purpose and use of every class (a.k.a. structure) is clearly
  defined in the header comment of its declaration.
- **51.** The purpose and use of every class member is clearly defined either in
  the header comment of the class declaration or when the member is declared or
  both.
- **52.** The names of class members clearly reflect their use.
- **53.** Members use the smallest practical datatype.
- **54.** Members are arranged within a class to make the class as small as
  practical.
- **55.** Every member of internal classes is constructively used. (In other
  words, we have removed class members that have fallen out of use.)
- **56.** Invariants for classes are clearly defined.
- **57.** Functions exist within `#ifndef NDEBUG` which test class invariants and
  these functions are called from within asserts at appropriate places.

## Variables and class instances

- **60.** The purpose and use of every variable is defined by a comment at the
  variable definition.
- **61.** The names of variables clearly reflect their use.
- **62.** Related variables have related names. (ex: `aSavepoint` and
  `nSavepoint`.)
- **63.** Variables have minimum practical scope.
- **64.** Automatic variables are small, not large objects or arrays.
- **65.** Constants are "const".
- **66.** Invariants on variables or groups of variables are defined and tested
  by asserts.
- **67.** Sufficient `testcase()` macros and/or code paths exist to verify that
  every variable is exercised through the full range of its possible values.
- **68.** When a variable that refers to the same value is used within multiple
  scopes, the same name is used in all cases.
- **69.** When variables refer to different values, different names are used even
  when the names are in different scopes.
- **70.** Variable names with wide scope are sufficiently distinctive to allow
  searching for them using grep.
