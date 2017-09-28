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
* [Front-end/back-end interaction](#front-endback-end-interaction)
* [Implementation analogies](#implementation-analogies)
    * [Finite state machine with transitions](#finite-state-machine-with-transitions)
    * [Makefile-like dependency tree](#makefile-like-dependency-tree)
    * [Connector Framework](#connector-framework)
    * [Very high-level DSL (like Okapi CLI)](#very-high-level-dsl-like-okapi-cli)
* [Implementation strategy](#implementation-strategy)
    * [Virtual Machine for running workflows](#virtual-machine-for-running-workflows)
    * [Expressing workflows](#expressing-workflows)
        * [XML or JSON](#xml-or-json)
        * [Human-readable/writeable DSL](#human-readablewriteable-dsl)
        * [VM](#vm)
        * [Visual](#visual)
        * [Discussion](#discussion)
    * [Tracking jobs](#tracking-jobs)
    * [Operations](#operations)
        * [Okapi calls](#okapi-calls)
        * [Transformations](#transformations)
        * [Control flow](#control-flow)
        * [Subroutines](#subroutines)
        * [Other](#other)
* [Error handling](#error-handling)
* [Appendix: using the notification system](#appendix-using-the-notification-system)
* [Appendix: object types](#appendix-object-types)



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

At this stage, the contents of this document should be considered as a foundation for discussion, rather than as a design for implementation.


## Background

The term "workflow" in FOLIO has been used in several different ways and has accumulated a lot of connotations. Among other things, it is a wildly popular future feature, which has been used to great effect by the marketing team. But it has yet to be clearly defined what it consists of, and seems to have been interpreted in different ways by different groups.

What mean here by "workflow" is that facility of a FOLIO system to guide human users through sequences of operations, filling in as much of the work as it can. This encompasses cases where _all_ parts of a workflow can be done by the computer without human intervention: in this case we refer to the process as "automation"; but automation in this sense is not a conceptually separate thing from workflow, it is simply workflow in which none of the steps require human intervention.

Workflow in FOLIO is more important than in other library systems, because FOLIO is composed of numerous small applications which interact. Workflow (including automation) will be the glue that binds them together into a powerful, flexible, configurable whole. Rather than a single monolithic Acquisitions application with tightly bound procedures built into it, we envisage smaller, substitutable applications such as Ordering, Invoicing and Unboxing, each of them operated primarily in terms of higher-level procedures defined as workflows.

Although "workflow" has been thought of as a FOLIO v2 feature, we need to think about this now, because the term has been used in two quite separate senses. The v2 feature is the visual workflow editor that has been [prototyped by Filip Jakobsen](http://ux.folio.org/prototype/en/workflows?view=full) and which will allow librarians to set up their own workflows. (For more on a UX-oriented perspective: see [the "Workflow app" issue in the UX Jira project](https://issues.folio.org/browse/UX-52).)

But well before this is required, FOLIO will need the underlying mechanisms for executing workflows. Until we have the visual editor, we can make do with hand-authored workflows: creating such workflows will be part of the job of developing high-level applications.

We need to think about this now, rather then deferring until after the v1 release, because otherwise we will need to waste a lot of re-engineering down the line: replacing what in effect will be hardwired workflows with applications of the workflow engine. Want to avoid wasting development effort on hardwired workflows that we are only going to throw away.


### Terminology

* A _workflow_ is a set of instructions for FOLIO and/or human users, expressing how to carry out a task. In v1, we will author these by hand, and in v2 there will be an editor. Either way, except when being edited, these are static, like classes in objct-oriented programs.

* A _workflow instance_ is a specific running instance of a workflow. It has parameters that may be set when it's started (e.g. the title of a book to order) and its own state that changes as the task progresseds (e.g. the status of an ongoing order). It resembles and instance of class in an objct-oriented program.

* A _job_ is a simpler term for a workflow instance.

* The _workflow editor_ is the visual editor for creating workflows, which will be created as part of FOLIO v2.

* The _workflow engine_ is the software that executes workflows on behalf of a user. It will exist quite separately from the workflow editor, which is only one way of authoring a workflow.

* A _workflow language_ is a textual representation of a workflow, which can be authored by a developer. See [below](#expressing-workflows). Since this will be a domain-specific language (DSL) this may also be referred to as a _workflow DSL_.



## Review of existing workflow systems

XXX Nassib volunteered to do this: no issue filed yet AFAIK.



## Example workflows

To ensure that FOLIO's workflow capabilities are sufficiently expressive to execute real library workflows, we need to establish some concrete scenarios. We will then be able to express these in a workflow language and consider how a workflow engine could execute them.

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

  * XXX How do we decide what the shelving location should be? Is that information already included in the data file that comes with the physical books, or do we need to make that decision and add it to the item records? Is the location the same for all books that are items of the same instance? And by "location" here, do we mean shelfmark, call number, or location in the technical sense?

    Peter comments:
    > Shelving locations will typically be determined by the call number (or shelf mark -- essentially different phrases for the same thing).  The call number specifies a specific ordering of the books on a shelf.  In any case of an academic library I can think of, the first part of the call number specifies the broad subject area.  (For instance, Dewey Decimal _560s_ and Library of Congress _QE_ -- both representing the subject of geology and dinosaurs.)  It is based on that first part that there would be rules for shelving items in one location or another.

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

This raises several questions:

* Do we want to implicitly pass records from one step in a workflow to the next, analogously to how Unix pipes data between processes? Or do we need to workflows to explicitly name objects and refer to them by name subsequently? The former is terser and more elegant when it works, but the latter is more explicit and expressive. It will be possible to think about this in more detail once we start to sketch the representation of workflows (see below).

* Should workflow-step inputs and outputs be objects or references to them? In the former approach, we would be passing, for example, item records around. In the latter, we would be passing only the ID that points to the relevant item in the data store.

* Whichever of these approaches we adopt, we will need an explicit notion of object type. If we pass only IDs, we will need to know the type of each object so we can look it up in the appropriate back-end service when we need data from it; and even if we pass whole objects, we will still need to know the type so that we can determine what service to use to persist changes. For the purposes of this document, we will refer to objects with a simple [URN](https://en.wikipedia.org/wiki/Uniform_Resource_Name)-like scheme where type-specific IDs look like `item:234` or `instance:543`.

* Does a workflow instance have its own state separate from the state of the objects it deals with? That is, does it suffice to mark a purchase as "needs authorization", or do we also need to mark the workflow instance that generated this purcahse as "awaiting purchase authorization"? The latter allows us to tie together related operations after the event. Workflow is a great source of audit logging capability, perhaps saving individual services from having to do excessive audit logging.

XXX Nassib's experience of workflow engines: pain comes in standardising inputs and outputs

XXX Should be avoidable here as data will be canonicalised on entering the system.



## Front-end/back-end interaction

XXX Goal is for most or all mechanism to be server-side. That is necessary for it to function in automation. Front-end involvement may be unavoidable in some cases.

XXX Workflow editor as UX'd by Filip is separate.

XXX Much will be done on back-end, though effects will be seen on front-end: e.g. partially pre-filled form that must be completed and submitted.



## Implementation analogies


### Finite state machine with transitions

XXX


### Makefile-like dependency tree

XXX


### Connector Framework

XXX


### Very high-level DSL (like Okapi CLI)

XXX Domain-specific loop control, e.g. `boxOfBooks.foreach(book) { ... }`.



## Implementation strategy

### Virtual Machine for running workflows

XXX Somewhat like the CF Engine

### Expressing workflows

XXX Bizarrely, there may be up to four of these.

#### XML or JSON

XXX Probably the persistent form that is stored in the FOLIO DB, exported and imported, etc.

#### Human-readable/writeable DSL

XXX May be needed before we have Visual editor, unless we want to hand-write XML or JSON

XXX Since this will be parseable, maybe we'll make it the persistent form and not bother with XML/JSON at all.

#### VM

XXX XML/JSON or DSL will be compiled to some representation that can be executed by the Workflow VM. We may store the compiled version (analogous to Java `.class` files), or maybe just compile and run from the internal representation as Perl and Python do.

#### Visual

XXX In v2, Filip's UK will guide us into a visual representation of workflows. The Workflow editor UI module will need to parse stores workflows (from XML/JSON or a DSL) to determine what to draw; and render edited drawings into a form to send back to the server.

#### Discussion

XXX Probably don't need all four representations. Which to omit?

XXX Programming language: since we will need to parse/render on the client side for the V2 workflow editor, we will need implementations in JS. That suggests we should use those JS implementations on the server side too. This may mean the first Okapi module written in JS, or may entail somehow calling out from an RMB-based module into the JS compiler/renderer.


### Tracking jobs

XXX A back-end module for CRUDding the status of jobs based on workflows. Will likely consist of a tree of pointers to job-step objects, each with its own state. Will not need to be transmitted, so no need for a serialised or human-readable form.


### Operations

#### Okapi calls

XXX CRUDding objects

#### Transformations

XXX Adding fields, modifying them, deleting them

#### Control flow

XXX if/then/else, while, foreach, do-in-parallel

#### Subroutines

XXX Call another named workflow with arguments

#### Other

XXX What else?



## Error handling

XXX If a step fails, do we fail the whole job? We can report back to the person who initiated the job.

XXX In the unboxing example: "here’s a report of the three things we weren’t able to do anything with."

XXX The workflow may have multiple operations on the go at once (e.g. "when these three parallel tasks are complete, go on to Step X). How do we handle recovert if one of them fails after others have already succeeded?



## Appendix: using the notification system

XXX We will want to do all notification using the notification system: "please authorize this", "your request has been accepted/rejected", "your job failed", etc. Users will want different kinds of message communicated in different ways, e.g. some by email, some by text-message, some waiting on the system until the next login. So notifications need a "type" drawn from a small controlled vocabulary (as well as other extensions such as sender, date, subject)



## Appendix: object types

XXX May need to be made explicit, with preferences registered to each type as in the MacOS registry: e.g. "When I get an object of type `user` I want to use the `@someVendor/users` app to open it.


&nbsp;

> **XXX NOTE TO SELF**: get David to proof-read this document before releasing.

