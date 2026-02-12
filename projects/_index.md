# Pipem

```ts
const isAdult = pipe(
	match([
		[isNumber, passValue],
		[isString, toInt]
	]),
	gte(18),
	wrap(
		lte(150),
		error("Person cannot be older than 150 years")
	),
);
```

# Vaal

```ts
const person = object({
	firstName: string(length(gte(4), lte(32))),
	secondName: string(length(gte(4), lte(32))),
	age: toInt(gte(1), lte(150)),
	gender: enum(Gender),
	pet: optional(string()),
	languages: array(enum(Language))
});
```

# uEve

```ts
const $click = event(tuple(float(), float()));

const unsubClick = subscribe($click, () => ...);

publish($click, [42.0, 6.9]);

unsubClick();
```

# Reval???

```ts
const $person = person({ ... });

watch($person, () => {
  
});

$($person, { ... });
```