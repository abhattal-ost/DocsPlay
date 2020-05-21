# Datalog

To efficiently answer queries, RDFox materialises all consequences of the inference rules and the facts so that queries can be answered directly without the rules. The rule language supported by RDFox extends the standard datalog language with stratified negation, stratified aggregation, built-in functions, and more, so as to provide additional data analysis capabilities. In this section, we discuss in detail the structure of an RDFox datalog rule and all supported constructs.

A rule has the form

\(this is written in Latex maths format, with a bit of buggering around with spacing around the :- operator\)

$$
H_{1} ,... , H_{j} :\!\!-\: L_{1} ,... , L_{k} .
$$

\(this is the original which used pre, code, and sub html tags, note that the subscripts have gone missing\)

```text
H1 ,... , Hj :- L1 ,... , Lk .
```

\(this is in markdown, just to show we can label the language, yes this isn't an actual piece of Datalog, but anyway...\)

{% tabs %}
{% tab title="Datalog" %}
```text
H_1 ,... , H_j :- L_1 ,... , L_k .
```
{% endtab %}
{% endtabs %}

where the formula to the left of the `:-` operator is the rule head and the formula to the right is the rule _body_. Intuitively, a rule says "if `L1`, ..., and `Lk` all hold, then `H1`, ..., and `Hj` hold as well". Each `Hi` with 1 ≤ i ≤ j is an _atom_, and each `Li` with 1 ≤ i ≤ k is a _literal_. A literal is an _atom_, a _negation_, a _bind literal_, a _filter literal_, or an _aggregate literal_. We next explain what these constructs are, and demonstrate how to use them by means of examples.

## Atom

An atom is either an default graph RDF atom or a general atom. General atoms can be used to access data in named graphs and mounted data sources.

### Default Graph RDF Atom

A default graph RDF atom has the form `[t1, t2, t3]` where `ti` is a _term_, which is either an RDF resource or a variable. To distinguish between these two kinds of terms, RDFox requires variables to start with the `?` symbol. Also note that when `t2` is an IRI, atom `[t1,t2,t3]` can be written alternatively as `t2[t1,t3]`; moreover, when `t2` is the special IRI "rdf:type" and `t3` is also an IRI, atom `[t1,t2,t3]` can be written alternatively as `t3[t1]`.

**Example** _A simple rule with default graph RDF atoms only_

```text
a1:Person[?X] :- a1:teacherOf[?X, ?Y] .
```

_As we discussed earlier, this is equivalent to:_

```text
[?X, rdf:type, a1:Person] :- [?X, a1:teacherOf, ?Y] .
```

_The above rule has only one atom in the rule body and one atom in the rule head. Intuitively, the rule says that if X is a teacher of Y, then X must be a person. Both the body and the head are matched in the default RDF graph._

### General Atom

A general atom has the form `A(t1,...,tn)` with n ≥ 1 where `A` is an IRI denoting the name of a tuple table and `t1`, ..., `tn` are terms. Each named RDF graph is represented in RDFox as a tuple table; thus, general atoms can be used to refer to data in named graphs.

**Example** _A rule with both RDF and general atoms_

```text
[?ID, fg:firstName, ?FN], [?ID, fg:lastName, ?LN] :- fg:person(?ID,?FN,?LN) .
```

_The general atom in the rule body refers to a tuple table containing three columns. The same rule can be written alternatively as the following._

```text
fg:firstName[?ID, ?FN], fg:lastName[?ID, ?LN] :- fg:person(?ID,?FN,?LN) .
```

_Note the difference in the syntax: `()` are used for general atoms, whereas `[]` are used for RDF atoms._

**Example** _A accessing named graphs_

```text
:Payroll(?ID, :montlyPayment, ?M) :- :Employee[?ID], :HR(?ID,:yearlySalary,?S), BIND(?S / 12 AS ?M) .
```

_This rule joins information from the default graph and the named graph called `HR`, and it inserts consequences into the named graph called `:Payroll`. Specifically, The first body atom of the rule identifies IDs of employees in the default RDF graph. The second body atom is a general atom: it is evaluated in the named graph called `:HR`, and it matches triples that connect IDs with their yearly salaries. The head of the rule contains a general atom that refers to the named graph called `:Payroll`, and it derives triples that connect IDs of employees with their respective monthly payments._

## Negation

Negation is useful when the user wants to require that certain conditions are not satisfied. A negation has one of the following forms, where k ≥ 2, j ≥ 1, `B1`, ..., `Bk` are atoms, and `?V1`, ..., `?Vj` are variables.

```text
NOT B1

NOT(B1, ..., Bk)

NOT EXIST ?V1, ..., ?Vj IN B1

NOT EXIST ?V1, ..., ?Vj IN (B1,...,Bk)

NOT EXISTS ?V1, ..., ?Vj IN B1

NOT EXISTS ?V1,..., ?Vj IN (B1,...,Bk)
```

**Note** RDFox will reject rules that use negation in all `equality` modes other than `off` \(see [equality](https://github.com/abhattal-ost/DocsPlay/tree/1b8fc58168cea59f6ab7341431e96d433a357014/05-using?id=equality/README.md)\).

**Example** _Using negation of the first form_

```text
a1:stranger[?X,?Y] :- a1:Per(?X), a1:Per(?Y), NOT a1:friend[?X,?Y] .
```

_Intuitively, the rule says that if X is a person, Y is a person, and X and Y are not friends, then they are strangers._

**Example** _Using negation of the last form_

```text
a1:basic[?X] :- a1:com[?X], NOT EXISTS ?Y IN (a1:com[?Y], a1:sub[?Y,?X]) .
```

_Intuitively, the rule says that if X is a component and it does not have any subcomponents, then X is a basic component._

## Bind Literal

A bind literal evaluates an _expression_ and assigns the value of the expression to a variable, or compares the value of the expression with a term. A bind literal is of the following form, where `exp` is an expression and `t` is a term not appearing in `exp`. An expression can be constructed from terms, operators, and functions. The operators and functions supported here are the same as those supported in RDFox SPARQL queries; refer to Section 3 for a detailed comparison between SPARQL 1.1 functions and the ones implemented in RDFox.

```text
BIND(exp AS t)
```

**Example** _Using bind literals_

```text
CTemp[?X, ?Z] :- FTemp[?X, ?Y], BIND ((?Y - 32)/1.8 AS ?Z) .
```

_The bind literal in the above rule converts Fahrenheit degrees to Celsius degrees._

## Filter Literal

Rule evaluation can be seen as the process of finding satisfying assignments for variables appearing in the rule. A filter literal is of the following form, and it restricts satisfying assignments of variables to those for which the expression `exp` evaluates to true. Thus, when the user writes a filter literal, the expression is expected to provide truth values.

```text
FILTER(exp)
```

Example _Using filter literals_

```text
PosNum[?X] :- Num[?X], FILTER(?X > 0)
```

_The rule says that a number is positive if it is larger than zero._

## Aggregate Literal

An aggregate literal applies an _aggregate function_ to groups of values to produce one value for each group. An aggregate literal has the form

```text
AGGREGATE(B1, ..., Bk ON ?X1, ..., ?Xj BIND f1(exp1) AS t1 ... BIND fn(expn) AS tn)
```

where k ≥ 1, j ≥ 0, n ≥ 1, and

* `B1, ..., Bk` are atoms,
* `?X1`, ..., `?Xj` are variables appearing in `B1`, ..., `Bk`,
* `exp1`, ..., `expn` are expressions constructed using variables from `B1`, ..., `Bk`,
* `f1`, ..., `fn` are aggregate functions, and
* `t1`, ..., `tn` are constants or variables that do not appear in `B1`, ..., `Bk`.

Sometimes the user might be interested in computing an aggregate value from a set of _distinct_ values. In this case, the keyword "distinct" can be used in front of an expression `expi`.

**Note** RDFox will reject rules that use aggregation in all `equality` modes other than `off` \(see [equality](https://github.com/abhattal-ost/DocsPlay/tree/1b8fc58168cea59f6ab7341431e96d433a357014/05-using?id=equality/README.md)\).

**Example** _Using aggregate literals_

```text
MinTemp[?X,?Z] :- City[?X], AGGREGATE(Temp[?X,?Y] ON ?X BIND MIN(?Y) AS ?Z) .
```

_Intuitively, the above rule computes a minimum temperature for each city._

**Example** _Using the keyword distinct_

```text
FamilyFriendCnt[?X,?Cnt] :-
    Family[?X],
    AGGREGATE(HasMem[?X,?Y], HasFriend[?Y,?Z] ON ?X BIND COUNT(DISTINCT ?Z) AS ?Cnt) .
```

_This rule counts the number of different friends for each family; a person is considered a friend of a family if he is a friend of a member of the family._

