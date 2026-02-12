---
link: https://people.eecs.berkeley.edu/~ksen/papers/jalangi-tool.pdf
github: https://github.com/Samsung/jalangi2
---

# TLDR;

Jalangi, a powerful and versatile framework for dynamic analysis of JavaScript applications. It provides a simple yet effective means of implementing heavy-weight dynamic analyses for JavaScript, enabling researchers and developers to conduct various types of analyses, such as concolic testing, symbolic execution, and taint analysis, across different platforms and environments.

---

Jalangi incorporates two key techniques: 

1) selective record-replay, a technique which enables to record and to faithfully replay a user-selected part of the program
2) shadow values and shadow execution, which enables easy implementation of heavy-weight dynamic analyses such as concolic testing and taint tracking

In Jalangi, we have implemented several dynamic analyses using shadow values and execution: 

1) concolic testing
2) pure symbolic execution
3) tracking origins of null and undefined
4) detecting likely type inconsistencies
5) a simple object allocation profiler
6) a simple dynamic taint analysis

## Analysis types

- Concolic testing: Concolic testing performs symbolic execution along a concrete execution path, generates a logical formula denoting a constraint on the input values, and solves a constraint to gener- ate new test inputs that would execute the program along previously unexplored paths. Concolic testing in Jalangi supports constraints over integer, string, and object types and novel type constraints. We intro- duced type constraints to handle the dynamic nature of JavaScript—the type of an input variable could be different for different feasible execution paths of the program.
- Pure symbolic execution: Pure symbolic execution ex- ecutes the program symbolically and never restarts the program for the purpose of backtracking. It check- points the state before executing a branch statement, executes one branch, and later backtracks with the checkpointed state to explore the other branch. For small programs, pure symbolic execution avoids time wastage due to repeated restarts.
- Tracking origins of null and undefined: This anal- ysis records source code locations where null and un- defined values originate and reports the location when an error occurs dues a null or undefined. Whenever there is an error due to such literals, such as accessing the field of a null value, the shadow value of the literal is reported to the user. Such a report helps the pro- grammer to easily identify the origin of the null value.
- Detecting Likely Type Inconsistencies: The dynamic analysis checks if an object created at a given pro- gram location can assume multiple inconsistent type. It computes the types of object and function values cre- ated at each definition site in the program. If an object or a function value defined at a program location has been observed to assume more than one type during the execution, the analysis reports the program loca- tion along with the observed types. Sometimes these kind of type inconsistencies could point us to a poten- tial bug in the program. We have noticed such issues in two SunSpider benchmark programs.
- Simple Object Allocation Profiler: This dynamic anal- ysis records the number of objects created at a given allocation site and how often the object has been ac- cessed. It reports if the objects created at a given allocation site are read-only or a constant. It also re- ports the maximum and average difference between the object creation time and the most recent access time of the object . If an allocation site creates too many constant objects, then it could lead to memory ineffi- ciency. We have found such a problem in one of the web applications in our benchmark suite.
- Dynamic taint analysis: A dynamic taint analy- sis is a form of information flow analysis which checks if information can flow from a specific set of mem- ory locations, called sources, to another set of memory locations, called sinks. We have implemented a sim- ple form of dynamic taint analysis in Jalangi. In the analysis, we treat read of any field of any object, which has not previously been written by the instrumented source, as a source of taint. We treat any read of a memory location that could change the control-flow of the program as a sink. We attach taint information with the shadow value of an actual value.
