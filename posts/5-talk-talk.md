---
title: TalkTalk
author: Pat Nakajima
publishedAt: 11/13/2024
excerpt: I was making a programming language but I'm not anymore but maybe I will again.
tags: remember me?
---

# TalkTalk

Hey remember when I started a blog and said I was gonna write about stuff? Me neither. Anyways in like July or June I read [Crafting Interpreters](https://craftinginterpreters.com) and it broke my brain (in a good way!) so I spent  a lil while building a programming language.

It's called TalkTalk and it's not named that because of Charli XCX's track but actually it is. The idea was a language written in Swift with a Swift-esque syntax but not requiring type annotations that could maybe be used to script Shortcuts or just running in an iOS app for fun. But really the idea was to learn about building a programming language, parsers, compilers, type inference, etc.

Anyways, you can [check it out here](https://talktalk.sh) or on [GitHub](https://github.com/talktalklang/talktalk) but I'm sorta over it at this point because I read another book and it broke my brain (in a good way!) again. Maybe in like six months I'll tell ya about it.

I kept a little diary while I was working on it, maybe it'll be interesting for anyone else making their own programming language. So here it is, unedited. Maybe it makes up for not postin' here.

---

## 7/6/2024

Gonna try to write an AST parser, then generate LLVM IR. Doing the VM + Compiler using C semantics from swift wasn't working great.

## 7/7/2024

Nevermind, we got it working at least. Still like twice as slow as the C implementation.

Right now properties are just string dict keys. Maybe see if storing as an Int hash value is faster? (eventually we want static types but for now `¯\_(ツ)_/¯`)

## 7/8/2024

Let’s add the ability to access the file system. Maybe sockets (TODO: learn what sockets actually are?)

Also collection types lol
## 7/10/2024

Rewriting the parser to an AST so we can maybe add types?

Should add some order of operations tests to make sure Pratt is workin

## 7/19/2024

LLVM JIT workin, what's next?

Workin' on closures. Right now we've just got a heap dictionary in the compiler visitor but names could probably clobber each other which wouldn't be ideal. Is this what name mangling is for?

OK we're starting on an Abstract Binding Tree thing that will be responsible for taking the AST and converting it into a form that's more appropriate for byte code (including with stuff for closures and environments and scopes and whatnot)

# 7.11.2024

Use semicolon for comment token? Could make stripping terminators easier plus discourage ending lines with the,  

## 7/28/2024

Closures workin' on LLVM. We're capturing stuff to the heap on func definition then passing that stuff an argument at call sites.

## 8/1/2024

Bailed on LLVM. I think it'd be more fun to be able to run this on iOS so it's back to bytecode VM. Maybe someday we'll come back to IR-world.

Sorta made a mess trying to fix up closures and simplify stuff. I think the new approach should be to pass closure objects everywhere at def and call times. Later we can look at optimizing those things away.

- [ ] Strings
- [ ] Classes
- [ ] GC?
- [ ] Hoistin'

## 8/7/2024

Closures work.

We've got basic LSP support. Now we gotta work on modules.

- [x] Also the LSP message receiving code needs to actually read the content length because it can get confused sometimes.
- [ ] Completions (still early!) don't work yet with variables defined outside the immediate local scope.

## 8/9/2024

Modules/Structs are getting closer to working.

- [ ] If there's an unknown identifier error, it'd be nice to be able to suggest a module to import if it exists in the module environment.
- [x] It'd be nice to emit synthetic inits

## 8/10/2024

Ok modules and structs work _enough_ (with like a thousand rough edges). Working on the stdlib now. Also sorta using it as an opportunity to try out moving to a new error approach where errors get added to the environment as well as being attached to nodes themselves so we don't just return ErrorSyntax from the analysis anymore. We'll see how we like it.

Made a _whole lot_ of assumptions about things being safe to force unwrap or cast because they made it through parsing that don't hold with the LSP, since programs can be in invalid state. Need to move away from that pattern going forward.

## 8/11/2024

Ok, actually adding arrays now.

- [x] Do a pass over the methods of a struct to make sure their names are available to other methods instead of relying on methods being defined before one another.
- [x] Need to implement while statements in the VM so we can copy stuff over when resizing arrays.
- [x] Comments, let's add 'em.

## 8/12/2024

Realized I need while loops to implement arrays (to resize an array, we need to copy the existing items to the new one).

Also missing the `+=` operator more than I thought I would, might go ahead and add it (or custom operators??) soon.

Also `var` and `let` statements.

- [ ] We've got a lot of `call` functions in the VM right now, there's def some redundancy that we could clean up.

Ok I thought we didn't need "expression statements" because "everything's an expression" seemed p chill. But I think actually what we want is "everything _can_ be an expression". By making everything an expression, the stack sorta just gets cluttered up by values we don't care about. By having expression statements we can pop the values off the stack at the end of the expression. Fiiine.

This is a sorta gnarly move to expression statements. So many places in the tests to add `pop` expectations, so many places where we were checking for exprs where we now need to unwrap the expr stmt to get at the creamy crunchy expr, so many places we assumed things were working great because we were leaving the stack one big mess, gonna have to evaluate uses of `return` to make sure we’re actually leaving stuff on the stack that we should be.

Also this is interesting reading on implementing generics: https://thume.ca/2019/07/14/a-tour-of-metaprogramming-models-for-generics/.

Ok totally revamping how type analysis works. Instead of adding types directly to tree nodes we give them a TypeID that can point to a type, but also be updated more easily as well as be shared with other objects that are known to have the same type. This way we should be able to do more inference? Also not worry about nodes being value types?
#### OK.

Rewrote the type system to wrap ValueType in TypeID which lets stuff get updated easier. Also introduced ExprStmt (see above). Now we're popping off the stack _too much_. Gotta either require explicit returns or adding an explicit return at the end of a thing. Or if there's only one statement make it a return? So many options.

For now I think I just force explicit returns just to get tests green again? Then we make it nice. Also add that `+=` operator everyone keeps talking about.

Ok Actually it might not be too gnarly to just wrap single statements in a return. It's like 11pm so everything seems both easy and impossible so let's try it.

Ok yea it w

_an hour later_

Ok yea it wasn't so bad. Had to special case some ExprStmts (funcs, def expr, structs) (which probably means that they don't want to be ExprStmts anyway) but we're back to the same failing tests we had like 12 hours ago so that's probably a win?

From wren docs: Sometimes, though, a statement doesn’t fit on a single line and jamming a newline in the middle would trip it up. To handle that, Wren has a very simple rule: It ignores a newline following any token that can’t end a statement.


## 8/13/2024

IS TODAY THE DAY? Do we get green again?

Well at some point maybe. But first we're gonna add a quick lil ruby script to make it easier to generate new syntax nodes. Because right now it requires changing like five files and there's no time to do stuff like that. We're so busy.

Ok, the script mostly works. It could be nicer (like including properties to generate) but so could everything here.

Now we're using the script to add more statements and move stuff away from everything being an expr (RIP struct expr, for now). The implicit return stuff seems to be working?

- [x] Need to make sure implicit returns in a `while` or `if` don't bail out of the function when they shouldn't...

Started adding var/let decls because scoping was getting too confusing without those keywords.

- [ ] Need to actually do more validation, like if you try to refer to a var inside a fn that hasn't been defined, we should add an analysis error.

Well well well, looks like someone got sidetracked by LSP stuff. Trying to edit Array.tlk and there's no syntax highlighting. I can't live like this.

--

Ok we're almost green. Actually implementing `Array` now.. imagine that. I'm definitely missing implicit `self` lookups.

omg we did it. Tests are green. Obsidian needs some sort of confetti thing or something.

--

Now what?

- [x] Comments?
- [x] A repl would be nice
- [x] var/let statements

### Where'd we end up?

Super simple arrays work. Comments work. Var statements work. Let statements act just like var statements (for now...). Gonna mess with Strings next I think?

Oh hell yes, CI is green. Good ol' build #28.

Moved init/func decls up into decl() which means we can stop unwrapping ExprStmts in so many places. Phew.

- [ ] One thing we need to do is consolidate errors. I think the overall goal should be to try to get as far as possible and just make sure our types can carry all error or warning info they might need as is appropriate for their stage. Also we should implement panic mode for the parser. Thank you for coming to my ted talk. 

## 8/14/2024

GOOD MORNING what will we break today?

What TODOs do we have?

- [x] A repl would be nice
- [x] Need to actually do more validation, like if you try to refer to a var inside a fn that hasn't been defined, we should add an analysis error.
- [ ] We've got a lot of `call` functions in the VM right now, there's def some redundancy that we could clean up.
- [ ] If there's an unknown identifier error, it'd be nice to be able to suggest a module to import if it exists in the module environment.
- [x] Need to make sure implicit returns in a `while` or `if` don't bail out of the function when they shouldn't...

Started adding var/let decls because scoping was getting too confusing without those keywords.

- [x] Need to actually do more validation, like if you try to refer to a var inside a fn that hasn't been defined, we should add an analysis error.

Well well well, looks like someone got sidetracked by LSP stuff. Trying to edit Array.tlk and there's no syntax highlighting. I can't live like this.

--

Ok we're almost green. Actually implementing `Array` now.. imagine that. I'm definitely missing implicit `self` lookups.

omg we did it. Tests are green. Obsidian needs some sort of confetti thing or something.

--

Now what?

- [x] Comments?
- [x] A repl would be nice
- [x] var/let statements

### Where'd we end up?

Super simple arrays work. Comments work. Var statements work. Let statements act just like var statements (for now...). Gonna mess with Strings next I think?

Oh hell yes, CI is green. Good ol' build #28.

Moved init/func decls up into decl() which means we can stop unwrapping ExprStmts in so many places. Phew.

- [ ] One thing we need to do is consolidate errors. I think the overall goal should be to try to get as far as possible and just make sure our types can carry all error or warning info they might need as is appropriate for their stage. Also we should implement panic mode for the parser. Thank you for coming to my ted talk. x] Strings

