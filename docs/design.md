# Design

Semi-structured thoughts about what this software does, and how to write it.

## Intent

I want to write checklists as Markdown files and render them into a compiled manual in the aviation technical order-style.

The resulting publication should be suitable for print in a standard checklist size (does not have to be variable).

The project should use standard Markdown - or a well-known version of the format - as much as practical.
A non-technical writer should be able to create a checklist in a text editor, using a few specific pieces of syntax
that are familiar (ex: bullet points).

Ideally, this resulting application would provide a GUI to guide the writer through finding the files to process.

## Technical Order Checklists

Examples of technical order (*TO* or *T.O.*) checklists can be found online, and one such example is the [Hellenic F-16 checklist](./HAF-F16-FlightChecklist.pdf) in this directory.
They are typically highly structured, abbreviated versions of full manuals for flight or other disciplines.

Checklist publications are built using different levels of content, from structure of the overall T.O. (ex: sections like Normal Operations) to the individual checklist (ex: Interior Inspection).

They contain different border styles and margin markings to indicate emergency checklists, section boundaries, and recent changes.

Each page is typically laid out as a single column with page number, checklist name, and other text as metadata.
In addition, each page may also have figures, tables, or multiple columns (or all of those together).

The overall publication is usually taller than it is wide, in the same general proportions as a common novel. 
T.O. checklist sizes are varied, but a large range of sizes near 15x22 cm can be appropriate.
They are meant to be bound on the long edge, typically with a few flexible rings.
Pages have alternating margins based on page side, with the inner/center margin wider than the outer margin, to support holes for binding without losing content.

## Technical Considerations

### Requirements

Overall, the code should support a few things separately, including and probably not limited to:

