
# RDFox Datalog

To efficiently answer queries, RDFox materialises all consequences of the inference rules and the facts so that queries can be answered directly without the rules. The rule language supported by RDFox extends the standard datalog language with stratified negation, stratified aggregation, built-in functions, and more, so as to provide additional data analysis capabilities. In this section, we discuss in detail the structure of an RDFox datalog rule and all supported constructs.

A rule has the form

(this is written in Latex maths format)
$$
H_{1} ,... , H_{j} :- L_{1} ,... , L_{k} .
$$

(this is the original which used pre, code, and sub html tags)
<pre><code>H<sub>1</sub> ,... , H<sub>j</sub> :- L<sub>1</sub> ,... , L<sub>k</sub> .</code></pre>

(this is in markdown, just to show we can label the language, yes this isn't an actual piece of Datalog, but anyway...)
{% tabs %}
{% tab title="Datalog" %}
```Datalog
H_1 ,... , H_j :- L_1 ,... , L_k .
```
{% endtab %}
{% endtabs %}

where the formula to the left of the `:-` operator is the rule head and the formula to the right is the rule *body*. Intuitively, a rule says "if <code>L<sub>1</sub></code>, ..., and <code>L<sub>k</sub></code> all hold, then <code>H<sub>1</sub></code>, ..., and <code>H<sub>j</sub></code> hold as well". Each <code>H<sub>i</sub></code> with 1 &le; i &le; j is an *atom*, and each <code>L<sub>i</sub></code> with 1 &le; i &le; k is a *literal*. A literal is an *atom*, a *negation*, a *bind literal*, a *filter literal*, or an *aggregate literal*. We next explain what these constructs are, and demonstrate how to use them by means of examples.

## Atom

An atom is either an default graph RDF atom or a general atom. General atoms can be used to access data in named graphs and mounted data sources.

### Default Graph RDF Atom

A default graph RDF atom has the form <code>[t<sub>1</sub>, t<sub>2</sub>, t<sub>3</sub>]</code> where <code>t<sub>i</sub></code> is a *term*, which is either an RDF resource or a variable. To distinguish between these two kinds of terms, RDFox requires variables to start with the <code>?</code> symbol. Also note that when <code>t<sub>2</sub></code> is an IRI, atom <code>[t<sub>1</sub>,t<sub>2</sub>,t<sub>3</sub>]</code> can be written alternatively as <code>t<sub>2</sub>[t<sub>1</sub>,t<sub>3</sub>]</code>; moreover, when <code>t<sub>2</sub></code> is the special IRI "rdf:type" and <code>t<sub>3</sub></code> is also an IRI, atom <code>[t<sub>1</sub>,t<sub>2</sub>,t<sub>3</sub>]</code> can be written alternatively as <code>t<sub>3</sub>[t<sub>1</sub>]</code>.

**Example** *A simple rule with default graph RDF atoms only*
```
a1:Person[?X] :- a1:teacherOf[?X, ?Y] .
```
*As we discussed earlier, this is equivalent to:*
```
[?X, rdf:type, a1:Person] :- [?X, a1:teacherOf, ?Y] .
```
*The above rule has only one atom in the rule body and one atom in the rule head. Intuitively, the rule says that if X is a teacher of Y, then X must be a person. Both the body and the head are matched in the default RDF graph.*

### General Atom

A general atom has the form <code>A(t<sub>1</sub>,...,t<sub>n</sub>)</code> with n &ge; 1 where <code>A</code> is an IRI denoting the name of a tuple table and <code>t<sub>1</sub></code>, ..., <code>t<sub>n</sub></code> are terms. Each named RDF graph is represented in RDFox as a tuple table; thus, general atoms can be used to refer to data in named graphs.

**Example** *A rule with both RDF and general atoms*
```
[?ID, fg:firstName, ?FN], [?ID, fg:lastName, ?LN] :- fg:person(?ID,?FN,?LN) .
```
*The general atom in the rule body refers to a tuple table containing three columns. The same rule can be written alternatively as the following.*
```
fg:firstName[?ID, ?FN], fg:lastName[?ID, ?LN] :- fg:person(?ID,?FN,?LN) .
```
*Note the difference in the syntax: `()` are used for general atoms, whereas `[]` are used for RDF atoms.*

**Example** *A accessing named graphs*
```
:Payroll(?ID, :montlyPayment, ?M) :- :Employee[?ID], :HR(?ID,:yearlySalary,?S), BIND(?S / 12 AS ?M) .
```
*This rule joins information from the default graph and the named graph called `HR`, and it inserts consequences into the named graph called `:Payroll`. Specifically, The first body atom of the rule identifies IDs of employees in the default RDF graph. The second body atom is a general atom: it is evaluated in the named graph called `:HR`, and it matches triples that connect IDs with their yearly salaries. The head of the rule contains a general atom that refers to the named graph called `:Payroll`, and it derives triples that connect IDs of employees with their respective monthly payments.*

## Negation

Negation is useful when the user wants to require that certain conditions are not satisfied. A negation has one of the following forms, where k &ge; 2, j &ge; 1, <code>B<sub>1</sub></code>, ..., <code>B<sub>k</sub></code> are atoms, and <code>?V<sub>1</sub></code>, ..., <code>?V<sub>j</sub></code> are variables.

<pre><code>NOT B<sub>1</sub><br>
NOT(B<sub>1</sub>, ..., B<sub>k</sub>)<br>
NOT EXIST ?V<sub>1</sub>, ..., ?V<sub>j</sub> IN B<sub>1</sub><br>
NOT EXIST ?V<sub>1</sub>, ..., ?V<sub>j</sub> IN (B<sub>1</sub>,...,B<sub>k</sub>)<br>
NOT EXISTS ?V<sub>1</sub>, ..., ?V<sub>j</sub> IN B<sub>1</sub><br>
NOT EXISTS ?V<sub>1</sub>,..., ?V<sub>j</sub> IN (B<sub>1</sub>,...,B<sub>k</sub>)
</code></pre>

**Note** RDFox will reject rules that use negation in all `equality` modes other than `off` (see [equality](05-using?id=equality)).

**Example** *Using negation of the first form*

```
a1:stranger[?X,?Y] :- a1:Per(?X), a1:Per(?Y), NOT a1:friend[?X,?Y] .
```
*Intuitively, the rule says that if X is a person, Y is a person, and X and Y are not friends, then they are strangers.*

**Example** *Using negation of the last form*
```
a1:basic[?X] :- a1:com[?X], NOT EXISTS ?Y IN (a1:com[?Y], a1:sub[?Y,?X]) .
```
*Intuitively, the rule says that if X is a component and it does not have any subcomponents, then X is a basic component.*

## Bind Literal

A bind literal evaluates an *expression* and assigns the value of the expression to a variable, or compares the value of the expression with a term. A bind literal is of the following form, where `exp` is an expression and `t` is a term not appearing in `exp`. An expression can be constructed from terms, operators, and functions. The operators and functions supported here are the same as those supported in RDFox SPARQL queries; refer to Section 3 for a detailed comparison between SPARQL 1.1 functions and the ones implemented in RDFox.

<pre><code>BIND(exp AS t)</code></pre>

**Example** *Using bind literals*
```
CTemp[?X, ?Z] :- FTemp[?X, ?Y], BIND ((?Y - 32)/1.8 AS ?Z) .
```
*The bind literal in the above rule converts Fahrenheit degrees to Celsius degrees.*

## Filter Literal

Rule evaluation can be seen as the process of finding satisfying assignments for variables appearing in the rule. A filter literal is of the following form, and it restricts satisfying assignments of variables to those for which the expression `exp` evaluates to true. Thus, when the user writes a filter literal, the expression is expected to provide truth values.

<pre><code>FILTER(exp)</code></pre>

Example *Using filter literals*
```
PosNum[?X] :- Num[?X], FILTER(?X > 0)
```
*The rule says that a number is positive if it is larger than zero.*

## Aggregate Literal

An aggregate literal applies an *aggregate function* to groups of values to produce one value for each group. An aggregate literal has the form

<pre><code>AGGREGATE(B<sub>1</sub>, ..., B<sub>k</sub> ON ?X<sub>1</sub>, ..., ?X<sub>j</sub> BIND f<sub>1</sub>(exp<sub>1</sub>) AS t<sub>1</sub> ... BIND f<sub>n</sub>(exp<sub>n</sub>) AS t<sub>n</sub>)</code></pre>

where k &ge; 1, j &ge; 0, n &ge; 1, and

- <code>B<sub>1</sub>, ..., B<sub>k</sub></code> are atoms,

- <code>?X<sub>1</sub></code>, ..., <code>?X<sub>j</sub></code> are variables appearing in <code>B<sub>1</sub></code>, ..., <code>B<sub>k</sub></code>,

- <code>exp<sub>1</sub></code>, ..., <code>exp<sub>n</sub></code> are expressions constructed using variables from <code>B<sub>1</sub></code>, ..., <code>B<sub>k</sub></code>,

- <code>f<sub>1</sub></code>, ..., <code>f<sub>n</sub></code> are aggregate functions, and

- <code>t<sub>1</sub></code>, ..., <code>t<sub>n</sub></code> are constants or variables that do not appear in <code>B<sub>1</sub></code>, ..., <code>B<sub>k</sub></code>.

Sometimes the user might be interested in computing an aggregate value from a set of *distinct* values. In this case, the keyword "distinct" can be used in front of an expression <code>exp<sub>i</sub></code>.

**Note** RDFox will reject rules that use aggregation in all `equality` modes other than `off` (see [equality](05-using?id=equality)).

**Example** *Using aggregate literals*
```
MinTemp[?X,?Z] :- City[?X], AGGREGATE(Temp[?X,?Y] ON ?X BIND MIN(?Y) AS ?Z) .
```
*Intuitively, the above rule computes a minimum temperature for each city.*

**Example** *Using the keyword distinct*
```
FamilyFriendCnt[?X,?Cnt] :-
    Family[?X],
	AGGREGATE(HasMem[?X,?Y], HasFriend[?Y,?Z] ON ?X BIND COUNT(DISTINCT ?Z) AS ?Cnt) .
```
*This rule counts the number of different friends for each family; a person is considered a friend of a family if he is a friend of a member of the family.*
