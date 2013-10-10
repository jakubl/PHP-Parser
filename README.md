PHP Parser
==========

This is a PHP parser written in PHP. It's purpose is to simplify static code analysis and
manipulation.

***Note: This project is experimental. There are no known bugs in the parser itself, but the API is
subject to change.***

Components
==========

This package currently bundles several components:

 * The `Parser` itself


 * A `NodeDumper` to dump the nodes to a human readable string representation
 * A `NodeTraverser` to traverse and modify the node tree
 * A `PrettyPrinter` to translate the node tree back to PHP

Autoloader
----------

In order to automatically include required files `PHPParser_Autoloader` can be used:

    require_once 'path/to/PHP-Parser/lib/PHPParser/Autoloader.php';
    PHPParser_Autoloader::register();

Parser and Parser_Debug
-----------------------

Parsing is performed using `PHPParser_Parser->parse()`. This method accepts a `PHPParser_Lexer`
as the only parameter and returns an array of statement nodes. If an error occurs it throws a
PHPParser_Error.

    $code = '<?php // some code';

    try {
        $parser = new PHPParser_Parser;
        $stmts = $parser->parse(new PHPParser_Lexer($code));
    } catch (PHPParser_Error $e) {
        echo 'Parse Error: ', $e->getMessage();
    }

The `PHPParser_Parser_Debug` class also parses PHP code, but outputs a debug trace while doing so.

Node Tree
---------

The output of the parser is an array of statement nodes. All nodes implement the `PHPParser_Node`
interface (and extend `PHPParser_NodeAbstract`). Furthermore nodes are divided into three categories:

 * `PHPParser_Node_Stmt`: A statement
 * `PHPParser_Node_Expr`: An expression
 * `PHPParser_Node_Scalar`: A scalar (which is a string, a number, aso.)
   `PHPParser_Node_Scalar` inherits from `PHPParser_Node_Expr`.

Each node may have subnodes. For example `PHPParser_Node_Expr_Plus` has two subnodes, namely `left`
and `right`, which represent the left hand side and right hand side expressions of the plus operation.
Subnodes are accessed as normal properties:

    $node->left

The subnodes which a certain node can have are documented as `@property` doccomments in the
respective files.

Additionally all nodes have two methods, `getLine()` and `getDocComment()`.
`getLine()` returns the line a node started in.
`getDocComment()` returns the doccomment before the node or `null` if there was none.

NodeDumper
----------

Nodes can be dumped into a string representation using the `PHPParser_NodeDumper->dump()` method:

    $code = <<<'CODE'
    <?php
        function printLine($msg) {
            echo $msg, "\n";
        }

        printLine('Hallo World!!!');
    CODE;

    try {
        $parser = new PHPParser_Parser;
        $stmts = $parser->parse(new PHPParser_Lexer($code));

        $nodeDumper = new PHPParser_NodeDumper;
        echo '<pre>' . htmlspecialchars($nodeDumper->dump($stmts)) . '</pre>';
    } catch (PHPParser_Error $e) {
        echo 'Parse Error: ', $e->getMessage();
    }

This script will have an output similar to the following:

    array(
        0: Stmt_Func(
            byRef: false
            name: printLine
            params: array(
                0: Stmt_FuncParam(
                    type: null
                    name: msg
                    byRef: false
                    default: null
                )
            )
            stmts: array(
                0: Stmt_Echo(
                    exprs: array(
                        0: Variable(
                            name: msg
                        )
                        1: Scalar_String(
                            value:

                        )
                    )
                )
            )
        )
        1: Expr_FuncCall(
            func: Name(
                parts: array(
                    0: printLine
                )
            )
            args: array(
                0: Arg(
                    value: Scalar_String(
                        value: Hallo World!!!
                    )
                    byRef: false
                )
            )
        )
    )

NodeTraverser
-------------

The node traverser allows traversing the node tree using a visitor class. A visitor class must
implement the `NodeVisitor` interface, which defines the following four methods:

    public function beforeTraverse(array $nodes);
    public function enterNode(PHPParser_Node $node);
    public function leaveNode(PHPParser_Node $node);
    public function afterTraverse(array $nodes);

The `beforeTraverse` method is called once before the traversal begins and is passed the nodes the
traverser was called with. This method can be used for resetting values before traversation or
preparing the tree for traversal.

The `afterTraverse` method is similar to the `beforeTraverse` method, with the only difference that
it is called once after the traversal.

The `enterNode` and `leaveNode` methods are called on every node, the former when it is entered,
i.e. before its subnodes are traversed, the latter when it is left.

All four methods can either return the changed node or not return at all (or return `null`) in which
case the current node is not changed. The `leaveNode` method can furthermore return two special
values: If `false` is returned the current node will be removed from the parent array. If an `array`
is returned the current node will be merged into the parent array at the offset of the current node.
I.e. if in `array(A, B, C)` the node `B` should be replaced with `array(X, Y, Z)` the result will be
`array(A, X, Y, Z, C)`.

The above described visitors are registered in the `NodeTraverser` class:

    $visitor = new MyVisitor;

    $traverser = new PHPParser_NodeTraverser;
    $traverser->addVisitor($visitor);

    $stmts = $parser->parse($lexer);
    $stmts = $traverser->traverse($stmts);

With `MyVisitor` being something like that:

    class MyVisitor extends PHPParser_NodeVisitorAbstract
    {
        public function enterNode(PHPParser_Node $node) {
            // ...
        }
    }

As you can see above you don't need to define all four methods if you extend
`PHPParser_NodeVisitorAbstract` instead of directly implementing the interface.

PrettyPrinter
-------------

The pretty printer compiles nodes back to PHP code. "Pretty printing" here is just the formal
name of the process and does not mean that the output is in any way pretty.

    $prettyPrinter = new PHPParser_PrettyPrinter_Zend;
    echo '<pre>' . htmlspecialchars($prettyPrinter->prettyPrint($stmts)) . '</pre>';

For the code mentioned in the above section this should create the output:

    function printLine($msg)
    {
        echo $msg, "\n";
    }
    printLine('Hallo World!!!');

You can also pretty print only a single expression using the `prettyPrintExpr()` method.