09:26 should we rename `none` to `nope`? or why haven't we done that already.

==Ok I think the focus for today is error handling.== We're gonna go all in on analysis nodes having an `analysisErrors` field and really try to stop returning ErrorSyntax nodes.

The amount of extra fields we've got hanging off of AnalyzedSyntax is starting to smell like we just need a new object.

14:37 Ok we have a repl now. I keep putting off doing strings because I feel like I don't know how to implement them in a "machiney" way. I think for now I just go with Swift strings then move on.

I think we could maybe take this opportunity to wrap primitive types in classes and have that be a thing.

17:32 oh baby we have made a mess. Added a String struct and thought "you know what, it'd be nice if string literals just instantiated one of these". Then "you know what, it'd be nice if int literals just instantiated an Int struct". But the analyzer couldn't find the Int struct??? So now we're in it.

Oh no. It's because Int is defined in the standard library. But there are int literals in the standard library. So when the compiler tries to compile the standard library the Int struct isn't defined yet? I think that's what's goin on?

23:00 Ok we punted on fancy string storage. It's just a new Value case that holds a swift string.

Today was bit all over the place (it's ok.). Maybe tomorrow we can buckle down a bit more. Hopefully.

## 8/15/2024

Goooooood morning we have some STOMACH issues today but we're gonna do our best to make some good progress.

11:08 Ok the tum is feeling better. Taking a break from language implementation stuff to see if we can get homebrew installation going.

ok enough of that.

14:20 Just played with neovim crap for a while, I think it's time to add more validation and work more on cleaning up errors.

- [ ] Static members would be nice.
- [ ] So would enums with pattern matching
- [x] Let's start requiring `var`/`let` decls before assigning

---

## 8/16/2024

6:55AM Ok we made good progress on validation and LSP stuff yesterday. Also we are _this_ close to having definition support in the LSP server. I think.

9:02AM We've got go to definition working for properties, methods, types, and vars. Need to figure out if it works for cross file/cross module stuff still but this is a p good step at least.

7:13PM ==Might need to introduce the concept of "mutating"==

9:43PM Ok cleaned some more stuff up, including making the language server understand the stdlib.

- [x] Need to teach the completer about the standard library (probz by moving it to use the module analyzer instead of just looking in the same file like we did for diagnostics)
- [x] Right now you can `let` the same var twice in the same scope. We don't love that.

---

## 8/17/2024

Oh hey. Couple things: trailing block syntax?

Also SwiftUI wrapper?

- [ ] Visibility? (Public/Private/Module?)
- [x] Need to actually establish and test structs passing by value

4:22PM We're cleaning up implicit single statement returns. Previously we were swapping return statements in for expr statements when it was the only statement in a block that allowed it. Now we've added a field to expr statement called `exitBehavior` which I think is a bit tidier.

- [ ] Starting to really want that parser panic mode

10:23PM Ok callin it here. Right now I'm working through why recursive functions aren't working. Or when i can get them to work, the counter example isn't working.

## 8/18/2024

**8:01AM** Oh where are my manners? Good morning! We're picking back up on some locals not being set propertly, either in the VM or in the compiler? We've got a failing test so let's crack on.

Ok we've got things narrowed down to named functions.
### What if we store locals in call frames instead of the stack?

Ok I think it's actually going ok? Everything is actually a bit easier to follow. The one outstanding thing is upvalues.

**1:07PM** Getting closer, I think the only thing remaining is upvalues that live longer than their enclosing scope. We'll need to shuffle those around. Also the `fib` test is failing still....

**3:53PM** Ok ok ok. So I think this whole upvalue thing is actually just not what we need to do. We're not writing this in C, we have Swift. We even have a "fake heap" so if we really wanna fake stuff we can, but on the other hand, if we really want performance we could just actually write it in C or just compile to LLVM. So I think maybe we just bail on this idea altogether.

**10:19PM** Okaaaaaaaaay we made an enormous mess. It was too much. So I backed out of all that and realized I could change [one line](https://github.com/nakajima/talktalk/commit/81f4819b0afc5f7037e65b73c759bc67b987a7a9#diff-1a021f5b5c31a5b7428cb1a9ab38a49fdbf0867614b54415e77dab067b1845a0R318-R342) and the original recursion bug was fixed. Oops. But in all my flailing I also realized i had an off by one error in how arguments were being passed to functions so that's fixed now too.

---

## 8/19/2024

**9:30AM** After some profiling last night, our new VM is slower than our old one. That's ok, we spent a bunch of time optimizing that one. So now let's optimize this one a lil bit. The first step is moving away from the `Chunk` class in the VM towards a `StaticChunk` struct. That should reduce our arc overhead which seemed to be an issue. Before, `fib(35)` was taking like 28-29 sec, just moving to a `StaticChunk` for the main chunk was enough to get it down to 20. Now I'm doing away with nested chunks and trying to put them all at the module level so we can simplify stuff more.

**8:04PM** Brought in the Stack implementation from the old talktalk and now we're doing `fib(35` in comparable time (like 6-7 seconds). We're still like 3-4x slower than c-lox but it's workable.

Tomorrow let's get some features goin. Off the top of my head:

- [ ] Subscripts
- [ ] Hashtables
- [ ] Byte arrays
- [ ] Filesystem support
- [ ] Sockets????????? (We still don't super know what a socket is)

## 8/20/2024

**7:52AM** Subscripts. Let's go. Ok also array literals too. Brackets. Let's go. ~~Let's go brackets.~~

- [ ] String interpolation would be nice.
- [ ] Stop duplicating type info between ValueType.function and AnalyzedFuncExpr
- [ ] We might want a new MethodDecl instead of trying to re-use func logic in structs
- [ ] Let's bring back `Heap.Pointer`

**4:38PM** I think we need some sort of notion of implicit method calls. Like how subscripts call a `get` or `set` method (for now... I don't love that convention). Or maybe how a string might have its `description` field called automatically. Right now there's a lot of manipulation to try to find these methods. It'd be nice to have a more codified system for it.

**11:20PM** Neil tried talktalk. It was great. He wanted `for` loops so badly. Also the LSP server crashed constantly. Also he doesn't know how to use vim but that's not his fault. He was using `let` when he meant `var` and there was no indication that it was wrong. Also wow there is a lot that an LSP can teach. But not when it's crashed. so we need to fix that.

## 8/21/2024

**9:53AM** Took a lil drink last night. Feelin a bit slow today. That's ok. I think the goal for today is pulling all symbol generation into the analysis layer.

- [ ] Visibility? (Public/Private/)
- [x] Add semicolon statement delimiters

## 8/22/2024

**8:46AM** Ok, we've moved symbol generation to the analysis layer (including slot allocation). I think the last thing we're dealing with at this point is that for imported methods, chunk offsets are wrong.

**11:25AM** Ok yea that's what going on. For example, the standard library's Array.append method calls resize() when it needs to resize itself. The location of `resize` is assumed and emitted during standard library compilation, so those locations are wrong in whatever module we've imported into.

At least I think that's what going on. Also not entirely sure what to do about it. Maybe pass around symbols until the _very_ last minute?

**12:43PM** That wasn't it.

**3:09PM** Ok I went back to methods being embedded into Struct objects. All the tests pass. I forget what we were doing.

**10:53PM** Started on Dictionary. Need to beef up generics a lil bit, they're not being specialized properly. The problem is that before, we just assumed a generic type list was a list of strings inside <>. But they can be nested. We've got that parsing but we need to actually make it mean something. Like if you have:

```swift
struct DictionaryEntry<K, V> {
	var key: K
	var value: V
}

struct Dictionary<Key, Value> {
	var storage: Array<DictionaryEntry<Key, Value>>
}
```

Then DictionaryEntry's `K` should be set to `Dictionary.Key` and `V` set to `Dictionary.Value`. Right now that's not working. We'll get it workin' tomorrow.

## 8/23/2024

**8:09AM** Let's get on those generics.

**10:13AM** Oh boy is this the day we solve the duplication between the AST and the ValueType function types?

I think maybe the safest thing to do would be to just start using the AST types more instead of just deleting a bunch of stuff? Then gradually moving stuff over? Maybe?

**11:10AM** Started pulling larger SourceFileAnalyzer visit methods into their own structs. Not sure why I'm telling you about it here but it seems like a good idea.

## 8/24/2024

**7:08AM** I think we need to use our 

**7:26PM** I didn't forget about you. Lol it's been 12 hours. Ok well we're still working on dictionaries. Right now the problem is that our type inference is _too powerful_. When we pass an argument to a parameter that has a placeholder generic type (say, `Array.append`), it's updating that parameter type ID globally, so later when we try to pass it an `int` it blows up. So we need to make these constraints scoped to the call site or instance or something.

**11:27PM** Ok we have a couple issues:

- [ ] Properties aren't being resolved properly. Dictionary sometimes (it's the _sometimes_ that is maddening) doesn't have any properties defined when we're in MemberExprAnalyzer so `self.storage` returns an error. Might be related to...
- [ ] Parameter inference is going too hard. We need to figure out a more local scope or just bail? I think we should maybe take a break and try to implement hindley milner on our own.

## 8/25/2024

**8:10AM** Ok, let's try a new type checker.

**10:14AM** We have created a new package. We have also added a new `id` field to `Syntax` so that we have a more reliable way of identifying fields. That meant that all of our analyzed types needed an `id` field which should have been easy since they all have a wrapped syntax node but the way we were doing that was sorta ad hoc. So we added a `wrapped` field to `AnalyzedSyntax` which points to a concrete syntax type. So we needed to update all the analyzed nodes that had an ad hoc `expr` or `stmt` field to use `wrapped` and point to a generic syntax node. Then we needed to go and update all the call sites where we created those nodes. It was fine.

Time to try to type check some very basic expressions.

## 8/26/2024

**11:05AM** Late start today! That's alright, we're on a good track with this HM stuff (F A M O U S  L A S T  W O R D S). Gonna try to get more of this struct stuff workin'.

**11:37PM** lol 12 hours again. Ok we've got expressions checked and normal structs. But generics are a problem. I think maybe introducing a separate StructType and Instance type was a mistake and I should just make it so that we use Scheme as a StructType sort of thing that can _instantiate_ instances? Need to figure out how property access works in that case but it seems like it might be better than these two concepts that feel too close together? We'll see how it feels in the morning.

## 8/27/2024

**8:49AM** The horrible parrots sing in the sky! 

**9:40AM** We're trying out claude.ai

## 8/28/2024

**8:43AM** We're kinda slacking on the note taking. I think that's because we've been flailing so much more? I don't know. Either way, still working on type inference. We've moved to more of a constraint based setup. Initially we were going willy nilly on generating type variables and trying to unify them everywhere which was kinda going off the rails. Now we're going to try to avoid creating them unless we need to.

It _is_ nice to have the inference visitor just be responsible for generating constraints and picking out lil bits of type info that we know to be true, then letting the solving step happen elsewhere. So that's a win.

## 8/30/2024

**9:47AM** What the heck, we didn't blog at all yesterday! Our readers will be up in arms. Apologies.

Welp we have now run our type checker tests at least once and they were all green. I'm sure there's probably some more bugs lurking but it was nice to see. 

I guess the next step is actually wiring it up to the analyzer.

Deleting a buttload of code from the analyzer, I'm sure it's fine. I think the main things we need to make sure it keeps are:

- [x] Captures
- [x] Pointers?
- [x] Exit behavior
- [ ] Make sure locations are still working for Definitions and the LSP
- [x] Mutability checks for var/let
- [ ] We need to hoist member functions

**9:29PM** Not a bad day's work. We've got source file analyzer tests green again with the new inference system. I think tomorrow it's on to the compiler and maybe LSP. Also probably find a bunch of other broken stuff, but at least it feels like we're on the right track.

Seems like we're probably gonna have to modify the inferencer to allow more stuff in a broken state for the LSP?

## 8/31/2024

**8:22AM** Here we go again! Feels good to run all of the source file analyzer tests and have them pass.

Had to bring back global detection in var/let decls.

**6:56PM** Made a bunch of progress, gettin stuff green today. I think each module should probably be responsible for how it handles the standard library instead of trying to somewhat unify them in the driver. The driver can probably go away tbh, I was just sorta copying Swift and didn't know what I was doing (as opposed to now, when I _definitely_ know what I'm doing.)

**9:35PM** We've made it to the VM tests. Time to debug stack stuff again, oh boy oh boy oh boy.

**11:20PM** We are lookin not bad. Let's call it here.

## 9/1/2024

**8:04AM** wow September. We're goin to vegas today so not sure how much work will get done but for now the plan is to get the analysis error and find definition tests green, then we can go back to banging out head against the stack in the VM tests.

**10:32AM** Ok we're green (kinda!). The factorial (recursion) type inference test is flakey as heck. Probably need to look into that but gonna merge for now.

**1:26PM** TURBULENCE. Ok we've started on an iOS app. Back to our roots. As of like a couple years ago. Realizing we need to probably not just print straight to standard out for print.

I think highlights are borking our selection.

## 9/2/2024

**11:43AM** I didn't forget about you! Yes I did but I'm here now. I'm hacking on the iOS app. I think we just don't worry about Mac interop since all the functionality should be shared by the LSP so on mac you can just use nvim (or VSCode once we make an extension for it).

I think the top two things we need atm for the iOS app are output capture (in the VM) and pretty formatting of talktalk values (in the iOS app).

**2:31PM** Ok we need to handle runaway processes (fib(35) isn't a good look.) Probably memory issues too

## 9/3/2024

**12:26PM** Whoops how are we doing. Still workin' on the iOS app but it looks like our VM has another stack imbalance issue. Whoops.

Could be fun to play with refinements in the type system. Also adding length to array types?

**5:23PM** I think we're gonna need to go through and completely remove fatal errors and forces. First of all because we never want the iOS app to crash, second because we should really just be handling this stuff.

## 9/4/2024

**2:50PM** Just chillin, workin' on showing diagnostics in the iOS app.

**10:15PM** Ok we have started on enums and pattern matching. The basics are in the parser so I think tomorrow we start on the type inference side of things.

## 9/5/2024

**10:07AM** We're doin this and that. Adding enum/pattern matching to the inferencer.

- [x] Also string interpolation would still be nice.

**9:27PM** Ok, we're good (probably!) on inferring match statements. I think tomorrow we get goin on the analysis side. Maybe compiler? Maybe VM? Maybe iOS? Maybe string interpolation?

## 9/6/2024

**8:20AM** ANALYSIS. Let's do it.

**10:25AM** We need to make sure generics work with enums. Fun!

- [x] Need to add exhaustivity checks for matches

## 9/7/2024

Travel day today, and party day. But I think we just wanna keep goin on c

## 9/8/2024

**12:06PM** Another travel day! What the heck?? Ok I had this weird idea but what if patterns are just function calls that return a pattern or something? Or maybe each case body is a chunk? That seems like it could simplify things.

**12:49PM** Gonna take a crack at simplifying the VM before we keep going on enums. That means less stack cleverness.

**9:18PM** Ok, compiler tests are green again. Gotta implement it in the VM now.

## 9/9/2024

**8:51PM** Oh hi. We landed the VM simplification today. Also cleaned up some return stuff (we now distinguish between returnValue and returnVoid in the opcodes and VM). Now we're back on enums. I think we can simplify things a bit by using member getters instead of trying to parse cases out so much. We'll see.

**9:28PM** Need to figure out how pattern matching is gonna work. 

## 9/10/2024

**8:42AM** Patterns! What? Match--- Ok.

**2:25PM** Ok we have added `and` support to the VM tho we should probably get around to adding it to the parser. Besides that... boy... pattern matching... I don't know.

10:01PM Ok, we've got us some pattern matching. The tests are all green?? But we don't have enough tests. Specifically we're missing:

- [x] Nested patterns aren't working yet
- [x] Enums referred to before their definition aren't working yet
- [x] We need to make new frames or something to ensure variables don't leak outside their match statement bodies
- [x] It'd be nice to get some exhaustivity checks going

**10:33PM** Bed time. But we have added some nice failing tests for tomorrow.

## 9/11/2024

**5:20PM** Hey hey. We've got our tests green again, including nested patterns.

**9:58PM** Ok so we're not done yet. We weren't applyingSubstitutions properly on enum cases, which led us to fix that. But then that started leaking enum generic types in their cases. So we've introduced enum case _instance_ to the mix. It works similarly to struct instances in that it can have its own substitutions for generic types and leave the normal EnumCase values alone. So we just have to finish wiring that up I think. It's a unix system, we know this.

## 9/12/2024

**3:48PM** Hey ho. We bailed on enum case instances. I forget how tbh but all the tests are green again. Also we changed how match statements work to just create a new chunk which means we don't leak variables anymore. Huzzah.

**5:48PM** Ok just fixed up variables leaking outside their scope so I think we're in a good spot? Truly impossible to know. But I guess we can merge this PR.

Ok we merged. Maybe we mess around with strings for a bit? Could be nice to have escapes as well as interpolation.

## 9/13/2024

**7:53AM** Time to *oh no it's friday the 13th spoooooky*  keep goin on string interpolation. I think the lexer works, so now we gotta parse these babies.

**10:29PM** Well we got string interpolation in. Now we're workin' on a proper formatter.

## 9/14/2024

**5:35PM** We got a formatter. And it's `.talk` now. I think maybe we play with homebrew now.

## 9/15/2024

**11:24AM** G'day mate! Let's throw another shrimp on the blarney! Homebrew is _sorta_ working, though the account we made on github to do automated releases got flagged lol. 

Maybe it's time to start on protocols in earnest?

## 9/16/2024

**10:01PM** G'night mate! We've been workin' on protocols. You can define them with requirements, we can check that structs implement those conformances, you can pass them around as args. Neat!

We've also added methods to enums. And made a big mess but we cleaned it up. I sort of wonder if the Analysis layer is more trouble than it's worth, now that we have the TypeChecker. I think all it really does at this point are:

- [ ] Symbol generation (do we even need this?)
- [ ] Additional validation (the type checker could do this)
- [ ] Match statement exhaustivity checking
- [ ] It's what the compiler consumes (but it could just consume AST? Or we could have a more deliberate IR?)

So who knows? Might be worth exploring ripping it out at some point. There's so much redundant info between that and the type checker.

**10:23PM** Ok, I think we want to actually have enum instances. Or something. We're struggling Optional right now and I think the other enum generics tests are passing just because it's changing the global type each time. Or maybe it's just storing stuff in other weird spots or something. I dunno let's look tomorrow.

## 9/17/2024

**7:37AM** What were we doing? At some point we should probably write a blog post or something about talktalk. That'll show them, that'll show ALL of them. That probably means we need to add syntax highlighting for HTML for talktalk somewhere. Neat.

But first let's finish this round of protocols. I think that means:

- [x] Making sure enums can conform
- [x] Associated types


Oops, we're starting on for loops in here. I guess that's fine since that's why we started on protocols to begin with but it probably would have been a better idea to do them in a follow up. But also how else are we gonna drive associated types without them? Some sort of FAKE situation? Not on my watch. No let's just make this PR enormous.

**1:38PM** tfw you realize you have to implement _optionals_ for the `next()` method you want to use for _for_ loops that you're implementing to drive out your silly _protocols_ feature.

==Oops found an issue. In CallConstraint.solveStruct, we're calling .member() to see if there's an init defined but it never returns one. If we change it to return something, it breaks SO MUCH STUFF. ==

Wait if we wanna do dictionary iteration do we need tuples?

Why is this feature EXPLODING.

## 9/18/2024

**8:07AM** Ok let's get generic enums fixed up. Then maybe we look at optimizing Symbol? It can just be a string under the hood and all the members could just be methods rather than having enum cases with multiple members.

Hmmm. hmm. Can we actually consolidate struct instance/enum instance? Treat cases as just members on a struct instance? Could be nice. We've got coffee. Let's try.

Wait could we also consolidate all Instantiables? Let's maybe try that next?

**9:15AM** OK we are trying it. The `.instance()` consolidation went like... too smoothly? Everything was just green. Neat. Now we're onto consolidating all instantiatable types, which is a bit more intrusive but I think is gonna be good.

**9:39AM**  Huh we're green there now too. I think `Instantiatable` is maybe not the _best_ name since protocols and enums aren't actually _instantiatable_, but we can cross that bridge later.

Programming is just making one decision on purpose that makes like 5 by accident.

Ok, I think we need to update our pattern handling stuff. I think maybe the pattern visitor is too much?

**10:35PM** Ok bed time. We're reworking a lot of the pattern handling stuff. Fresh eyes tomorrow will probably be helpful but we _are_ keeping the pattern visitor for now. The pattern matching tests are green (heck yea) but the optionals and for loop tests are broken (boohoo). Ohno and the dreaded "infer generic enum" test is failing too.

## 9/19/2024

**7:32AM** Okey doke where were we? Looks like the changes we ended up with last night broke generic handling in patterns. So let's take a look. it's a me, mario.

**12:00PM** Ok I think we've isolated the last problem with generic enums. It's because `solveEnumCase` in the call constraint was ignoring args. Ok so let's just unify the attached types with the args? Not so fast wise guy. That unfortunately unifies them everywhere. We only want it to be on an enum case instance. But we don't have enum case instances. And last time we tried to add them we made a MESS. But maybe this time will be different? It _does_ seem like enum case should be `Instantiatable`. So maybe? At least we can always roll back to `d2322d0` if we get into trouble.

**5:02PM** Ok we got a bunch of stuff cleaned up but we're still failing on using a func call as a match target. I think we might need to move a lot of this stuff in PatternVisitor to constraints.

## 9/20/2024

**8:29AM** Well we go the tests green. Let's see what we missed.

- [ ] ==Lol we need to compile for stmts==
- [ ] Method extensions
- [ ] Property extensions??? 
- [ ] We should make sure `continue` and `break` work.
- [ ] Wildcard patterns
- [ ] Gonna probably need nested conformances? So if `protocol A` conforms to `protocol B`, then `struct S` needs to conform to both A and B in order to conform to A.
- [ ] It'd be nice to cleanup how params/args are handled, including unnamed arguments with the `_` stuff like in Swift.
- [ ] Parser recovery/panic
- [ ] Dictionaries need finishing

SO of course thing to do is embark upon killing the analysis layer.

Maybe the way it can work is that we just visit the AST and save info to the side (like node comments in the formatter) rather than creating new nodes?

Ok so what are all the things the analysis layer does that inference doesn't?

- [ ] Analysis nodes are consumed by the compiler. That wouldn't be too tough to swap out for normal syntax nodes
- [ ] Hold inference type information. We already have that due to the inference layer
- [ ] Generates symbols for the compiler. Could the compiler do this on the fly?
- [ ] Perform exhaustivity checks for match statements
- [ ] Provides LSP location lookup info
- [ ] Coordinates imports
- [ ] Determines exprstmt exit behavior (pop/return/none)
- [ ] Determines captures? Or does the compiler just handle this?
- [ ] Synthesizes inits

What is being duplicated

- [ ] The entire AST tree
- [ ] All of the type information gathered by the type checker

Ok if we're gonna do this we should go test by test in the analysis tests and make sure we have one in either compiler or type checker that tests the same thing.

**11:06AM** Ok after a couple hours of digging into, I don't think it seems worth it for now.

**4:39PM** We had such big aspirations this morning. Well that went out the window which is ok. You're ok. Anyways, we finished up for loops (Probably!) since we forgot to actually analyze/compile them last time. Oops. Of course no compiler/vm feature would be complete without juggling shuffling a bunch of stack logic so we did that too. Tests are green again though!

## 9/22/2024

**3:38PM** We didn't blog yesterday! Holy moly. Well we've got a website. It runs talktalk in webassembly which I think is p neat. 

## 9/23/2024

**7:55AM** Let's try giving funcs their own stacks. Just to see what it's like.

**8:12AM** That didn't feel cozy.

Ok i think we want some IO. I think it might be nice to provide a couple static funcs on an IO object. So we need static funcs. Or do we do method extensions on protocols first? Static funcs i think? Or do we clean up LSP messages for protocol nonconformance? Hmm.

Also still need dicts lol.

**4:03PM** Ok, we've started on static members. Might be time to revisit how methods work. I think we might just want to pass the receiver to the function as the first arg? That'll make it easier to compile for LLVM or whatever.

## 9/26/2024

**9:54AM** ok we took some time off to make some cash money. Where were we? OH yea what's goin on with static funcs?

## 9/27/2024

**6:24PM** Ok we spent some time makin' stuff quicker. Let's tackle some odds and ends:

- [x] Parse +=/-= as its own thing
- [x] && and ||

## 9/28/2024

**8:50AM** Let's get dicts goin.

**12:44PM** I wonder how much slower (if at all) a purely interpreted version of talktalk is.

**10:39PM** We started on dicts, we need to make the LSP better to work on the actual dict algo. We're losing comments in the formatter.

## 9/29/2024

**10:20AM** Ok we're still on dicts. We did a bunch of other stuff. But now to implement the hash table lookup, it'd be nice to have doubles for calculating the capacity load?

- [ ] Looks like for loops aren't working for generic arrays.
- [ ] Multiple string interpolations aren't working?
- [ ] We're not really doing array bounds checks properly

**3:48PM** ohno.  I think we need to redo our type checker (again). We need to move away from any sort of actual inference happening in the visitor and have it _only_ generate constraints.

**9:35PM** we have avoided the impulse to rewrite the type checker (again). We've also introduced a new Pattern syntax type that means our parser can start to do some work for patterns instead of having to overload call expressions. We're starting with `if let` statements but if that feels good we can look at introducing it to match statements.

## 9/30/2024

**10:10AM** OK! I think we need to make constraints scope aware. Oops.

## 10/1/2024

**8:36AM** Wow it's october. Who knew? Ok we are redoing the type checker. Sorta. We're going to be more aware of contexts. So instead of having one big context, full of bad types, we're going to model contexts as a tree that corresponds to scopes. The impetus for this was that we were having trouble with `if let foo` statements, since we only wanted `foo` to be defined in the consequence block but it was leaking.

Hopefully we can be more deliberate about constraints here too.

**1:39PM** Ok brb but right now we're moving .function() to have inference types instead of results.

**8:24PM** ok the problem is that we define the parameter values for a function as fresh type variables. Then we save a scheme that has those free variables. Then we visit the function body but it uses the original variables. So we never unify what we learn from bodies with the parameters (and vice versa).

## 10/2/2024

**9:54PM** We made p solid progress today on the new type checker. We've yet again reached generics. I think the current problem is that we're not respecting explicit type parameters, so for example `struct Inner<InnerWrapped>` getting set with `Inner<Foo>` is ignoring `Foo` and just spawning new type variables.

## 10/5/2024

**6:59AM** FORGOT TO BLOG AGAIN. It's ok we were making good progress. We got a bunch of core stuff goin, now it's time to get into optionals..

**9:53PM** Hey hey. So we got the new type checker green and deleted the stuff from the old. Now we're trying to get the Analysis tests green again. Once that happens I think we can actually go through and strip out a bunch of redundant stuff from analysis. At some point it would be lovely to be able to remove it entirely but in the mean time we can kill stuff like type exprs and whatnot since they aren't needed past type checking.

## 10/6/2024

**8:28AM** Hey chat, we're replacing the InferenceType enum with a protocol so that types can be more than enum cases. It's gonna make pattern matching strictly worse but give us a place to hang source location info and bound types.

## 11/13/2024

We've moved on to something else. But we'll be back

