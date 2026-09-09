Three keywords, describing use of fields:
- `from`
- `to`
- `by`

These 3 describe whole semantics of references/owned, lifetimes, const/mutable/volatile

# from
- Local/owned fields: `from this`
- References: `from <other field>`
- Owned but not local (managed): `from rel this`
- Covers everything needed for lifetimes
- A list of field names, which might own this
- Called: region? source? home?
- Used keywords before: `own`, `ref`

# to
- "exposed to"
- Who can modify the referred-to field
- whether local or reference, whether directly modified or volatile by the system
- Variable: `to this`
- Volatile: tbd.
- A list of functions, or records containing functions
- Called: exposition? ~~mutator?~~
- Used keywords before: `mut`, `vol`, `ex`

# by
- Who can modify the reference field?
- Is the field variable or a const?
- Local variables, local fields that refer to some const value
- const: `by <nothing>`
- normal variable: `by this`
- Allowing some other function to change this: `by <something>`
- Whom can we give a reference to this field?
- Called: ~~mutator?~~ variability?
- Used keywords before: `var`, `const`, `@`, also `vol`

- Calling a function that is allowed to modify this implies we can modify it too - check `by this`
- Calling a function that *has access to a function that* is allowed to modify this implies we can modify it too - check `by this` too