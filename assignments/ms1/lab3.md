---
layout: page
title: "Lab 3: Simple Term Rewriting"
excerpt: "Lab 3: Simple Term Rewriting"
tags: ["assignment"]
context: assign
subcontext: ms1
---

{% include _toc.html %}

This lab is your first encounter with [Stratego](http://metaborg.org/en/latest/source/langdev/meta/lang/stratego/index.html).
You add an outline view and a desugaring transformation to an initial editor provided by us.

## Overview

### Objectives

1. Specify rewrite rules `to-outline-label` which map AST nodes to labels in an outline view. You should include:
  * classes (class name and, if available, parent class name),
  * fields (field name and type),
  * methods (method name, parameter types, return type) and
  * local variables (variable name and type).
2. Define rewrite rules `desugar` which desugar
  * unary expressions into terms of the form `UnExp(op, exp)`,
  * binary expressions into terms of the form `BinExp(op, exp1, exp2)` and
  * octal numbers into decimal numbers.
3. Integrate `desugar` into a strategy `desugar-all` which desugars subtrees in an AST.

For grading, it is required to comply with all constructor names, rule names, and strategy names literally.
{: .notice .notice-warning}

### Submission

You need to submit your MiniJava project with a pull request against branch `assignment3` on GitHub.
The [Git documentation](/documentation/git.html#submitting-an-assignment) explains how to file such a request.

The deadline for submission is October 4th, 23:59.
{: .notice .notice-warning}

### Grading

You can earn up to 40 points for your outline view:

* names in labels (8 points)
* super classes in labels (2 points)
* types in labels (15 points)
* challenges (10 points)
* code quality (5 points)

You can earn up to 60 points for your desugarings:

* signature (20 points)
* rewrite rules (20 points)
* strategy (10 points)
* challenge  (5 points)
* code quality (5 points)

### Early Feedback

This assignment is graded manually. Thus, we do not provide early feedback for this submission.

## Detailed Instructions

### Preliminaries

#### GitHub Repository

We provide you with a template for this assignment in the `assignment4` branch.
See the [Git documentation](/documentation/git.html#template) on how to check out this branch.

### Anatomy of a Spoofax Project

Until now, you mainly worked on files in the `syntax` folder of your project.
During this lab you will also edit files in the `editor` and `trans` folders.
So this is a good point to talk about the general structure of a Spoofax project.

First of all, every Spoofax project is an Eclipse plug-in project.
This allows you to deploy your editor as a plugin using the Eclipse update site mechanism.
Users do not need to have Spoofax installed for using your editor.
You will find the typical administrative boilerplate code for such plug-ins in files
`plugin.xml`, `build.properties`, `META-INF/MANIFEST.MF`, and `editor/java`.

The actual language definition is spread over three folders:

* `syntax` contains all syntax definition files, including the main file `MiniJava.sdf3`.
* `trans` contains all transformation files, including the main file `minijava.str`.
* `editor` contains editor service definition files, including the main file `MiniJava.main.esv`,
  which you have seen already during the last lab.

In the `src-gen` folder, you will find files which were generated from your syntax definition.
For each file `<name>.sdf3`, there are generated files

* `syntax/<name>.sdf`: an SDF2 definition which is equivalent to the SDF3 definition.
* `completions/<name>-esv.esv`: completion templates derived from SDF3 templates.
* `pp/<name>-pp.str`: pretty-printing strategies derived from SDF3 templates.
* `signatures/<name>-sig.str`: signatures derived from SDF3 templates.
* `check/<name>-chk.str`: checks for correctness of abstract syntax trees.

You can find more generated files in `editor` and `include` folders:

* `editor/MiniJava-Folding.generated.esv`: folding patterns derived from your syntax definition.
* `include/MiniJava.def`: your complete syntax definition in SDF2.
* `include/MiniJava-Permissive.def`: a permissive version of the syntax definition, which supports error recovery.
* `include/MiniJava.tbl`: the parse table of your language.
* `include/MiniJava.str`: the signature for ASTs of your language.
* `include/MiniJava-parenthesize.str`: strategies to add parentheses to an AST according to the priorities of your language.
* `include/minijava.ctree` and/or `include/minijava.jar`: compiled Stratego code of your language.

#### Initial Editor Project

We provide you with an initial MiniJava project in the branch `assignment4`.
This project is a common starting point for all of you.
It includes:

* a parse table `reference/sdf.tbl` which passes all syntax tests,
* a corresponding signature `reference/src-gen/signatures/minijava-sig.str`,
* a pretty-printing definition `reference/src-gen/pp/minijava-pp.str`, and a
* a content-completion definition `reference/src-gen/completion/minijava-cp.str` (errors in the completion files can be ignored).


#### Signature

Signatures declare sorts and constructors for terms.
In Spoofax, terms are used to represent abstract syntax trees.
The corresponding signature is generated from the constructors in a syntax definition.
You can find a signature for MiniJava in `reference/src-gen/signatures/minijava-sig.str`.
It was generated from a syntax definition, which itself is not included in the initial project.
If you write your own syntax definition, the generated signatures can be found in `src-gen/signatures/`.

### Outline View

#### Rewrite Rules

An outline view can be specified by rewrite rules `to-outline-label` in `trans/outline.str`.
These rules should rewrite AST nodes to their label in an outline view.
For example, the following rule rewrites a variable declaration to its name, which will be used as a label.

```
rules

  to-outline-label: Var(t, v) -> v
```

On the left-hand side, the rule matches a variable declaration.
During the match, variables `t` and `v` are bound to actual terms.
On the right-hand side, the rule instantiates a label.
During the instantiation, variable `v` is replaced with the term it is bound to.
You can extend `to-outline-label` to provide labels for

* class declarations,
* field declarations and
* method declarations.

When you build the project and open a MiniJava file, you will get an outline of this program in the outline view.
In case you do not see any outline view, you can select it in *Show View* from Eclipse's *Window* menu.

#### Naming Conventions

In Stratego, we use the following naming conventions:

* constructor and sort names: camel case, starting with an upper case (e.g. `Add`, `BinExp`)
* rule names, strategy names, variable names: lower case, multiple words separated by `-` (e.g. `e1`, `project-path`)

#### String Interpolation

In many cases, you want to provide more information than just the name.
For example, you might want to show not only a variable's name, but also it's type.
The following rule achieves this:

```
to-outline-label:
  Var(t, v) -> label
  where
    t'    := <pp-partial-minijava-string> t
  ; label := <concat-strings> [v, ": ", t']
```

On its right-hand side, it produces a `label`, which is bound in the `where` clause.
First, the term bound to `t` is turned into a string bound to `t'` by applying a strategy `pp-partial-minijava-string`.
This strategy is defined in `minijava/trans/pp.str` and uses the pretty-printing rules generated from our SDF3 grammar for MiniJava.

Next, the label is bound to the concatenation of
the string bound to `v`,
a constant string `": "`,
and the string bound to `t'`.

String concatenation is not very intuitive.
Instead, you can also use string interpolation:

```
to-outline-label:
  Var(t, v) -> $[[v]: [t']]
  where
    t' := <pp-partial-minijava-string> t
```

String interpolation allows you to combine text with variables.
Text is enclosed in `$[` and `]`, while variables inside the text are enclosed in `[` and `]`.
These variables need to be bound to strings.

You should provide the following information in your outline labels:

* class name and super class name
* field name and type
* method name, parameter types (not parameter names), return type
* variable name and type

For parameter types, you need to turn a list of parameters into a string.
You can do this with a recursive strategy:

```
pp-params: // empty parameter list
  [] -> ...

pp-params: // single parameter
  [Param(t, p)] -> ...

pp-params: // at least two parameters
  [Param(t, p), param | params] -> ...
  where
    // do something on first parameter
    ...
    // recursive call on remaining parameters
    ... := <pp-params> [param | params]
```

Your current outline view is missing a root node.
You can add a root node by providing a label for programs.

#### Annotations

In Stratego, terms can be annotated with additional information.
The Spoofax outline view uses annotations to determine the icon of a node.
You can specify the icon to use in an annotation:

```
to-outline-label:
  Var(t, v) -> label{icon}
  where
    t'    := <pp-partial-minijava-string> t
  ; label := $[[v]: [t']]
  ; icon  := "icons/var.gif"
```

We do not require you to use icons and you will not earn any points with them.
If you want to use them anyway, you should put the icons into the folder `icons`
and place a proper attribution or license file next to them.

#### Challenge

Challenges are meant to distinguish excellent solutions from good solutions.
Typically, they are less guided and require more investigation or higher programming skills.
{: .notice .notice-success}

1. Provide the file name as the root node label.
<!-- TODO: This is moved -->
See `lib/runtime/editor/origins/` for a suitable strategy.

2. Outline the main method as a subnode of the main class.
You need to change this strategy in the `minijava.str` file:

    ```
    outline := <custom-label-outline(to-outline-label, to-outline-node)> ast
    ```

    <!-- TODO: This is moved -->
    Visit `lib/runtime/editor/outline-library` for inspiration.

3. Use one of the library strategies for folding to implement `pp-params`. You can find various folding strategies in the [API docs](http://releases.strategoxt.org/docs/api/libstratego-lib/stable/docs/).

### Desugaring

A uniform representation of unary and binary expressions eases static analysis and code generation.
To get such a uniform representation, you need to desugar abstract syntax trees during the analysis phase.

#### Signature

Before you can implement a desugaring,
you need to define a signature for the uniform representation of expressions in `trans/desugar.str`:

1. Identify unary and binary expressions in MiniJava.
A unary expression has one subexpression and an operator.
A binary expression has two subexpressions and an operator.
There are more than one kind of unary expressions and more than three kinds of binary expressions in MiniJava.
2. Specify new constants for unary and binary operators in a signature.
Use `UnOp` and `BinOp` as types of these operators.
Again, you should use names based on the semantics of an operator, not on its syntax.
Reading an expression aloud might help you to find suitable constructor names.
3. Define constructors `UnExp` and `BinExp`,
which combine an operator and an expression (respectively two expressions) to an expression.

#### Rewrite Rules

The following rewrite rule defines a rule to desugar an addition:

```
rules

  desugar: Add(e1, e2) -> BinExp(Plus(), e1, e2)
```

This rewrite rule is named `desugar`.
On the left-hand side, the rule matches an addition.
During the match, variables `e1` and `e2` are bound to actual terms.
On the right-hand side, the rule instantiates a binary expression (in a uniform representation).
During the instantiation, variables `e1` and `e2` are replaced with the terms they are bound to.
You can extend `desugar` to replace the different unary and binary expressions in the abstract syntax tree
with a uniform representation of these expressions.
Define a rewrite rule `desugar` in `trans/desugar.str` for every unary or binary operator,
which transforms the original expression into a uniform representation.

#### Editor Integration

To test your transformation, you need to define a builder.
This is done similar to the builder for pretty-printing. Add the following rewrite rule to `trans/minijava.str`:

```
editor-desugar:
  (selected, position, ast, path, project-path) -> (filename, text)
  where
    filename := <guarantee-extension(|"desugared.aterm")> path ;
    text     := <desugar> selected
```

This rule follows Spoofax' convention for strategies which implement editor services.
On the left-hand site, it matches a tuple of

* the `selected` node,
* its `position` in the `ast`,
* the `path` of the current file and
* the `project path`.

On the right-hand site, it instantiates a pair, consisting of a `filename` and the designated `text` of the file.
Both variables are bound in the `where` clause.
The file name is derived from the path of the current file,
while the content of the file is a desugared version of the selected AST node.
You also need to hook your strategy into the editor, making desugaring available in the *Syntax* menu.
You can do this in `editor/Syntax.esv`:

```
action : "Show desugared syntax" = editor-desugar (realtime) (meta) (source)
```

This rule defines

* a builder,
* its label in the *Syntax* menu, and
* its implementation strategy `editor-desugar`.

Annotations can be used for different variants of builders:

* `(openeditor)` from the Syntax menu ensures that a new editor window is opened for the result.
* `(realtime)` requires this editor to be updated whenever the content in the original editor changes.
* `(meta)` restricts the builder to be only available to the language engineer, but not to the language user.
While you can invoke the builder, people who install your MiniJava plugin cannot.
* Finally, `(source)` tells Spoofax to run the builder on an unanalysed (and also not desugared) AST.

#### Strategies

Rewrite rules typically define local transformations inside an AST.
Rewrite rules with the same name define a strategy of this name.
Furthermore, strategies can be defined to orchestrate rewrite rules to complex transformations of complete ASTs.
A strategy consists of a name and a definition, which is typically a combination of strategy applications.
For example, the following strategy orchestrates local desugarings to a desugaring of complete ASTs:

```
strategies

  desugar-all = innermost(desugar)
```

This strategy is named `desugar-all`.
It applies local `desugar` rules.
The application is guided by a generic traversal strategy `innermost`,
which tries to apply its parameter inside a tree, starting at the leaves (bottom-up, left-to-right).
Whenever an application is successful, the result is traversed again.

Same results can be achieved with different generic traversals.
You should try different traversals in `trans/desugar.str`:

* `desugar-all = innermost(desugar)`
* `desugar-all = topdown(desugar)`
* `desugar-all = topdown(try(desugar))`
* `desugar-all = bottomup(desugar)`
* `desugar-all = bottomup(try(desugar))`
* `desugar-all = alltd(desugar)`

Try to understand what is going on and decide for a suitable one.
You can use the library strategy `debug` to print the currently visited node.
For example, `innermost(debug; desugar)` will `debug` all nodes before it tries to `desugar` them.

For the submission, you need to provide an explanation (1 paragraph) of
1. the choice you made,
2. why this choice is suitable for this project, and
3. why other choices would be less suitable.

<!-- TODO: This is outdated -->
#### Editor Integration Revisited

With a builder, you can invoke a transformation manually from any of the menus.
However, desugaring should be an automatic transformation as part of static analysis.
In Spoofax, static analysis is performed by strategies.
In the initial project, these strategies are implemented in `trans/minijava.str`:

```
rules // Analysis

  editor-analyze = analysis-default-editor

  analysis-single-default-interface =
    analysis-single-default(id, id, id|<language>)
  analysis-multiple-default-interface =
    analysis-multiple-default(parse-file <+ !(), id, id, id|<language>, <project-path>)
```

The strategies `analysis-single-default` and `analysis-multiple-default` require various strategy parameters.
The first parameter in `analysis-single-default` and
the second parameter in `analysis-multiple-default` are used for transformations before name analysis.
This is the place, where you should hook-in your desugaring:

```
rules // Analysis

  editor-analyze = analysis-default-editor

  analysis-single-default-interface =
    analysis-single-default(desugar-all, id, id|<language>)
  analysis-multiple-default-interface =
    analysis-multiple-default(parse-file <+ !(), desugar-all, id, id|<language>, <project-path>)
```

To test your implementation, you can use a predefined builder labeled *Show analyzed syntax*.
Open a MiniJava program and run the builder.
At this point, you can get rid of your old desugaring builder.

#### Challenge

Define a desugaring for octal numbers.
In Java, octal numbers start with leading zeros.
Define a rewrite rule which matches such numbers and transforms them to decimal integers.
See the [API docs](http://releases.strategoxt.org/docs/api/libstratego-lib/stable/docs/) for useful helper strategies.
