# C++ Expression Parser • [ ![Build Status] [travis-image] ] [travis] [ ![License] [license-image] ] [license]

[travis-image]: https://travis-ci.org/bamos/cpp-expression-parser.png?branch=master
[travis]: http://travis-ci.org/bamos/cpp-expression-parser

[release-image]: http://img.shields.io/badge/release-0.2.1-blue.svg?style=flat
[releases]: https://github.com/cmusatyalab/openface/releases

[license-image]: http://img.shields.io/badge/license-MIT-blue.svg?style=flat
[license]: LICENSE.mit

This project provides a C++ library to parse a character sequence
as an expression using Dijkstra's
[Shunting-yard algorithm](http://en.wikipedia.org/wiki/Shunting-yard_algorithm),
which modifies
[Jesse Brown's original code](http://www.daniweb.com/software-development/cpp/code/427500/calculator-using-shunting-yard-algorithm).

*This project was created by [Brandon Amos](http://bamos.github.io) and
contains substantial modifications from Vinícius Garcia.*

---

# Modifications and Uses of This Code
+ In [Andrew Steiner's](https://github.com/awsteiner)
  [o2scl](https://github.com/awsteiner/o2scl):
  + Adds simple unary functions like sin, cos, and exp.
  + [shunting_yard.cpp](https://github.com/awsteiner/o2scl/blob/master/src/base/shunting_yard.cpp)
  + [shunting_yard.h](https://github.com/awsteiner/o2scl/blob/master/src/base/shunting_yard.h)
  + [shunting_yard_ts.cpp](https://github.com/awsteiner/o2scl/blob/master/src/base/shunting_yard_ts.cpp)

# Minimal example.

```C
#include <iostream>
#include "shunting-yard.h"

int main() {
  TokenMap vars;
  vars["pi"] = 3.14;
  std::cout << calculator::calculate("-pi+1", &vars) << std::endl;

  // Or if you want to evaluate an expression
  // several times efficiently:
  calculator c1("pi-b");
  vars["b"] = 0.14;
  std::cout << c1.eval(&vars) << std::endl; // 3
  vars["b"] = 2.14;
  std::cout << c1.eval(&vars) << std::endl; // 1

  return 0;
}
```

# More examples.
 + See `test-shunting-yard.cpp`.

# Features.
 + Unary operators. +, -
 + Binary operators. +, -, /, *, %, <<, >>, ^
 + Boolean operators. <, >, <=, >=, ==, !=, &&, ||
 + Map of variable names.
 + Functions. sin, cos, tan, abs, print
 + Easy to add new operators, functions and even new types.

# Adding a binary operator.
To add a binary operator,

 1. Update the operator precedence map in `calculator::buildOpPrecedence`.
 2. Add the computation to `calculator::calculate`.

# Implementation Details
The main steps of the calculation process are:

 1. Create the operator precedence map.
 2. Convert to [RPN](http://en.wikipedia.org/wiki/Reverse_Polish_notation)
    with Dijkstra's Shunting-yard algorithm.
 3. Evaluate the expression in RPN form.

## Converting to RPN.
Most of the Shunting-yard algorithm resides here.
The idea is to do everything in one pass for elegance.
Please see the
[source code](https://github.com/bamos/cpp-expression-parser/blob/master/shunting-yard.cpp)
for implementation-specific details,
and refer to the pruned code below for a summary.

```C++
TokenQueue_t calculator::toRPN(const char* expr,
    std::map<std::string, double>* vars,
    std::map<std::string, int> opPrecedence) {
  TokenQueue_t rpnQueue; std::stack<std::string> operatorStack;

  while (*expr ) {
    if (isdigit(*expr )) {
      // If the token is a number, add it to the output queue.
    } else if (isvariablechar(*expr )) {
      // If the function is a variable, resolve it and
      // add the parsed number to the output queue.
    } else {
      // Otherwise, the variable is an operator or parenthesis.
      switch (*expr) {
        case '(':
          operatorStack.push("(");
          ++expr;
          break;
        case ')':
          while (operatorStack.top().compare("(")) {
            rpnQueue.push(new Token<std::string>(operatorStack.top()));
            operatorStack.pop();
          }
          operatorStack.pop();
          ++expr;
          break;
        default:
          {
            // The token is an operator.
            //
            // Let p(o) denote the precedence of an operator o.
            //
            // If the token is an operator, o1, then
            //   While there is an operator token, o2, at the top
            //       and p(o1) <= p(o2), then
            //     pop o2 off the stack onto the output queue.
            //   Push o1 on the stack.
          }
      }
    }
  }
  while (!operatorStack.empty()) {
    rpnQueue.push(new Token<std::string>(operatorStack.top()));
    operatorStack.pop();
  }
  return rpnQueue;
}
```


## Evaluating RPN form.
The RPN is represented as tokens in a stack.
To evaluate this, pop all of the elements off and handle
operations when encountered.


```C++
std::stack<double> evaluation;
while (!rpn.empty()) {
  TokenBase* base = rpn.front();
  rpn.pop();

  if (base->type == OP) {
    Token<std::string>* strTok = static_cast<Token<std::string>*>(base);
    std::string str = strTok->val;
    if (evaluation.size() < 2) {
      throw std::domain_error("Invalid equation.");
    }
    double right = evaluation.top(); evaluation.pop();
    double left  = evaluation.top(); evaluation.pop();
    if (!str.compare("+")) {
      evaluation.push(left + right);
    } else if (!str.compare("*")) {
      evaluation.push(left * right);
    } else if (!str.compare("-")) {
      evaluation.push(left - right);
    } else if (!str.compare("/")) {
      evaluation.push(left / right);
    } else if (!str.compare("<<")) {
      evaluation.push((int) left << (int) right);
    } else if (!str.compare(">>")) {
      evaluation.push((int) left >> (int) right);
    } else {
      throw std::domain_error("Unknown operator: '" + str + "'.");
    }
  } else if (base->type == NUM) {
    Token<double>* doubleTok = static_cast<Token<double>*>(base);
    evaluation.push(doubleTok->val);
  } else {
    throw std::domain_error("Invalid token.");
  }
  delete base;
}
```

The evaluated value resides in `evaluation.top` of type double.
