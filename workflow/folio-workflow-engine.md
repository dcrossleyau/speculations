# White paper: the FOLIO workflow engine

<!-- md2toc -l 2 folio-workflow-engine.md -->
* [Preface: what this document is](#preface-what-this-document-is)
* [Background](#background)
    * [Terminology](#terminology)
* [Review of existing workflow systems](#review-of-existing-workflow-systems)
* [Example workflows](#example-workflows)
    * [Scenario 1: acquisition of a requested book](#scenario-1-acquisition-of-a-requested-book)
    * [Scenario 2: unboxing a delivery](#scenario-2-unboxing-a-delivery)
    * [Scenario 3: submitting a dissertation](#scenario-3-submitting-a-dissertation)
* [Use of data in workflows](#use-of-data-in-workflows)
    * [Implicit or explicit data-passing?](#implicit-or-explicit-data-passing)
    * [Pass by value or by reference?](#pass-by-value-or-by-reference)
    * [Notion of object type](#notion-of-object-type)
    * [Workflow state vs. object state](#workflow-state-vs-object-state)
    * [Data standardisation](#data-standardisation)
* [Implementation analogies](#implementation-analogies)
    * [Finite state machine with transitions](#finite-state-machine-with-transitions)
    * [Makefile-like dependency tree](#makefile-like-dependency-tree)
    * [Connector Framework](#connector-framework)
    * [Very high-level DSL](#very-high-level-dsl)
* [Implementation strategy](#implementation-strategy)
    * [Front-end/back-end interaction](#front-endback-end-interaction)
    * [Virtual Machine for running workflows](#virtual-machine-for-running-workflows)
    * [Expressing workflows](#expressing-workflows)
        * [XML or JSON](#xml-or-json)
        * [Human-readable/writeable DSL](#human-readablewriteable-dsl)
        * [Compiled form](#compiled-form)
        * [Visual representation](#visual-representation)
        * [Discussion](#discussion)
    * [Data types](#data-types)
    * [Operations](#operations)
        * [Okapi calls](#okapi-calls)
        * [Record manipulation](#record-manipulation)
        * [Control flow](#control-flow)
        * [Subroutines](#subroutines)
        * [Human interaction](#human-interaction)
        * [Other](#other)
    * [Tracking jobs](#tracking-jobs)
    * [Error handling](#error-handling)
* [Appendix: using the notification system](#appendix-using-the-notification-system)



## Preface: what this document is

This is an overview of what workflow is or could be in FOLIO, how it breaks down into v1 and v2 features, what implications it has for the broader FOLIO architecture, and how we might go about implementing the v1 parts of it without circumscribing our options for the v2 parts when the time comes.

Much of what is contained herein was first discussed in the Workflow breakout group on the morning of Friday 20 September 2017, in the Montreal FOLIO developer's meeting. Contributors in that session included 
D Ellen Bonner,
Ian Ibbotson,
Heikki Levanto,
Peter Murray,
Nassib Nasar,
Jason Skomorowski,
and
Mike Taylor.

At this stage, the contents of this document should be considered as a foundation for discussion, rather than as a design. The document's purpose at this stage is not to guide implementation, but to provide a more concrete vision of what workflow is or could be, so that future discussions have a more secure footing and do not drift away into abstractions.



## Background

The term "workflow" in FOLIO has been used in several different ways and has accumulated a lot of connotations. Among other things, it is a wildly popular future feature, which has been used to great effect by the marketing team. But it has yet to be clearly defined what it consists of, and seems to have been interpreted in different ways by different groups.

What mean here by "workflow" is that facility of a FOLIO system to guide human users through sequences of operations, filling in as much of the work as it can. This encompasses cases where _all_ parts of a workflow can be done by the computer without human intervention: in this case we refer to the process as "automation"; but automation in this sense is not a conceptually separate thing from workflow, it is simply workflow in which none of the steps require human intervention.

Workflow in FOLIO is more important than in other library systems, because FOLIO is composed of numerous small applications which interact. Workflow (including automation) will be the glue that binds them together into a powerful, flexible, configurable whole. Rather than a single monolithic Acquisitions application with tightly bound procedures built into it, we envisage smaller, substitutable applications such as Ordering, Invoicing and Unboxing, each of them operated primarily in terms of higher-level procedures defined as workflows.

Although "workflow" has been thought of as a FOLIO v2 feature, we need to think about this now, because the term has been used in two quite separate senses. The v2 feature is the visual Workflow Editor that has been [prototyped by Filip Jakobsen](http://ux.folio.org/prototype/en/workflows?view=full) and which will allow librarians to set up their own workflows. (For more on a UX-oriented perspective: see [the "Workflow app" issue in the UX Jira project](https://issues.folio.org/browse/UX-52).)

But well before this is required, FOLIO will need the underlying mechanisms for executing workflows. Until we have the visual editor, we can make do with hand-authored workflows: creating such workflows will be part of the job of developing high-level applications.

We need to think about this now, rather then deferring until after the v1 release, because otherwise we will need to waste a lot of re-engineering down the line: replacing what in effect will be hardwired workflows with applications of the Workflow Engine. Want to avoid wasting development effort on hardwired workflows that we are only going to throw away.


### Terminology

* A _workflow_ is a set of instructions for FOLIO and/or human users, expressing how to carry out a task. In v1, we will author these by hand, and in v2 there will be an editor. Either way, except when being edited, these are static, like classes in objct-oriented programs.

* A _workflow instance_ is a specific running instance of a workflow. It has parameters that may be set when it's started (e.g. the title of a book to order) and its own state that changes as the task progresseds (e.g. the status of an ongoing order). It resembles and instance of class in an objct-oriented program.

* A _job_ is a simpler term for a workflow instance.

* The _Workflow Editor_ is the visual editor for creating workflows, which will be created as part of FOLIO v2.

* The _Workflow Engine_ is the software that executes workflows on behalf of a user. It will exist quite separately from the Workflow Editor, which is only one way of authoring a workflow.

* A _workflow language_ is a textual representation of a workflow, which can be authored by a developer. See [below](#expressing-workflows). Since this will be a domain-specific language (DSL) this may also be referred to as a _workflow DSL_.



## Review of existing workflow systems

Nassib has volunteered to do this: see [INTFOLIO-6](https://jira.indexdata.com/browse/INTFOLIO-6).



## Example workflows

To ensure that FOLIO's workflow capabilities are sufficiently expressive to execute real library workflows, we need to establish some concrete scenarios. We will then be able to express these in a workflow language and consider how a Workflow Engine could execute them.

Here are three scenarios.


### Scenario 1: acquisition of a requested book

The following rather unlikely sequence of actions involved in placing a purchase order illustrates some of the possible difficulties that can be encountered in the course of this seemingly simple task.

* A patron wants to read a certain book -- an edition of _Pride and Prejudice_ with an interesting introductory essay, say -- but it is not in the library. She submits a request.

* A paraprofessional (see below) picks up the request and finds a source for a hardback edtion of the book. He submits it as a purchase request.

* An acquisitions librarian picks up the purchase request and notices that the suggested edition is expensive. She pushes that purchase request back to the patron, asking whether there is any reason that an inexpensive paperback edition will not suffice.

* The patron responds that only the nominated edition is acceptable since her  interest is in the introductory essay.

* The acquisitions librarian accepts this, but is not authorised to spend the high price of this edition, to she escalates the purchase request to a senior librarian.

* The senior librarian receives the request, along with the explanation for why the expensive edition of the book is necessary, and OKs the purchase.

At this stage, a purchase order is raised, and the present job is complete. A new job may be created to handle the financial side of the transaction, but this will likely involve different people.

("Paraprofessional" is an awful term used for people that do library jobs without a library degree. Seriously.)


### Scenario 2: unboxing a delivery

When a package of books arrives at a library, a file is also supplied that contains some relevant metadata. This may be physically included in the package, or downloaded from a specified FTP location, or obtained by some other method. Ingesting the file is the beginning of the process of getting the books into the system and onto the shelves.

* Librarian imports the file of bibliographic data -- or, better, specifies a location, and FOLIO fetches and imports it.instance-level records.

* Typically this file will consist only of instance-level bibliographic records. For each such record:

  * If the system does not already have a record with this ISBN, insert the record into the instance store.
  * Otherwise, merge the data in the new record with that in the existing record. This process will potentially be complex, involving heuristics and perhaps human intervention. (This might in fact be a whole other workflow, to be called as a subroutine.)

  * The books typically come shelf-ready with barcodes already attached. Somewhere in the bib records will be a list of the barcodes for each physical item -- e.g. for all six copies of a given book. For each barcode:

    * An item-level record must be created.
    * Each item record must be populated with certain fields copied across from the instance record that it pertains to.

  * Shelving locations (buildings/branches and shelf ranges) will typically be determined by the call number (or shelf mark -- essentially different phrases for the same thing).  Call numbers specify the sort order at a local library, but the call numbers themselves are usually determined by a national library that is looking across the whole of published literature.  The first part of the call number specifies the broad subject area.  (For instance, Dewey Decimal _560s_ and Library of Congress _QE_ -- both representing the subject of geology and dinosaurs.)  The shelving location is based on that first part of the call number; there would be rules for shelving items in one location or another. Locations would need to be derived and set into each item record.

  * Check for holds on the newly added instance. If they exist, notify the patrons that the book is now available. Note that holds may be on either instances or (less often) individual items. Typically the patron doesn't care which item they get, but sometimes they do.


### Scenario 3: submitting a dissertation

Workflows for dissertation submission vary widely between libraries. Here is one possible flow.

* A student submits her dissertation.

* The submitted dissertation is added to the institutional repository (IR) with status "not yet accepted".

* Examiners evaluate the dissertation. Assuming it is adequate, they arrange and evaluate the student's defence.

* Depending on their verdict, the status of the dissertation in the IR is updated to "accepted", "rejected" or "revisions required".

* If revisions were required, the student submits a revised dissertation.

* The revised version is added to the IR with status "not yet accepted". It now becomes the version that is discovered by default in the IR, but retains history so that the original submitted version can still be obtained.

* The examiners evaluate the revisions. Assuming they are acceptable, the status of the revised manuscript in the IR is updated to "accepted".

* Once the dissertation is accepted (whether with or without revisions), the cataloguing librarian writes an abstract and it is added to the library catalogue.

Note that this example extends FOLIO workflow's influence outside the library to the institutional repository and even potentially to the disseratation examination process.



## Use of data in workflows

It will be apparent from the scenarios above that many kinds of data are going to be involved in workflows:
instances,
items,
dissertations,
locations,
holds,
purchase requests,
purchase orders
and more. Each of these kinds of object will presumably be represented by its own kind of record within FOLIO, defined by its own schema and stored by its own back-end module. For many of them there will also be a front-end module allowing users to manage them.

This raises several issues.


### Implicit or explicit data-passing?

Do we want to implicitly pass anonymous records from one step in a workflow to the next, analogously to how Unix pipes data between processes? Or do we need workflows to explicitly name objects and refer to them by name subsequently?

First approach:
```
book < getPatronRequest | findSource | obtainBook
```
Second approach:
```js
request = getPatronRequest(book)
vendor = findSource(request)
obtainBookFrom(book, vendor)
```

The former is terser and more elegant when it works, but the latter is more explicit and expressive. It's nice to imagine that we could support the simpler style when it's sufficiently expressive, but also the more explicit style when it's needed.

Is there precendent in existing programming languages for suppoting both styles? The shell itself offers something along these lines, but it is rather inelegant. The second approach would look like:
```
getPatronRequest `cat book` > request
cat request | findSource > vendor
cat vendor | obtainBookFrom `cat book`
```
I would hope we could improve on that.

It will be possible to think about this in more detail once we start to sketch the representation of workflows (see [below](#expressing-workflows)).


### Pass by value or by reference?

Should the objects are are inputs to and outputs from workflow steps be whole objects, or merely references to them?

In the former approach, we would be passing around, for example, item records with `id`, `title`, `barcode` and `status` fields; and workflow steps would make changes to the in-memory records and pass the changed versions on.

In the latter approach, we would be passing only the ID that points to the relevant item in the data store; and workflow steps would need to fetch the record in order to see data fields, and to write back modifications.

The latter approach is less efficient, in that most steps will need to carry out several HTTP requests. That may not be a fatal objection, though: more important is getting the conceptual model right.


### Notion of object type

Whether we use pass-by-value or pass-by-reference, we will need an explicit notion of object type. If we pass only IDs, we will need to know the type of each object so we can look it up in the appropriate back-end service when we need data from it; and even if we pass whole objects, we will still need to know the type so that we can determine what service to use to persist changes.

For the purposes of this document, we will refer to objects with a simple [URN](https://en.wikipedia.org/wiki/Uniform_Resource_Name)-like scheme where type-specific IDs look like `item:234` or `instance:543`. But that should not be taken to indicate a preference for passing by reference: it's just a notation adopted for convenience.

Since we will need a notion of type, we may well wish to make it more explicit and visible within FOLIO. For example, we could introduce a type registry similar to those in operating systems like Mac OS, allowing users to register their preference of which application to use when viewing objects of a given type: "When I get an object of type `user` I want to use the `@folio/users` app to open it" (i.e. the standard Users application) but "When I get an object of type `item` I want to use Frontside's `@fontside/pretty-users` instead".


### Workflow state vs. object state

Does a workflow instance have its own state separate from the state of the objects it deals with? That is, does it suffice to mark a purchase as "needs authorization", or do we also need to mark the workflow instance that generated this purcahse as "awaiting purchase authorization"?

Suppose the progress of a job is halted on the need for a purchase order to be okayed by someone with sufficient authority. We do not need to assign the responsiblity for doing this to an individual: there may be a whole group of people who can OK the purchase. So it's tempting to imagine we could place some kind of trigger on the purchase record itself to restart the workflow, and so avoid having the workflow instance itself track where it is in its sequence of operations. But making all the other kinds of task workflow-aware may spread the responsibility thinner than we want. Lots more thought required here.

If the workflow instance has its own state, then we can tie together related operations after the event. Workflow can be a useful source of audit capability, perhaps saving individual services from having to do excessive audit logging.


### Data standardisation

Our experience of information systems is that data-cleaning is invariably a more extensive and costly task than anticipated. Experience of workflow systems in scientific contexts corroborates this impressions: large sections of scientific workflows tend to be concerned with transforming and massaging data -- both on input, to make it amenable to processing, and on output so it is usable to the destination system.

Happily, much of this difficulty should be avoidable in the context of FOLIO, as we will generally be dealing with data that has already been canonicalised on entering the system. So data normalisation should not be something that most workflows need to worry about.

However, importing data from other systems may well be one of the important use-cases for the workflow mechanisms. It is useful that workflow mechanisms provide us with a means to script and customise import.

Since we aim to import data without human intervention, this process would usually be classified as automation; but these processes will be able, if necessary, to delegate to a human to fix problems encountered along the way.



## Implementation analogies

To help guide our thinking as we begin to consider implementation, it's useful to consider some analogies -- none of them perfect, but each potentially shedding some light on the problem we need to solve and how it is best modelled.


### Finite state machine with transitions

A workflow can be seen as defining a state machine with a start state, a desired end state, and a set of transitions between states which can result in arriving at the end state.

Considering it in these terms may help us to think more clearly about what kind of state needs to be represented and where it is best maintained. Most fundamental here is the question of how much state we want to centralise within the workflow instance itself, and how much can reside in the affected objects (items, instances, etc).

In practical terms, the entire FOLIO database contains state (as all databases do). This is far too much state to model coherently, and we will be interested in the transitions between possible states of only a small part of this.


### Makefile-like dependency tree

In many cases, a workflow will require that some set of two or more tasks are completed before it can proceed to the next step, but the order in which those tasks are completed does not matter: for example, it might require one person to validate the choice of a book to buy while another person clears funds for the purchase:
```
        Request received
             /    \
            /      \
           /        \
          v          v
Request cleared    Funds acquired
           \        /
            \      /
             \    /
              v  v
           Place order
```

The combination of these requirements cannot be satisfactorily modelled as a sequence of steps, since we do not wish to make one person wait until the other has finishes his part of the job. Such situations may be better expressed by considering each task as separate, and specifying a directed acyclic graph of their dependencies. In the Unix `make` program, it would be specified something like this:
```
**target**: placeOrder

placeOrder: requestCleared fundsAcquired
	doPlaceOrderActions

requestCleared: requestReceived
	doRequestClearedActions

fundsAcquired: requestReceived
	doFundsAcquiredActions
```

(In practice, most implementations of `make` run tasks one at a time, but some implementatiins really do run parallel tasks simulaneously on multi-processor machines.)

If we go some way towards adopting this model, we will face the challenge of integrating `make`-like dependency-graph syntax with the more conventional programming-language syntax used in whatever DSL we come up with.


### Connector Framework

The FOLIO workflow system has several points in common with the
Connector Framework (CF), and experience gained on that project may be valuable here.

* Workflows, like connectors, are composed of functional steps that perform high-level, domain-specific operations. ("Navigate to URL", "Fill form field", "Parse HTML", etc. in the Connector Framework; "Create item record", "Obtain human confirmation", "Place purchase order", etc. in FOLIO workflow).

* Workflows, like connectors, will need a persistent representation -- possibly several (see [below](#expressing-workflows)).

* FOLIO workflow, like the Connector Framework, will need a powerful and general "Transform" step to handle data-cleaning.

* The separation between CF Engine and the CF Builder mirrors that between the Workflow Engine (which we probably need for v1) and the Workflow Editor (which is not for v1).


### Very high-level DSL

Another way to think of workflows is as programs written in a very high-level domain-specific language (DSL). In this language, fundamental operations are not on integers and strings, but on high-level objects such as items, loans and invoices. Many operations will be implemented in whole or in part by CRUD reqeusts against Okpai.

In these respects, such a DSL will resemble commands in the forthcoming Okapi CLI ([OKAPI-425](https://issues.folio.org/browse/OKAPI-425)).

Howeverm a workflow DSL will require control-flow primitives -- and, most likely, domain-specific primitives such as the ability to iterate over all books in a delivery, or all items related to an intance. See [below](#control-flow).



## Implementation strategy

With these analogies in mind, and aware of the three workflow scenarios outlined above, we can now begin to sketch what an implementation might look look. We will look first at the interaction between front-end and back-end components; then consider the need for a virtual machine that can run workflows, before considering different representations -- for humans and computers -- of workflows. We'll then consider some of the operations that we will need to be able to represent and execute, and finally work through what will be required the track the state of an in-progress workflow, and how to handle errors.


### Front-end/back-end interaction

Leaving aside the v2 Workflow Editor, which is outside the scope of this document, we expect that most or all of the workflow implementation will be on the back-end. In fact, this is necessary for it to function in automation: we cannot have background tasks stop executing when the user's browser is closed. So while workflows will often be invoked from the front-end, the effects must run mostly on the back-end. (Some workflows will also be started from the back-end: for example, scheduled jobs.)

In some cases, front-end involvement will of course be necessary: for example, when a librarian is required to OK a purchase. Much of the design work in the Workflow system pertains to how this can be done. The most promising approach seems to be using the FOLIO notification system to inform individuals when a task await their input. Then we can make it possible for individuals to configure how that system treats various kinds of notifications: for example, low-priority tasks may simply be queued within FOLIO, to be picked up when the user finds it convenient; higher-priority tasks can be emailed out to solicit a more rapid response; and top-priority tasks might even interrupt a user who is working elsewhere within FOLIO. (As always, our goal here is to provide mechanism, not dictate policy.)

In other cases, front-end/back-end interaction will be simpler and more transitory. Consider for example a workflow to create a new item record based on a specific instance. The workflow will embody the knowledge of which instance fields to copy across to the new item record, and how to modify those fields, but the result will likely not be the immediate creation of a new item record, but rather transitioning the UI to a "create new item" page with the form already largely filled in. In this scenario, we expect that no new record is persisted within FOLIO until the user hits the Save button.


### Virtual Machine for running workflows

At the heart of the Workflow Engine will be a virtual machine that can interpret instructions to execute the high-level actions that constutite a job. It will need the ability to load the instructions that make up a workflow using some persistent format (see the next section), and to move through the steps while maintaining an approrpriate notion of the workflow's state, which must be able to persist across long jobs that involve human intervention.

We have at least two candidate approaches for how to implement the Workflow VM. These re:

* Interpreting each operation and executing it directly from an engine.

* Compiling workflow to some existing language and running it within the that language's execution environment, in the presence of a runtime system that provides the necessary APIs for actually executing the steps. For example, we might compile to JVM instructions (or to a JVM language such as Groovy or Java itelf) and run inside a Java Virtual Machine; or compile to JavaScript, and run under the Node interpreter.

While the second approach is superficially appealing in that is seems to give us less work to do, that is largely an illusion: providing the runtime support will be the great bulk of the work. In comparison, creating an interpreter loop that can execute control-flow primitives as well as FOLIO operations will be only a little more work, but give us much more flexibility -- especially in dealing with the thorny difficulty of blocking on steps that require human action that may not be completed for days or weeks.

At least initially, the control gained by executing workflows inside
our own VM outweighs any possible performace gain from compiling to a
language with an optimised execution environment. And this is likley
to remain true, as the overhead of primitives like looping and
branching is going to be absolutely swamped by the much more expensive
operations being executed: Okapi requests and suchlike.


### Expressing workflows

Surprisingly, there could conceivably be as many as four different
representations of workflows.

#### XML or JSON

We will likely (but not necessarily -- see [discussion](#discussion) below) have a representation of workflows that is simple for machines to parse, and which is stored in the FOLIO database as the canonical representation of workflows. This format is analogous to the XML representation of connectors in the CF, although we might conceivably pick a different metaformat in 2017: JSON is a contender.

#### Human-readable/writeable DSL

XML and JSON are difficult for humans to write: consider, for example, a typical CF connector such [the PLOS ONE connector](http://cfrepo.indexdata.com/repo.pl/allcon/PLoS_ONE.10.cf). This is for a relatively well-behaved site, yet it comes to more than 700 lines of XML, much of it difficult to follow even for those already familiar with CF concepts.

This is not a problem in the CF, because we have the CF Builder to create these for us. However, the analogous Workflow Editor is a very substantial piece of work, which we will not even begin to work on until after the v1 release. Until then, we will need a reasonable way of authoring workflows. A high-level domain-specific language is the obvious way to do this: it will not enable regular librarians to author workflows, but will make it easy for developers to do so.

It's likely that we will find ourselves sketching workflows in a high-level pseudocode in any case. So going ahead and creating an interpreter that can run those workflows directly (or compile them into a form that can be run) seems like an obvious step.

#### Compiled form

When a workflow is loaded into the engine -- whether from an XML/JSON format file, or a DSL -- the result will be an in-memory representation, most likely a tree, that can be directly executed. We may find it useful to eliminate the overhead of parsing and compiling by storing the compiled version in a binary format that can be loaded very quickly -- analogous to Java's `.class` files. Or perhaps the potential efficiency gain will be little enough that we benefit more from simplicity of just compiling every time and running directly from the internal representation, as Perl, Python and JavaScript do.

#### Visual representation

The fourth and final form of workflows is the visual representation that will appear in the v2 Workflow Editor. This will need to parse stored workflows (from XML/JSON or a DSL) to determine what to draw on the page; and it will need to render edited drawings back into a form to send back to the server.

#### Discussion

Four representations of workflows certainly feels like too many. For the moment. we can simplify matters by ignoring the visual representation, which certainly will not be in v1, but certainly will need to appear in a subsequent version. Of the other three representations, how many do we need?

We can certainly do without the compiled form, at least in early versions of the workflow implementation. This is purely an optimisation, to reduce parsing overhead, and can be ignored until and unless profiling shows that it is needed.

But that still leaves is with two forms: XML/JSON and the DSL. In one conception, the latter exists as a human-maintainable form to compiler into the former. But in that case, do we even need the XML/JSON version? Instead, the DSL could itself be the canonical form: workflows will be written by developers, using the DSL; they will be stored in the FOLIO database in that form; then they compiled and run directly from the parse tree by the Workflow Engine. Later on after v1, these workflows will be compiled and translated into visual form by the Workflow Editor, which will then emit updated workflows in the same DSL, and write them back to the database.

So it seems possible that we have no pressing need for an XML/JSON form.

What language should we write the DSL parser in? (Or, if we decide to make an XML/JSON representation canonical, the parser that translates raw parse trees into an executable form?) In a future version of FOLIO, we will need to parse and render this language on the client side for the Workflow Editor, so we will need an implementation of the parser/renderer in JavaScript. That suggests we should create those JaveScript implementations for version 1, using them on the server side initially, and avoid later having two separate implementation in different languages. This may mean that the workflow compiler becomes the first Okapi module written in JavaScript; or it may entail somehow calling out from an RMB-based module into a separate JavaScript compiler/renderer.


### Data types

The workflow VM will need to represent, and provide operations on, several data-types. These include the obvious primitives:
* `integer`
* `boolean`
* `string`

And of course structured types:
* `array`
* `hash` (a mapping of key-names to values)

But also possibly domain-specific types:
* `item`
* `instance`
* `loan`

It is an open question whether we need to reify domain-dependent concepts such as items and instances, or whether it will suffice to blindly operate on named fields. If we do add type-specific operatoins, where would they be defined? If the Engine is made schema-aware, and is able to read the necessary JSON Schema files (where from?), what additional capabilities would that facilitate?


### Operations

We now consider some of the operations that will need to be supported by the Workflow Engine. These will most likely be represented by nodes in a parse tree or equivalent.

#### Okapi calls

Most fundamentally, we will need operations for CRUDding various kinds of objects: items, instances, loans, etc. A workflow for renewing a loan will consist primarily of fetching the loan (HTTP GET), modifying its due-date, and rewriting it (HTTP PUT). So we will want a very efficient, readable way of specifying such operations. Perhaps something like:
```js
record = fetchItem(id)
// Make whatever changes we wish
storeItem(id, record)
// Oh, we changed our mind
deleteItem(id)
```
Or maybe something more more explicit along the lines of
```js
record = fetch(`/items/${id}`)
// Make whatever changes we wish
store(`/items/${id}`, record)
// Oh, we changed our mind
delete(`/items/${id}`)
```
(Or of course could implement the former in terms of the latter)

#### Record manipulation

Manipulation of records will be the other major part of most workflows. These changes, typically made to a single field at a time, fall into several categories:

* Adding a field, or appending a new subfield to an existing field.
* Replacing the value of an existing field.
* Deleting a field or a subfield.

It seems clear that the model will need to recognise subfields from the outset, and that we may need to handle both structured and list fields (analogous to JSON objects and arrays).

Transformations of fields will need to be supported: at minimum, the ability to copy one field, transformed by regular-expression substitution, either to another field or back to itself. For example, we may need a workflow step to normalise names from "Brian W. Kerninghan" form to "Kerninghan, Brian W." form:
```
assign author = author: /(.*) (.*)/ to "\2, \1"
```

Or to reset a loan date while remembering the old one:
```
append oldReturnDates, returnDate
assign returnDate = $1
```

#### Control flow

Like any sufficiently expressive language, the Workflow language will need primitives to control the flow of execution.

In the Connector Framework, we tried initially to sidestep this requirement by providing the "simpler" alternative of alternative steps; this proved insufficiently expressive, and the Skip-If step had to be introduced, leaving us with an unsatisfactory situation that has neither the simplicity we desired or the expressiveness we might prefer. For this new VM, it's better to face to problem of control-flow head on, and design it in from the start.

We will of course need the standard set of primitive:
* `if` / `then` / `else`
* `while`
* Maybe `do` ... `while`

And perhaps some domain-specific iterators:
* `foreach` item related to an instance
* `foreach` loan held by a patron
* `foreach` patron in a group

It's not yet clear how these would best be expressed, and whether they would need the schema-aware type system hinted at [above](#data-types).

More importantly, we will need control flow for parallel jobs. Consider the [earlier example](#makefile-like-dependency-tree) of a request to buy a book, which must be cleared by a librarian and the funds acquired by a purchaser. Since both these tasks require human intervention, they will typically take hours, minutes or even days, so parallelising is indispensible to avoid long delays.

How might such a control-flow be most cleanly expressed? Perhaps something like a `doall` keyword which governs a comma-separated list of statements?
```js
doall
  authorizeChoice(),
  clearFunds()
```

Or, with braces for clarity:
```js
doall {
  authorizeChoice(),
}, {
  clearFunds()
}
```

#### Subroutines

In many cases, one workflow will need to invoke another. This is equivalent to a subroutine call. In fact, we can think of individual workflows as simply being subroutines -- and the ones that we invoke from the UI as being like main functions.

Note that this requirement makes explicit the notion that workflows must take arguments. When invoking workflows from the UI -- as, for example, a "make new item for this instance" workflow -- some of the parameters will be determined by the context: for example, "this instance" would be the be that the user is viewing when invoking the workflow. In other contexts, some or all of the arguments will need to be explicitly specified.

#### Human interaction

XXX Hardest part

#### Other

XXX What else?


### Tracking jobs

XXX A back-end module for CRUDding the status of jobs based on workflows. Will likely consist of a tree of pointers to job-step objects, each with its own state. Will not need to be transmitted, so no need for a serialised or human-readable form.


### Error handling

XXX If a step fails, do we fail the whole job? We can report back to the person who initiated the job.

XXX In the unboxing example: "here’s a report of the three things we weren’t able to do anything with."

XXX The workflow may have multiple operations on the go at once (e.g. "when these three parallel tasks are complete, go on to Step X). How do we handle recovert if one of them fails after others have already succeeded?

XXX Heikki writes: There can be various levels of errors, some failing the whole job, some reporting back the three items that failed, etc. It all just needs to be configurable, maybe even on the level that a job could have an error list, and every step could have some checkboxes on how to handle things going wrong. Bit like we did in CF.



## Appendix: using the notification system

XXX We will want to do all notification using the notification system: "please authorize this", "your request has been accepted/rejected", "your job failed", etc. Users will want different kinds of message communicated in different ways, e.g. some by email, some by text-message, some waiting on the system until the next login. So notifications need a "type" drawn from a small controlled vocabulary (as well as other extensions such as sender, date, subject)



&nbsp;

> **XXX NOTE TO SELF**: get David to proof-read this document before releasing.

> **XXX NOTE TO SELF**: consider Charlotte's use-cases: https://docs.google.com/spreadsheets/d/1i0WKhIY04G38OQJ0KzbiIEUHat_5Wt-3kvFDMlrovK0/edit#gid=1999745637

> **XXX NOTE TO SELF**: consider Filip's presentation of workflow editor UX: https://discuss.folio.org/t/zap-workflows-ux-iteration-3-english/1041

