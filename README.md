# Concrete Syntax Tree

Defining a standard JavaScript CST (concrete syntax tree) to complement ASTs.

## Problem

The standard [SpiderMonkey AST](https://developer.mozilla.org/en-US/docs/SpiderMonkey/Parser_API) format holds only information which is considered necessary for producing valid execution of a JS program. Parsing to this format explicitly discards certain "unnecessary" information such as comments, whitespace, and perhaps ignored syntactic forms like extra `( )` pairs, extra `;`, etc.

Unfortunately, there's a class of emerging tools for which the AST is insufficient as an intermediate representation format, because loss of this information defeats the purpose of the tools. Whitespace, comments, etc need to be representable in the tree, so that a tool which wants to produce or use them, can.

More back-reading on this topic can be found in [this really long thread](https://github.com/Constellation/escodegen/pull/133).

Examples:

* a JSDoc interpreter needs the comments, and in particular, exactly where and how they appear in the code is strictly important information.
* a code formatter may only concern itself with modifying part of a JS program's formatting, and would need to preserve other code exactly as it originally appeared.

## Goal

Our goal is to define an alternate tree format, called a CST (concrete syntax tree), which holds the additional information these other tools need. Of course, any tool could **produce** a CST, not just parsers: transpilers from other languages, code formatters, etc.

We need one standard format that all the various tools can agree to use as an intermediate form, so that processing flows can integrate multiple tools in the handling of a snippet of source code.

## Plan

This is an open group, an ad hoc working committee if you will, where all interested parties can register their interest to participate. Ultimately, it is the tools makers (who must participate to make this work!) who will decide if what we come up with is suitable. The success of this group effort is if tools makers agree and implement what we propose.

We will conduct all discussions on this repo, in the open, or at least post transcribed notes here from any teleconference/online meetings. It seems like a good idea to have at least one initial meeting to kick things off, but we will self-organize here and make such decisions as we go.

### "Register" Interest

Please go to [this issue thread](https://github.com/getify/concrete-syntax-tree/issues/1) and add a comment to "register" your interest in this "committee".

We'll all figure out next steps after people have a chance to join.

## Design

Some stated design biases (to be validated):

* a CST should be pretty similar to the standard AST, as much as possible, so as to make conversion between CST and AST (or vice versa?) straightforward.
* the CST must be a format that a code parser could practically produce while parsing, either by default or by option, as the main use-case of this format is to prevent information-loss during parsing.
* most importantly, a CST format must be suitable for code generators to use while producing standard JS code.

### Current Proposals

There were some discussions on this topic awhile back, and two rough ideas for CST format emerged. These should serve to inform the design process, but not limit/control it.

To illustrate the two proposals' approaches, consider this sample piece of code:

```js
/*1*/ function /*2*/ foo /*3*/ ( /*4*/ ) /*5*/ {
  // ..
}
```

This program is parsed into the following AST:

```js
{
    "type": "Program",
    "body": [
        {
            "type": "FunctionDeclaration",
            "id": {
                "type": "Identifier",
                "name": "foo"
            },
            "params": [],
            "defaults": [],
            "body": {
                "type": "BlockStatement",
                "body": []
            },
            "rest": null,
            "generator": false,
            "expression": false
        }
    ]
}
```

As you can see, there's whitespace and comments appearing in a several different syntactic "positions" in that function declaration header. Now, if you consider the current standard AST, not all of those syntactic positions have natural AST representations where these "extras" (comments, whitespace, etc) can be attached.

For example, `/*4*/` appears inside an empty parameters-list. But how could we represent the whitespace/comments there in the above AST? The `params` list is an array, and in JSON/object-literal format, an array cannot also have properties.

We need a CST to do that.

So, briefly, a CST could look like:

1. These "extras" annotations could be represented by an "extras" property attached to any node, with three standard position names as properties of the "extras" annotation: "before", "inside", "after".

   Since those names aren't sufficient to describe all syntactic forms with just the standard AST, such as `/*4*/` above, we can add additional synthetic/virtual *nodes* to the tree (parent or child nodes) that act as anchors to hang these "extras" annotations in the proper positions.
   
   We could solve this by adding a node called `paramList` in the function declaration, which is an object. This node *could* have as a child property the `params` array seen above. Or it could just be a sibling to `params` in the function declaration. Either way, `paramList` as an object can have a property called `extras`, and in that object, the `inside` position descriptor could be an array to hold the whitespace and comments items that appear in between the `( )`.
   
   The upside is that we only have 3 position names to know about and handle ("before", "inside", "after"). That makes tooling logic easier in the generation and consumption phases. The downside is that it creates extra nodes, which means current tools that deal with ASTs either need a "conversion" which strips the CST down to an AST, or have to change to be able to handle the presence of virtual nodes it should ignore in its processing. It could make some code paths a bit harder, while making other code paths much easier. Tradeoffs abound.

2. Instead of adding extra virtual nodes, we can just limit which nodes can have "extras" annotations to those which are already objects. This means that we would need to have potentially lots more "position descriptors", as "before", "inside", and "after" would not be sufficient in all the various different JS syntactic forms.

  "inside" when attached to a function declaration could always mean "inside the param list", and so whether there are params or not, the whitespace/comments extras can always be represented. In the above example, "inside" kinda works, but there are lots of other syntactic forms where different position names would have to be used. Instead of knowing/handling just 3 positional labels, we might have 5-10 different labels (not all of which made sense in different nodes).
  
  The upside is that a CST is structurally the same as an AST (simplifying some existing tools handling ASTs), with just extra properties which could theoretically be either used or ignored easily. The downside is that producing and consuming these extras is a fair bit more complicated as there are quite a few different positional names to handle.

### Chicken Or The Egg?

Another way to boil down the contrast between these two proposals is: Is an AST a CST, or is a CST an AST? Or, which comes first, the AST or the CST?

The first proposal is that all ASTs are CSTs, but not vice versa. Any (future) tool which could take a (future) CST could just take an AST, and work just fine (without extras, of course!). Any existing tool which expects ASTs would probably need you to "convert" (that is, strip out the virtual nodes) a CST down to a base AST, so that they didn't need to handle all those virtual nodes.
   
The second proposal is essentially the reverse: all CSTs are ASTs, but not vice versa. Any tool which currently takes ASTs can start taking CSTs (without conversion) and not have to change (as long as it's tolerant to ignore properties it doesn't care about, which is sort of an implicit "conversion" from CST to AST). Any tool which would take a (future) CST could take an AST (without extras, of course!).

Neither proposal is perfect. They both imply different tradeoffs. They each imply that some code paths will be simpler and some will be harder. It's not totally clear that either is the right choice, or that another option wouldn't be better. These are things that should be explored and debated.

## License & Assignment

By participating in this group, you are explicitly granting/assigning any and all ideas you present to the public domain. No one will retain any "rights" over anything we present/discuss here in this ad hoc committee process, so that there are no concerns over any intellectual property disputes/concerns.

Do not participate if you have a conflict of interest that prevents you from assigning this content to the public domain.
