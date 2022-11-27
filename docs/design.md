# Design

Semi-structured thoughts about what this software does, and how to write it.

## Intent

I want to write checklists as Markdown files and render them into a compiled manual in the aviation technical order-style.

The resulting publication should be suitable for print in a standard checklist size (does not have to be variable).

The project should use standard Markdown - or a well-known version of the format - as much as practical.
A non-technical writer should be able to create a checklist in a text editor, using a few specific pieces of syntax
that are familiar (ex: bullet points).

Ideally, this resulting application would provide a GUI to guide the writer through finding the files to process.

## Checklists

Examples of technical order checklists can be found online, and one such example is the [Hellenic F-16 checklist](./HAF-F16-FlightChecklist.pdf) in this directory.
They are typically highly structured, abbreviated versions of full flight manuals.

Checklists contain headings for the overall technical order (ex: , sections (ex: Emergency and Normal Operations), and checklist (ex: Interior Inspection).

They contain different border styles and margin markings, to indicate emergency checklists, section boundaries, and recent changes.

Each page is typically laid out as a single column with page number, checklist name, and other text as metadata.
Nonethless, each page may also have figures, tables, or multiple columns (or all of those together).

The overall publication is intended to be bound on the long edge, typically with a few flexible rings.
The alternating margin supports holes for binding without losing content.

## Technical Considerations

Overall, the code should support a few things separately, including and probably not limited to:

* User interface
* Checklist file discovery
* Markdown parsing (including custom syntax)
* Parsed content unification (more [below](#content-unification))
* Checklist storage (maybe?)
* Checklist rendering

It must also support being written slowly by a single person, in short bursts over a long period of time.

My initial instinct is to focus on the file discovery --> rendering chain, and develop an API that a CLI, TUI, server, ... could all use.

### User Interface

A graphical interface is easy for non-technical writers to use; such a UI can be native code or web-based.
This kind of UI could also support autocomplete and other interesting tooling!

Let's imagine we want both of these, plus a CLI, and punt on implementation details.
The upshot is that the underlying application must expose a coherent API for frontend integration.

### File Discovery

The application should support many checklists, spread over one or many directories as one or many files.

Writers are not bound to a single-checklist-per-file schema, and neither are they bound to write thousands of lines into a single file.

More on how we handle this in [content unification](#content-unification).

#### Implementation

Let's choose a root directory and recursively include every `.md` file.

In the future, support a limited set of filepath-level exclusions, like directory names or filenames.
We can always add more functionality here.

### Parsing

A custom parser is not in the scope of this project. We can use an off-the-shelf parser with support for plugins.

However, we do need to understand our heading level semantics and any custom syntax.
This table will explain non-standard uses of existing syntax and custom syntax requirements.

| Syntax   | Meaning                                              |
|----------|------------------------------------------------------|
| `#`      | Publication name (ex: `T.O. 1F-16-CJ-1CL-2`)         |
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

### Content Unification

Imagine we have one large file - the place our checklist was born.
Over time, it grew unwieldy, so future sections got their own files.
One day, even that was too much, so some checklists have their own files.

Now we have many files with varying levels of meaning - how should we understand this structure?
This is what I mean by `content unification`.

#### Implementation

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

### Storage

Explore storage in a database for later retrieval and/or rendering without parsing - or for backup and sharing purposes.

An exportable, in-memory database could also be used for autocomplete queries out of the box, versus a custom data structure.

### Rendering

Markdown wasn't necessarily made for rendering this kind of document. 
We may need to implement custom rendering (and even syntax) for columns, figures, and special purpose annotations.

Examples of special purpose annotations are notes, cautions, and warnings; title page information; justified text; other non-standard layout considerations.

The rendering engine should support automatic layout.
Input from writers must not be required to enable clean page breaks and sensible checklist-oriented flow.

## Initial Implementation

Fastest effective implementation is probably in the form of a script. The script will:

* Accept a single command line argument (root directory)
* Recursively scan for Markdown files
* Parse each file and add to a unified tree
* Render the checklist and save
* Output errors and warnings

### Implied API

Some basic shape of the API can be inferred from the steps above. This is a rough idea of what we need:

* `findFiles(...options FileOptions) (Path, error)`
* `parse(filename Path, content ContentTree) (ContentTree, Warning[], error)`
* `newWriter(format ChecklistFormat) (ChecklistWriter, error)`
* `render(content ContentTree, w ChecklistWriter, outFilename Path) (Warning[], error)`

Something like this is a good start.

I'll start investigating parsers by looking at `goldmark` and similar.