* User interface
* Checklist file discovery
* Markdown parsing (including custom syntax)
* Parsed content unification (more [below](#content-unification))
* Checklist storage (maybe?)
* Checklist rendering

It must also support being written slowly by a single person, in short bursts over a long period of time.

My initial instinct is to focus on the file discovery --> rendering chain, and develop an API that a CLI, TUI, server, ... could all use.

#### User Interface

A graphical interface is easy for non-technical writers to use; such a UI can be native code or web-based.
This kind of UI could also support autocomplete and other interesting tooling!

Let's imagine we want both of these, plus a CLI, and punt on implementation details.
The upshot is that the underlying application must expose a coherent API for frontend integration.

#### File Discovery

The application should support many checklists, spread over one or many directories as one or many files.

Writers are not bound to a single-checklist-per-file schema, and neither are they bound to write thousands of lines into a single file.

More on how we handle this in [content unification](#content-unification).

##### Implementation

Let's choose a root directory and recursively include every `.md` file.

In the future, support a limited set of filepath-level exclusions, like directory names or filenames.
We can always add more functionality here.

#### Parsing

A custom parser is not in the scope of this project. We can use an off-the-shelf parser with support for plugins.

At the same time, Markdown wasn't made for describing this kind of document.
We may need to implement custom symbols for columns, figures, and special purpose annotations.
Examples of special purpose annotations are notes, cautions, and warnings; title page information; justified text; other non-standard layout considerations.
This project does not aim to support the full bevy of Markdown features, either. Triple nested quotes, for example, would be out of place. 
The end result is a targeted, lighter version of Markdown, with specific grammar for known layouts.
We could call this new flavor of Markdown something like... CheckMark (sorry).

Regardless, we need to understand the semantics around existing Markdown syntax, and enumerate any custom syntax.
This table will explain non-standard uses of existing syntax and custom syntax requirements.

| Syntax   | Meaning                                              |
|----------|------------------------------------------------------|
| `#`      | Publication (ex: `T.O. 1F-16-CJ-1CL-2`)              |
| `##`     | Section (ex: `Emergency Procedures`)                 |
| `###`    | Subsection (ex: `Electrical Emergencies`)            |
| `####`   | Checklist (ex: `Inverter Failure`)                   |
| `#####`  | Subchecklist (ex: `After landing:` - A-14.1)         |
| `######` | Information Page (ex: A-14 in example pub)           |
| `>`      | Information heading (when under `######`)            |
| `:1:`    | Note 1 (follow with text on Information Page)        |
| `* :3:`  | Note 3 on Information page (text to follow)          |
| `:2c:`   | Caution 2                                            |
| `:4w:`   | Warning 4                                            |
| `:5s:`   | Nothing - this renders without special consideration |
| `:col:`  | Column (left/right - in order of appearance)         |
| `>emer<` | Emergency - applies status recursively               |

The Information Page is typically the left side page, facing the checklist opposite.
An item of information can be general, or a note/caution/warning, or both.

An example checklist:

```
# T.O. 1F-16CJ-1CL-2

## Normal Procedures

## Maintenance Procedures

## Emergency Procedures >emer<

### Miscellaneous Emergencies

#### Out of Coffee

* Maintain aircraft control. :1:
* Autopilot - Set. :2w:

:col: If energy drink is available:

* Energy drink - Consume. (as required)
* Land as soon as practical.

:col: If no energy drink is available:

* Oxygen - 100%.
* Land as soon as possible.

##### After landing:

* Stop straight ahead and engage parking brake. :3c:

###### Information

> Major Inoperative Equipment:

* Stick actuator. Throttle position control assembly. Decision module.

> Other Considerations:

* :1: If depleted coffee stores are detected while in the terminal approach phase, continue the approach to land.
* :2w: Do not set autopilot while in a critical phase of flight.
* Some energy drinks may not be approved for coffee substitution. Do not consume disapproved energy drinks.
* :3c: Brakes should be applied in a single, moderate, and steady application without cycling the antiskid.
```

#### Content Unification

Imagine we have one large file - the place our checklist was born.
Over time, it grew unwieldy, so future sections got their own files.
One day, even that was too much, so some checklists have their own files.

Now we have many files with varying levels of meaning - how should we understand this structure?
This is what I mean by `content unification`.

##### Implementation

At its simplest, this implementation proposal is merely bucketing each heading level to construct one overall tree.

Directory depth and name do not matter.
The directory tree may inform our bucketizing algorithm for those files without complete tree information - most files.

##### Rules and Example

These rules are largely written for headings, which will render as sections, checklists, even the entire publication!

They might also apply in other cases, too (like notes, cautions, and warnings).

1. A node falls under the immediately preceding parent-level node (ie: heading level 3 falls under the heading level 2 just before it).
2. Order of precedence for nodes is:
    i. Same file, nearest vertically above first (if multiple parent candidates in file)
    ii. Same directory, nearest prior file (increasing lexographically) first - this respects intentionally numbered files
    iii. Apply the rules recursively at shallower depths until a match

```
- my-checklists/
  |- inspections.md
  |- engine-start.md
  |- winterizing-ops.md
  |- emergency/
  |   |- failed-bilge-pump.md
  |   |- fire-on-board.md
  |- preparation/
      |- preparation.md
      |- common-regulations.md
      |- signals.md
```

Without the contents of the above files, it's tough to apply the rules above - still, let's try some cases.

| Case                                                               | Subcase                                                                                                 | Outcome                                                                                                                                             |
|--------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| `emergency/` has no section-level heading                          | `failed-bilge-pump.md` has `#` through `####` level headings                                            | Checklists in this file are nested properly                                                                                                         |
|                                                                    | `failed-bilge-pump.md` has only `####` headings                                                         | Content unification looks at `fire-on-board.md` for higher-level headings, then the files in the root directory (`winterizing-ops.md` first)        |
| `preparation/preparation.md` has a section heading with no content | `signals.md` has only checklist-level headings                                                          | All checklists in `signals.md` fall under the section defined in `preparation.md`                                                                   |
|                                                                    | `common-regulations.md` has some checklist level headings, then a section heading, then more checklists | All checklists in `common-regulations.md` fall under the single section outlined in that file, regardless of relative vertical position in the file |

In potentially ambiguous situations, we should output clear logs about the choice that we made and why, plus how to change/fix if our choices don't reflect the intent of the writer.

#### Storage

Explore storage in a database for later retrieval and/or rendering without parsing - or for backup and sharing purposes.

An exportable, in-memory database could also be used for autocomplete queries out of the box, versus a custom data structure.

While tempting, it's important to remember that our core storage format is plain, human-readable text (Markdown).
Unless more compelling use cases emerge, it is probably best to leave another storage format for later.

#### Rendering

In the Content Unification block, we explicitly discuss Markdown conversion into concepts of TOs.
To actually make it to the rendering step, we'll need to store the results of parsing in some intermediate data structure.

In the end, a technical order is a linked list - a strictly ordered, flat sequence of pages, beginning to end.
This is true for any kind of document that's meant to be printed and bound in a page-by-page ordering.
That linear structure doesn't work as an intermediate format, from parsing to rendering.
Our language so far (sections, subsections, bullet points) strongly implies a hierarchy with a tree shape.

Such a tree, for the purposes of TO description, has a knowable, limited maximum depth.
It has rules around which types of content go together, and how.
For example, an information page has no place in the root `#` Publication block.

It's likely that existing Markdown packages support the kind of structure we need, without inbuilt support for our rules.
In the interest of time, we should strongly consider using a known-capable interface to a slightly-misfit structure,
even if it means creating an external and tightly coupled validation layer.

Finally, the rendering engine will support automatic layout.
Input from writers must not be required to enable clean page breaks and sensible checklist-oriented flow.

### Application Structure

A shape is emerging from our discussion so far, with three levels of conceptual space.

1. Top level user interfaces
2. Markdown as core UI and storage format
3. Technical order representation and rendering

In a common Go project layout the user interfaces would live in `cmd`, separate from the main logic of the application.

Both the Markdown and TO representation pieces would live in `pkg`, or `internal` if we wish to limit use of our code as a library.
There will be some glue code between these two packages.
We also need to work to avoid tight coupling, especially if we use the same Markdown library to handle parsing and TO representation.

### Deployment

Ideally, we have no deployment. We can use GitHub to host source code and even distributable files.

At the same time, a web UI would add significantly to the usability of the project.
The browser is a familiar interface, requires low trust from writers, and can be very low friction.

I propose to ensure the core application code in this project can be compiled to WASM.
We can embed WASM directly into a single HTML file, along with the JS required to load and run it.
That HTML file can be downloaded directly or hosted cheaply/freely on a CDN somewhere.

## Implementation Alternatives

### Hackfest Script

Just hack together the fastest thing that works, without much regard to extensibility or the future.
Probably use a batteries-included Markdown package with some custom plugins.
Probably cut some corners.
This approach might work. It might stop working rather quickly, too.

### Complete Script

Fastest effective implementation is probably in the form of a big-but-straightforward script. The script will:

* Accept a single command line argument (root directory)
* Recursively scan for Markdown files
* Parse each file and add to a unified tree
* Render the checklist and save to file
* Fail on errors and return any warnings

This script will be tightly coupled to the chosen Markdown parsing library.
It is possible to evolve from the script approach to a more robust application, with a thoughtful approach.

### Real Application

In this alternative, we actually make something that can be used by others.
Some basic shape of the API can be inferred from the steps above. This is a rough idea of what we need:

* `findFiles(...options FileOptions) (Path, error)`
* `parse(filename Path, content Content) (Content, Warning[], error)` - turn a Markdown file into TO data
* `addContent(content Content) (Content, error)` - add content to the technical order representation
* `newWriter(format ChecklistFormat, destination Path) (ChecklistWriter, error)` - different output formats, file types
* `render(root Content, w ChecklistWriter) (Warning[], error)` - turn TO content into the given writer's format

Something like this is a good start, and can conform to the three layers described above.

## Next Steps

1. Start investigating parsers by looking at `goldmark` and similar.
2. Explore the shortfalls in the described syntax for CheckMark by translating pages of the example PDF. Iterate.
3. Choose from the alternatives for implementation - or make a better one.

Once we have examples properly translated to CheckMark, and some candidate parsers, it's time to create prototypes.
Parse the example technical order pages and implement custom rendering.
Check for completeness of the underlying API and ergonomics.
Then pick something and move ahead!
