It's a merge between [[Functional Programming]] and [[Reactive programming]].

Typically, systems described as reactive programming emphasize distributed processing, whereas FRP is more **fine-grained** and starts with **strong consistency**.

It's a composable, modular way to code **event-driven** logic and it can also be seen as a replacement for the [[Observer pattern]].

The program is expressed as a reaction to its inputs or as a **flow of data**. It brings order to the management of the program **state**.

It's not a methodology and it won't solve all problems. It can help drastically improve and refactor event propagation.

FRP works the same way a spreadsheet does.

FRP belongs to a new [[Paradigm]]. Although you can use FRP to make incremental improvements to an existing code base, there’s a different way of thinking underlying it. If you embrace that way of thinking, you’ll get the most out of FRP.

FRP code has a struncture like a directed graph, and an FRP engine derives execution order automatically from it.

FRP code is typically half pure logic and half management of variable scoping.

Often this means there are more explicit input and outputs than you would get in normal programming. It’s not that there are more inputs and outputs; it’s just that they’re more visible. We think this is a good thing because inputs and outputs are a source of complexity. Having them right up in your face pushes you to deal with them earlier.

To future-proof our code and free us for hardware innovation, we need to do one simple thing: program in a way that fits the problem and gives dependency information to the compiler—and, in our specific case, the FRP system—so it can write the best code for whatever machine it’s targeting. Functional programming in general does this by tracking data dependencies and removin in-place state mutation. That means that FRP helps overcome the shortcomings of the moderns CPUs pretending to be [[von Neumann machine]]s.

We say FRP is compositional because it imposes mathematical rules that force the interaction of program elements to be simple and predictable, without subtleties. Their complexities add instead of multiplying, so overall complexity stays closer to proportional to program size. Put things together, and you don’t get any surprises.

FRPs [[Compositionality]] plays very well with [[Reductionism in engineering]]. Event propagation is widely used as "glue" for composing software components. FRP works because it imposes rules on composition.

FRP reduces the number of if statements meaning it reduces [[Cyclomatic complexity]].

Uses [[Listeners]] and [[Producers]].

It’s safe to call send() from any context except an FRP listener, and it never blocks.

## True FRP
*Conal Elliott is one of the inventors of FRP. We'll call his specifiaction of FRP as the True FRP.*

**If sequence and time don't matter in True Functiona Reactive programming then how do I ensure that for example close doesn't come before open which would let the thing (for example water) open which might be catastrophic**

State changes happen at predictable times.

User `streams` for actions and `cells` for data.

### Ideas
#### Denotative
Founded on a precise, simple, implementation-independent, compositional semantics that exactly specifie the meaning of each type and building block.

It uses [[Denotational Semantics]] together with [[Compositionality]] for definition of its behavior.

#### Temporally continuous

### Life cycle
#### Initialization
Typically during program startup, FRP code statements are converted into a directed graph in memory.

#### Running
For the rest of the program execution you feed in values and turn the crank handle, and the FRP engine produces output.

### Primitives
#### Streams
Stream of events.

#### Cells
Valu that changes over time.

#### Operations
- map
- merge
- hold
- snapshot
- filter
- lift
- never
- constant
- sample
- switch

## Implenetations
- [[SodiumFRP]]
- [[RxJS]]

## Future directions
- Performance
	- JIT optimization based on measurements of runtime performance
	- Precompiling
- [[Parallelism]]
- Syntax improvements
	- Auto-lifting
	- Implicit forward references
	- Infix operators
	- Type inference
- Standardization and code reuse
- FRP database applications
- Visualization and debugging tools
- Visual programming
- Refactoring tools