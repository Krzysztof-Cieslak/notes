# Reactive approach to FSAC design 
Modern IDE (in this context IDE is basically language server - with raise of language server approach to editor tooling, such as LSP, the editor part is just thin UI layer, and most processing is done in server layer) is complex, highly asynchronous system - it needs to react to users actions (such as hover or autocomplete request), perform operations on its own and push changes to the UI, or do things in the background to populate caches or update non-critical parts of UI ( for example getting cross-project errors in error panel is something that may happen more slowly than getting a errors from current file as red marks in the editor)

In the title and in text blow I’m mentioning mostly FSAC but that’s due to the fact I’m familiar with it. But the proposed ideas should work in any editor tooling using FCS.

## Features definitions 
The good IDE may be described with following qualities:

* **Accurate and resilient** - we don’t want IDE report false information (at least for most critical parts such as error reporting). And we don’t want IDE crash if some small part of the subsystem stopped working (especially if the problems are in non-critical parts)
* **Responsive** - IDE must react to users actions in timely manner. Please note this is not really directly to performance but is about perceived by the user performance. This means we could trade performance (as in doing as small amount of actions as quickly as possible) for performance - by duplicating some work or doing additional work in the background to prepare caches, or data that can be used in the future. However what’s important here - this background processing must not impact responsiveness perceived by the users.
* **Elastic**  - the system ( or its critical parts) needs to stay responsive under varying degrees of usage (especially project and solutions size)

The important part thing here is deciding that some of the features are non-critical - they can potentially fail, or be slower - and that’s fine as long as they don’t impact general system **responsiveness** 
 
It turns out that those described above qualities are really similar to the system qualities described by **Reactive Manifesto**. It also provides a solution that can help us to build systems with those qualities- separation of parts of a system into multiple computation units (for example actors) and using message passing for a communication between those separated parts. 

## Current design 
In principle current design of the FSAC is pretty simple - its a single process using FCS to provide rich tooling features. This design is a result of internal FCS design - reactor queue is single-threaded, singleton, FIFO queue, it just processes operations one by one ( supporting limited cancellation capabilities). Such design makes sense from the compiler point of view but it doesn’t allow us to create **elastic** or **responsive** system - our complex, asynchronous system is bottle-necked by this single part of the system. This is especially viewable when we try to provide advanced features working on **solution scope** such as finding all references of symbol across projects in solution, and especially providing cross-project errors. Those features puts lot of **pressure** on reactor queue and us such they lowers responsiveness of the system. For example, my tooltip request won’t get processed until all type checking that’s queued in process of getting cross-project errors is finished. Such behavior shouldn’t be acceptable especially given those operations have different priorities- tooltip requests is something invoked by the user, and he usually wants the result as soon as possible, cross-project errors will be updating error panel, but that’s not something opened by the user all the time or actively invoked by the user. 

### Experimental feature - symbol cache 

Currently FSAC implements one **experimental feature** that tries to workaround this limitation. It’s a background symbol cache - separate process, that can create its own instance of FCS checker, that’s responsible for persisting usages of symbols to local database and for quering this database. In result it allows us to provide responses to requests like find-all-uses-of-symbols with the latency on the 10-100ms level. This has really huge impact on UX, in my personal experience it changes a way you navigate the code base in editor, and it enabled us to implement number-of-references CodeLens known from VS and C#.

Symbol Cache (SC) is started by main FSAC process just after start. The processes communicate with each other with HTTP protocol.
First message is sent from FSAC to SC when FSAC finished cracking projects - the message contains serialized FSharpProjectOptions for all projects in workspace. After getting it SC starts typechecking all the projects in current workspace and after typechecking it gets all the usages of the all symbols in each project. This data is saved to local database.
Next step is updating database - this happens whenever FSAC typechecked a file - it gets all usages of all symbols for just typechecked file and it sends this data to SC that’s responsible fo updating database.
When FSAC gets from user get-usages-of-symbol requests it calls SC, SC queries DB and return results back to FSAC which pushes result to user.

This design has **negative impact** in terms of the performance - some operations are duplicated in both processes ( initial type checking of the projects is probably done, at some point of the time, on both sides), it probably slightly increases memory usage, and CPU usage. However it increases general **responsiveness** of this particular feature achieving levels of latency we could never achieve with the classic approach.

## Proposal - even more processes
Using a above experiment as a test case, and noticing it follows Reactive Manifesto ideas, I’d like to propose introducing **more processes** to the FSAC system that will encapsulate the parts of the system and use own instance of FCS to perform some operations. The obvious candidate for such encapsulation is **cross-project error checking** - it’s non critical feature, with huge impact on FSAC responsiveness (due to the amount of typechecking operations it puts on reactor queue). It’s not a critical feature- it’s not invoked directly by the users, it updates only the error panel, users are able to understand why it takes longer (while no one understands why tooltips are not instant). At the same time it’s valuable feature to have - I hate when my editor tells me that everything is fine, I build an project and then it fails due to the error in other project in my solution caused by the fact I’ve changed name of one function.

### Drawbacks

There are several drawbacks of such design:

* It increases general **complexity** of the system - we have more moving parts, more things can go wrong.
* It increases difficulty of **debugging** of the system - totally separate processes means that we need to debug them separately which may be problematic 
* It may impact **memory usage and CPU pressure** - especially in case of something like moving cross-project errors to its own process we would need to be really carefully about the memory and CPU pressure caused by typechecking of the projects in workspace. We would make sure that we use right FCS settings, we handle cancelation properly and we don’t typecheck projects  we don’t need to .

### Alternatives

There are 2 main alternatives I can think of right now:

* **Disable cross-project error checking** in Ionide by default and work on general compiler performance. I came to the conclusion that in its current shape the features is not worth the problems it causes so we would change default settings and work on making compiler itself fast enough to make this work well in the future (however, it’s bit hard to believe that we can make so huge improvement in compiler performance)
* Use **priority queue** for the reactor queue - it’s one of the ideas I’ve discussed with Don some time ago, and i think it would solve some of the issues - we could prioritize actions invoked by the users and use lower priority for non-critical background things. I feel this could have really positive impact on system responsiveness, but I’m not sure if to the same degree as complete separation of some operations. Also, I could see this being implemented together with my proposal to achieve really good results.
* Follow path suggested by Reactive Manifesto even further and create something looking like actor model. For example, we could create **FSharpChecker instance per project file** (each in separate process) and have supervisor that’s responsible for querying those actors and controlling their lifecycle 
