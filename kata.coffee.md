Kata
====

Kata is an idea for teaching software mechanics. I [wrote about it
here](https://samgentle.com/posts/2015-08-09-kata) but the short version is
that maybe we could do a better job of training people in the base skills of
programming: rearranging functions and arguments, abstracting similar code
into functions or loops, punching a variable down through a bunch of nested
functions, that sort of thing.

It might seem mindless and mechanical, and maybe it is, but for a lot of
people starting out having a way to practice these skills could make a
difference in how quickly they reach the higher-level skills. After all, you
can't really be thinking high-level if you're spending a lot of energy on
simple things.

This prototype only handles one particular transform: refactoring a function
with nested conditions into using early-exit guard conditions at the
top of the function.

You can try this out by visiting [the demo site](https://demos.samgentle.com/kata).


Plumbing
--------

Require (or ghetto require if we're in the browser)

    escodegen = if typeof window is 'undefined' then require 'escodegen' else window.escodegen
    esprima = if typeof window is 'undefined' then require 'esprima' else window.esprima


General useful things

    parse = (str) -> esprima.parse("(#{str})").body[0].expression
    generate = escodegen.generate


Code generation DSL
-------------------

We want to be able to generate all of these things at the same time:

1. The expression itself
2. A human-friendly function name
3. The free variables used by the expression (so we can put them in the function node)
4. The list of constraints (so we can put them in a guard conditional)

We do that by building and operating on our own AST-like object.


Free variable generation. Super useful but a bit gross that it's not scoped to
anything, so you have to reset it manually.

    nextFree = 1
    letterMap = 'abcdefghijklmnopqrstuvwxyz'
    numToLetters = (num) ->
      letters = ''
      while num > 0
        letters = letterMap[(num - 1) % 26] + letters
        num = (num - 1) // 26
      letters
    freeVar = -> numToLetters nextFree++
    freeVar.reset = -> nextFree = 1

Number generation. We only generate numbers (and lists of numbers) right now.

    num = ->
      variable = freeVar()
      description: ["number"]
      code:
        type: 'Identifier'
        name: variable
      variables: [variable]

    nums = (n=2) ->
      elems = (type: 'Identifier', name: freeVar() for [1..n])
      description: ["numbers"]
      code:
        type: 'ArrayExpression'
        elements: elems
      variables: (elem.name for elem in elems)


Constraints
-----------

These are helpers for handling multiple variables in constraints.

    listify = (names) ->
      return names[0] if names.length is 1
      names.slice(0,-1).join(', ') + ' and ' + names[names.length-1]

    andify = (expr, newexpr) ->
      type: 'LogicalExpression'
      operator: '&&'
      left: expr
      right: newexpr

This is the constraint factory, we give it a function that transforms
expressions into constraint checks, and it does the rest.

    constraint = (name, constrainer) -> (expr) ->
      description: [name].concat(expr.description or [])
      code: expr.code
      constraints: [
        description: "#{listify expr.variables} must be #{name}"
        code: (constrainer(type: 'Identifier', name: v) for v in expr.variables).reduce(andify)
      ].concat(expr.constraints or [])
      variables: expr.variables

And now the various constraints:

    even = constraint 'even', (expr) ->
      type: 'BinaryExpression'
      operator: '=='
      left:
        type: 'BinaryExpression'
        operator: '%'
        left: expr
        right:
          type: 'Literal'
          value: 2
      right:
        type: 'Literal'
        value: 0

    odd = constraint 'odd', (expr) ->
      type: 'BinaryExpression'
      operator: '!='
      left:
        type: 'BinaryExpression'
        operator: '%'
        left: expr
        right:
          type: 'Literal'
          value: 2
      right:
        type: 'Literal'
        value: 0

    positive = constraint 'positive', (expr) ->
      type: 'BinaryExpression'
      operator: '>='
      left: expr
      right:
        type: 'Literal'
        value: 0

    negative = constraint 'negative', (expr) ->
      type: 'BinaryExpression'
      operator: '<'
      left: expr
      right:
        type: 'Literal'
        value: 0


Operators
---------

We only do binary operations. These have to be a little tricky because they
generate very different human-strings from their homogenous analogues. ie. we
divide 3 *by* 4, but we subtract 3 *from* 4

    binaryOp = (name, preposition, operator) -> (num1, num2) ->
        description: [name].concat(num1.description or [], preposition, num2.description or [])
        code:
          type: 'BinaryExpression'
          operator: operator
          left: num1.code
          right: num2.code
        constraints: (num1.constraints or []).concat num2.constraints or []
        variables: (num1.variables or []).concat num2.variables or []

    add      = binaryOp 'add',      'to',   '+'
    subtract = binaryOp 'subtract', 'from', '-'
    multiply = binaryOp 'multiply', 'by',   '*'
    divide   = binaryOp 'divide',   'by',   '/'


To handle homogenous binary operations (ie where all arguments have the same
constraints), we unpack them out of the array and build up a big tree of
operations with the opreducer.

    opreducer = (op) -> (vals, val) ->
      type: 'BinaryExpression'
      operator: op
      left: vals
      right: val

    basicNums = 'zero one two three four five six seven eight nine ten'.split(' ')

    hBinaryOp = (name, operator) -> (expr) ->
      elems = expr.code.elements
      throw new Error "Need at least 2 elements to #{name}" if !elems or elems.length < 2
      count = (basicNums[elems.length] or elems.length)
      description: [name,count].concat(expr.description)
      code: elems.reduce(opreducer(operator))
      constraints: expr.constraints
      variables: expr.variables

    hadd      = hBinaryOp 'add',      '+'
    hsubtract = hBinaryOp 'subtract', '-'
    hmultiply = hBinaryOp 'multiply', '*'
    hdivide   = hBinaryOp 'divide',   '/'


Some useful helpers for particular AST bits we use a lot

    blockWrap = (code) ->
      type: 'BlockStatement'
      body: [code]

    thrower = (constraint) ->
      type: 'ThrowStatement'
      argument:
        type: 'NewExpression'
        callee:
          type: 'Identifier'
          name: 'Error'
        arguments: [
          type: 'Literal'
          value: constraint.description
        ]


Constraint generation
---------------------

We turn the constraints into checks in one of two ways, either as traditional
nested conditions, or as early-exit guards.

    checkConstraints = (expr) ->
      code = {
        type: "ReturnStatement"
        argument: expr.code
      }
      constraints = expr.constraints or []
      for constraint in constraints by -1
        code =
          type: "IfStatement"
          test: constraint.code
          consequent: blockWrap code
          alternate: blockWrap thrower constraint

      description: expr.description
      code: blockWrap code
      constraints: expr.constraints
      variables: expr.variables


    guardConstraints = (expr) ->
      statements = []
      for constraint in expr.constraints or []
        statements.push
          type: 'IfStatement'
          test: negate constraint.code
          consequent: thrower constraint

      statements.push
        type: 'ReturnStatement'
        argument: expr.code

      description: expr.description
      code:
        type: 'BlockStatement'
        body: statements
      constraints: expr.constraints
      variables: expr.variables

    camelise = (description) ->
      for word, i in description
        if i is 0 then word else word[0].toUpperCase() + word.slice(1)


Finally, we can generate the wrapper function and our AST is complete.

    fun = (expr) ->
      type: 'FunctionExpression'
      id:
        type: "Identifier"
        name: camelise(expr.description).join('')
      params: ({type: "Identifier", name: x} for x in expr.variables)
      body: if expr.code.type is 'BlockStatement' then expr.code else blockWrap expr.code


AST Munging
-----------

In order to make guard conditions work, we need to be able to negate them. We
could just throw a unary `!` in there, but it looks super ugly, and we'd need
to normalise it anyway because most people don't write code like that.

Instead, we walk the AST and do the appropriate gymnastics to negate the
condition properly.

    opposites = {}
    ops = '== === < > && || <= >= !== !='.split ' '
    for x, i in ops
      opposites[x] = ops[ops.length-1-i]

    negate = (expr) ->
      type = expr.type; op = expr.operator
      if type is 'BinaryExpression' and opposites[op]?
        expr.operator = opposites[op]
        expr

      else if type is 'LogicalExpression'
        expr.operator = opposites[op]
        expr.left = negate expr.left
        expr.right = negate expr.right
        expr

      else if type is 'UnaryExpression' and op is '!'
        expr.argument

      else
        type: 'UnaryExpression'
        operator: '!'
        prefix: true
        argument: expr


Even so, we still want to normalise the AST we take in from the user. It might
have all sorts of weird issues, but for now we only concentrate on making one-
line if statements equivalent to multiline if statements with a block.

We can also normalise negations, but it's off by default because just wrapping
your whole expression in a `not` is bad style.

    defaultopts = {blocks: true, negation: false}
    normalise = (expr, opts=defaultopts) ->
      switch expr?.type
        when 'UnaryExpression'
          break unless opts.negation
          expr = negate expr.argument if expr.operator is '!'

        when 'IfStatement'
          break unless opts.blocks
          if expr.consequent.type isnt 'BlockStatement'
            expr.consequent = blockWrap expr.consequent
          if expr.alternate and expr.alternate.type isnt 'BlockStatement'
            expr.alternate = blockWrap expr.alternate

      if expr?.type or Array.isArray(expr)
        expr[k] = normalise v, opts for k, v of expr when v? and typeof v is 'object'

      expr


Random generation
-----------------

Now that we have all our generating/pseudo-AST code set up, we just need to
generate random permutations. That's actually pretty easy because everything
is just function calls.

    randOf = (args...) -> args[Math.floor(Math.random() * args.length)]

    hetRand = ->
      first = randOf(even, odd, positive, negative)
      second = first
      second = randOf(even, odd, positive, negative) while second == first
      randOf(add, subtract, multiply, divide) (first num()), (second num())

    homRand = ->
      randOf(hadd, hsubtract, hmultiply, hdivide) randOf(even, odd, positive, negative) nums(randOf(2, 3))

    randBody = -> randOf(hetRand, homRand)()


Exercise generation
-------------------

We can now generate question/answer pairs for testing.

    gen = ->
      freeVar.reset()
      body = randBody()
      question: fun checkConstraints body
      answer: normalise fun guardConstraints JSON.parse(JSON.stringify(body))

    maybeGenerate = (x) ->
      try
        generate x
      catch
        null


    addPath = (path, tree, k) ->
      path = path.slice()
      path[path.length-1] += if isNaN(k) then ".#{k}" else "[#{k}]"
      sub = tree?.type
      path.push sub if sub
      path

    diff = (tree1, tree2, path = [tree2.type]) ->
      return null if tree1 == tree2
      return [tree1, tree2, path] if typeof tree1 != typeof tree2 or Array.isArray(tree1) != Array.isArray(tree2)
      if typeof tree1 is 'object'
        return [tree1, tree2, path] if tree1.type != tree2.type
        for k of tree1 when d = diff tree1[k], tree2[k], addPath(path, tree2[k], k)
          return d
        return null
      return [tree1, tree2, path]

Export
------

    ex = {normalise, diff, gen, generate, parse, maybeGenerate}

    if typeof module is 'undefined' and typeof 'window' isnt 'undefined'
      window.kata = ex
    else
      module.exports = ex
