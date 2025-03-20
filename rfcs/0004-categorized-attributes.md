- Feature Name: Categorized attributes
- Affected audience: Game Developers
- RFC PR: [#9114](https://github.com/endless-sky/endless-sky/pull/9114)
- Relevant Issues/RFCs: [#9025](https://github.com/endless-sky/endless-sky/pull/9025)

# Summary

Provides a way of programmatically accessing attributes instead of requiring the unique, string-form name of the attribute.

It is a **goal** to provide programmatic access to attributes relating to runtime ship properties, such as shield amount, damage dealt by a projectile, or energy consumption.
It is a **goal** to maintain compatibility with the old data format, including support for every existing attribute with engine support.

It is a **non-goal** to provide such access to every attribute.
It is a **non-goal** to limit the supported set of attributes.
It is a **non-goal** to coerce the format of the string attributes to allow for generating the attribute names programmatically.
It is a **non-goal** to maintain parity with the old data format; new attributes might receive engine support without a corresponding old attribute name.

# Motivation

The game is rapidly expanding the number of attributes that require in-engine support. One great example can be seen in the damage types: we have shield damage, hull damage, ion damage, fuel damage, slowing damage, heat damage, etc. These damage types all have their own damage, protection, resistance etc. attributes for parity. Currently, all of these require a unique, non machine-generated string name that is used as a literal across multiple engine functions. This usage is costly to maintain due to the number of engine changes required for a new attribute, and also leaves great potential for errors due to the use of arbitrary string literals.

This RFC proposes a unified system replacing the string literals used for attribute access inside the engine, offering better maintainability and, in the future, better performance as well.

# Detailed Design

A _categorized attribute_ is an attribute that is affected by the proposed changes, as outlined in the goals section.

This RFC identifies the various components of a _categorized attribute_ as:
- Category (when is the change made?)
- Effect (what is changed?)
- Modifier (how is this changed?)

_Categorized attributes_ are referred to as _old-form_ when denoted by a string, and _new-form_ when denoted by a _category_+_effect_+_modifier_ combination.

Every _categorized attribute_ MUST be represented by exactly one _category_+_effect_+_modifier_ combination, though some combinations SHOULD NOT be supported.

There MUST be a _default modifier_ indicating the base value of a non over-time effect.

## Data format

The data format MUST define a keyword for every _category_. It MUST either define a keyword for every _effect_ and _modifier_ individually, or a keyword for every _effect_ and _modifier_ combination. It MUST NOT define a keyword for the _default modifier_.

These keywords MUST be supported for parsing, and MUST NOT be context-sensitive. These keywords MUST be supported by a parser in a nested approach.

### Parsers

A parser MAY follow one of the example formats:

```
ship <name>
	attributes
		<category-keyword> [<value#>]
			<effect-keyword> [<value#>]
				<modifier-keyword> [<value#>]
```

```
ship <name>
	attributes
		<category-keyword> [<value#>]
			<effect-modifier-keyword> [<value#>]
```

A parser MUST support parsing from a ship, outfit or weapon definition. It MUST support parsing attributes from `attribute` nodes containing mixed _new-form_ and _old-form_ _categorized attributes_. A parser MUST support all _old-form_ and _new-form_ attribute definitions. A parser MAY provide feedback about unsupported attributes.

A parser SHALL provide feedback about unsupported value tokens. It MUST support a value token for the _effect_ node, implying the _default modifier_. It MAY support values for _category_ keywords directly, implying a default _effect_. (This _effect_ SHOULD be different for every _category_.) A parser SHALL support applicable _category_, _effect_, and _modifier_ nodes without a value, assuming a default value of 1. This behaviour MUST NOT be conditional on the presence of child nodes.

A parser MUST abstract the presence of _old-form_ _categorized attributes_ from the engine.

### Generators

A generator for the save file format MUST support _new-form_ attribute generation. It SHALL NOT generate _old-form_ syntax for _categorized attributes_. It MUST NOT generate duplicate entries of the _category_, _effect_, _modifier_, or combined _modifier_ + _effect_ keywords.

A generator SHALL use a data node to represent a _category_ and _effect_ or _effect_ + _category_ combination. A generator SHOULD NOT include attributes with their default value.

A generator SHOULD use a deterministic output ordering for attributes, and MAY order them alphabetically. It SHALL provide the most concise output possible.

A generator MAY NOT support _old-form_ _categorized attributes_.

### Engine support

The supported _categories_ SHOULD include all of the following:

```
shield generation
hull repair
thrusting
reverse thrusting
steering
active cooling
ramscooping
cloaking
afterburning
firing
hyperjump
jump
```

It MAY include any or all of the following:
```
protection
resistance
damage
capacity
```

The supported _effects_ SHOULD include all of the following:
```
shields
hull
thrust
reverse thrust
turn
active cooling
ramscoop
cloak
cooling
force
energy
fuel
heat
jam
disabled damage
minable damage
piercing
```

The supported _modifiers_ SHOULD include all of the following:
```
none
multiplier
relative
over time
```

The _default modifier_ MUST be `none`, for which a keyword is SHOULD NOT be specified.

### Implementation

The constraints of attribute values MUST be enforced in runtime. _Categorized attributes_ with their default value MAY NOT be stored. There MUST exist an abstraction layer supporting queries and updates of stored and non-stored _categorized attributes_. This abstraction layer SHOULD support non-_categorized attributes_ as well. This layer MUST support storing any _category_+_effect_+_modifier_ combination as a _categorized attribute_, regardless of engine support for such an attribute.

Querying the value of a non-stored attribute MUST return its default value, and MUST NOT cause exceptions or undefined behaviour. 

An implementation MUST allow efficient query of all _categorized attributes_ when given a _category_. It MUST allow for efficient transformation of all _categorized attributes_ of a given category, such as adding another set of matching attributes to it on a per-value basis.

For instance, such an operation MUST be performed efficiently:

```
capacity
	shield 100
	hull 20

-

damage
	shield 10
	hull 10

=

capacity
	shield 90
	hull 10
```

An implementation SHOULD store _categorized attributes_ grouped by _category_. It SHOULD NOT store the _category_ of the individual attribute values.

An implementation MAY use a similar structure for storing attributes:

```
class Entry {
	Effect e;
	Modifier m;
}

std::map<Entry, double> attributes[CATEGORY_COUNT];
```
```
class Entry {
	Effect e;
	Modifier m;

	double value;
}

std::map<Category, std::set<Entry>> attributes;
```

An implementation MUST provide const iterator functionality over attributes. It MAY provide non-const iterator functionality.

An implementation MUST provide efficient functionality matching that of Outfit::CanAdd for _categorized attributes_.

# Drawbacks

## Maintenance cost:

We have to keep a list of all _old-form_ attribute names that the engine refers to using the new format. Though this list need not be updated or maintained, it represents an additional cost.

This change will also heavily conflict with and other pull requests that create new _categorized attributes_, or change engine code using such attributes. They will all have to be moved to the _new-form_ syntax, creating a burden on the PR authors.

## Performance

Though the performance of the final form of this RFC should be better than our current performance, it would perform significantly worse while only the data format is implemented. Early tests suggest a doubling of the individual lookup cost of categorized attributes using the currently implemented method in the worst case.

# Alternatives

We could programmatically generate the _old-form_ attribute names. This would preserve a larger portion of the existing code base and would require no changes from plugin authors, however [past efforts show it to be prohibitively complex](https://github.com/tibetiroka/endless-sky/blob/99108ba344a9de01bcab5eba7f949e39b9bb3758/source/Attribute.cpp#L85-L165).

# Unresolved Questions

Is there a better way of managing compatibility than storing the _old-form_ to _new-form_ mappings?

Is it worth using a Dictionary-style flap map over std::map for storing attributes? Is it really that much faster even for non-_categorized attributes_?

Should the constraints of attribute values be enforced by the individual attribute entries, or the data store? Is the performance cost of enforcing them on every operation worth it? In what cases should we validate the input?
