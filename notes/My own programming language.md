```
# get average score of people above 18
# { users: { name: string, age: number, score: number }[] }

=(users pick(users <<))
=(count size(users))
=(scores |(
	users
	filter(<=(18))
	map(pick(score))
	reduce(+)
))

>>(/(scores count));
```

```
/(|(
	$
	filter(<=(18))
	map(pick(score))
	reduce(+)
) size($))
```

```
[
	filter(|v| <=(18 v.age))
	map(pick(score))
	sum
	|v| /(v ?)
]
```