# Handoff: accessible plots (alt text) and accessible PlutoUI controls

You are picking up work in the Julia package PlutoTeachingTools.jl. The goal is to make it easy
for notebook authors to (a) display plots with alt text that meets WCAG 2.2 Level AA, and (b) add
interactive PlutoUI controls that meet WCAG 2.2 AA, including ARIA labels. Some of this belongs in
this package. Some of it can only be fixed upstream in PlutoUI.jl, so part of the deliverable is a
ready-to-open upstream pull request.

Read this whole file before touching code. Everything below was verified against the current
sources on 2026-09-18.

## Ground rules

- Work on branch `claude/elegant-ride-selbxt`. Commit with clear messages and push with
  `git push -u origin claude/elegant-ride-selbxt`. Do not open a pull request on this repo unless
  the user asks for one.
- Julia is not installed in this container. Check `which julia` first. If it is missing, try
  installing it with juliaup (`curl -fsSL https://install.julialang.org | sh -s -- -y`) and run
  the test suite with `julia --project -e 'using Pkg; Pkg.test()'`. If you cannot get Julia
  running, say so plainly in your final report. Do not claim tests pass if you did not run them.
  Note that the `Test.yml` workflow only runs on pushes to `main` and on pull requests, so a push
  to the feature branch does not trigger CI.
- Style: `.JuliaFormatter.toml` sets `style = "blue"`. Four-space indentation, keyword
  arguments after `;`, docstrings on every exported function.
- Linting is part of the test suite (`test/test_linting.jl` runs Aqua, JET, and
  ExplicitImports on Julia 1.10+). Every name you use from another package must be imported
  explicitly at the top of `src/PlutoTeachingTools.jl`, in the style already used there
  (`using PlutoUI: combine # used in footnotes.jl`). Unused imports fail the stale-import check.
- Every user-visible string must go through the i18n framework in `src/i18n/i18n.jl`: declare a
  `something_str(lang::AbstractLanguage) = @error "Please define a translation."` stub there and
  add a method in all six language files (`english.jl`, `french.jl`, `german.jl`, `russian.jl`,
  `spanish.jl`, `chinese.jl`). Machine translations are acceptable for now, mark them with a
  `# TODO: verify translation` comment.
- Match the existing accessibility conventions. Look at `aria_live_status` and `wrap_lang` in
  `src/computational_thinking.jl` (an `@htl` wrapper that adds `lang=` and ARIA attributes), the
  `:focus-visible` CSS in `src/other.jl`, and the tests in `test/computational_thinking.jl` and
  `test/other.jl`, which check `repr(MIME"text/html"(), x)` for the expected attributes.
- Document every change in `accessibility.md`. Its summary table has columns
  `Function(s) | WCAG criterion | Fix`. Retitle the document from WCAG 2.1 to WCAG 2.2 while you
  are there (the 2.1 criteria are unchanged in 2.2, so nothing already listed is wrong).
- Add demo cells to `example.jl`. It is a Pluto notebook file. Keep the `# ╔═╡ <uuid>` cell
  markers and the cell-order list at the bottom of the file consistent, and generate fresh UUIDs
  for new cells. Do not add Plots.jl to that notebook's dependencies unless you also confirm the
  `ExportPluto.yaml` workflow can still build it.
- Bump `version` in `Project.toml` to `0.4.8` (additive features only).
- Do not put model names in commits, comments, or docs.

## Part A: plots with alt text (in this package)

### Why this is needed

Pluto shows a Plots.jl figure by picking the richest MIME the object supports. With the GR backend
that is `image/svg+xml`, so the figure lands in the page as an inline `<svg>` with no accessible
name. Screen readers either skip it or read hundreds of path elements. WCAG 2.2 AA requires a text
alternative (1.1.1 Non-text Content). There is currently no helper for this anywhere in the Pluto
ecosystem. `PlutoUI.LocalResource(path, :alt => "...")` handles image files (and
`RobustLocalResource` in `src/robustlocalresource.jl` forwards those attribute pairs), but nothing
handles an in-memory plot object.

### What to build

Add a new file `src/accessible_figure.jl`, included and exported from `src/PlutoTeachingTools.jl`.

1. A struct and a function, something like:

   ```julia
   plot_with_alt(p; alt::AbstractString, caption=nothing, description=nothing,
                 decorative::Bool=false, width=nothing, lang::AbstractLanguage=default_language[])
   ```

   Return a small struct (say `AccessibleFigure`) rather than a bare `@htl` object, so you can
   define both `Base.show(io, ::MIME"text/html", ...)` and `Base.show(io, ::MIME"text/plain", ...)`.
   The plain-text method should print the alt text, so a notebook exported to plain text still
   carries the description.

2. Rendering rules for the `text/html` method:
   - If `p` is already a `PlutoUI.Resource` (that is what `LocalResource` and `RobustLocalResource`
     return), rebuild it with `:alt => alt` merged into its `html_attributes` field. Read
     `/home/user/juliapluto/plutoui.jl/src/Resource.jl` for the struct layout.
   - Otherwise, if `showable(MIME"image/svg+xml"(), p)`, render with
     `repr(MIME"image/svg+xml"(), p)`, base64-encode it, and emit
     `<img src="data:image/svg+xml;base64,..." alt="...">`. Fall back to `image/png` the same
     way. If neither MIME is supported (for example a Plotly backend figure, which only offers
     `text/html`), throw an `ArgumentError` that names the supported MIMEs and suggests switching
     backend or passing an image file.
   - Prefer `<img alt>` over an inline `<svg role="img" aria-label>`. Assistive technology
     handles `<img alt>` uniformly, and an inline GR SVG exposes every path node to the
     accessibility tree.
   - Wrap in `<figure>`. If `caption` is given, add `<figcaption id=...>` and put
     `aria-labelledby` on the figure. If `description` is given (for complex charts, where the
     short alt is not enough), render it after the image inside a `<details>` whose `<summary>`
     text comes from a new i18n string (English: "Figure description"), and point
     `aria-describedby` on the `<img>` at the description element. Follow the W3C complex-images
     pattern: https://www.w3.org/WAI/tutorials/images/complex/
   - `decorative=true` emits `alt=""` and requires no `alt`. Otherwise `alt` is required. Throw a
     helpful error if it is missing or empty, and `@warn` once if it is longer than about 150
     characters, pointing the author at the `description` keyword.
   - Use `Base64` (add it to `[deps]` and `[compat]` in `Project.toml`, it is a stdlib).
   - Apply `lang=` on the figure via `get_language_code(lang)`, matching `wrap_lang`.
   - Generate unique ids per call the way `Columns` in `src/present.jl` does
     (`string(rand(UInt64); base=16)`), so several figures on one page do not collide.

3. Docstring guidance for authors, since the package can only enforce the presence of alt text,
   not its quality. Include a short checklist: say what the chart shows and the main takeaway,
   mention axes and units, do not start with "image of" or "plot of", put trends and key numbers
   in `description` when there are more than a sentence of them.

4. Tests in a new `test/accessible_figure.jl`, included from `test/runtests.jl`. Do not add
   Plots.jl to the test dependencies. Define a tiny fake type in the test that implements
   `Base.show(io, ::MIME"image/svg+xml", ::FakePlot)` and check:
   - the HTML contains `<figure`, `<img`, `alt="..."` with escaping of quotes and `<`,
   - `decorative=true` gives `alt=""`,
   - missing alt throws, over-long alt warns,
   - `caption` produces `<figcaption` and `aria-labelledby`,
   - `description` produces `aria-describedby` and `<details>`,
   - a `PlutoUI.Resource` input gets an `alt` attribute in the rendered `<img>`,
   - a type with no image MIME throws `ArgumentError`,
   - `repr(MIME"text/plain"(), fig)` contains the alt text.

### Optional, only after everything else is done: a Plots.jl extension

The maintainer uses Plots.jl, so a draft-alt-text helper for it would be valuable. Do this as a
package extension so Plots never becomes a hard dependency:

- Add `[weakdeps] Plots = "91a5bcdd-55d7-5caf-9e0b-520d859cae80"` and
  `[extensions] PlutoTeachingToolsPlotsExt = "Plots"` to `Project.toml`, with a `Plots` compat
  entry (Aqua checks that weakdeps have compat too). Extensions need Julia 1.9, but the current
  `julia = "1.6"` compat can stay: older Julia versions ignore `[weakdeps]` and simply never load
  the extension. CI runs LTS, stable, and pre-release, so the extension is exercised there.
- Implement `plot_description(p::Plots.Plot)` in `ext/PlutoTeachingToolsPlotsExt.jl`, returning a
  draft string built from the plot title, `xlabel`, `ylabel`, the number of series and their
  `label`s, and the x and y extents. Read attributes through `Plots` public accessors (for example
  `p[1][:title]`, `p[1][:xaxis][:guide]`, `p[1].series_list`) and verify the attribute names in
  the installed Plots version rather than trusting memory. Document loudly that the result is a
  starting point and that the author must edit it: an auto-generated string is not a WCAG-conformant
  text alternative on its own.
- Also provide `accessible_plot_attrs()` returning a keyword-argument NamedTuple authors can splat
  into `plot(...)`: a colour-vision-deficiency-safe palette (ColorSchemes.jl ships `:okabe_ito`,
  confirm it resolves through `Plots.palette(:okabe_ito)`), cycling `linestyle` and
  `markershape` so series are distinguishable without colour (1.4.1 Use of Color), and
  `linewidth=2` or more so lines and markers reach the 3:1 ratio in 1.4.11 Non-text Contrast.
- Define a generic method in the main package that throws a clear "load Plots.jl to enable this"
  error when the extension is not loaded, so the name exists either way.
- Test the extension only if Plots can be installed in the test environment in reasonable time.
  If it cannot, add a `test/Project.toml` entry anyway and gate the testset on
  `Base.get_extension(PlutoTeachingTools, :PlutoTeachingToolsPlotsExt) !== nothing`.

## Part B: accessible PlutoUI controls

### What the upstream widgets actually emit today

Read these before designing anything. The PlutoUI source is cloned read-only at
`/home/user/juliapluto/plutoui.jl` (version 0.7.83, `src/Builtins.jl` unless noted). Its source
files are Pluto notebooks, so they carry `# ╔═╡` cell markers that must be preserved.

| Widget | Markup | Accessibility gap |
|---|---|---|
| `Slider` | `<input type="range" min=1 max=N value=i>` plus, only when `show_value=true`, a `<script>` and an `<output>` | No accessible name. The DOM value is a 1-based index into `slider.values`, so a screen reader announces "3" for a slider over `0.0:0.1:1.0`. Needs `aria-valuetext` updated on input. The `<output>` has no `for=` link to the input. |
| `NumberField` | `<input type="number" ...>` | No accessible name. |
| `CheckBox` | `<input type="checkbox">` (plain string, not `@htl`) | No accessible name. |
| `Switch` (`src/Switch.jl`) | `<input type="checkbox" class=pui-switch>` styled to `2.2em x 1em` | No accessible name, no `role="switch"`, and the 1em height is below the 24 CSS px minimum in 2.5.8 Target Size. |
| `TextField` | `<input type="text">` or `<textarea>` | No accessible name. `placeholder` is not a substitute for a label. |
| `Button` / `LabelButton` | `<input type="button" value=label>` | Fine on its own, the `value` is the name. |
| `Select` | `<select><option value='puiselect-i'>` | No accessible name. |
| `MultiSelect` | `<select multiple title='Cmd+Click...'>` | No accessible name. The `title` is a tooltip, not a label, and the Ctrl-click instruction is mouse-only wording. |
| `Radio` | `<form>` of `<div><input type=radio id=...><label for=...>` with a JS `value` property on the form | Each option is labelled, but the group has no name. Needs `<fieldset><legend>`. |
| `MultiCheckBox` (`src/MultiCheckBox.jl`) | `<plj-multi-checkbox>` custom element filled by JS | Same group-name gap as `Radio`. |
| `DateField`, `TimeField`, `ColorStringPicker`, `RangeSlider` | bare inputs | No accessible name. |

`@bindname` (`src/bindname.jl`) already wraps its widget in a `<label>` containing the variable
name, which is why it is accessible and the plain `@bind` form is not.

There is no accessibility issue or pull request open on PlutoUI.jl. The closest upstream
discussion is Pluto.jl issue 2768, "Improve screen reader support for reactive output changes",
https://github.com/fonsp/Pluto.jl/issues/2768, which is about Pluto's own output area and not
about widgets.

### The wrapping mechanism you must use

Pluto's `@bind` reads `.value` from the first element of a cell's output and listens for `input`
events on it. Wrapping a widget in arbitrary HTML breaks that unless the wrapper forwards the
value. PlutoUI solves this with `combine` (`src/Combine.jl`): the wrapper `<span>` gets a JS
`value` property, and each `Child(widget)` call is wrapped in
`<pl-combined-child key=...>` so the script can find the real inputs. `PlutoUI.Experimental.wrapped`
(`src/Wrapped.jl`) is the one-child version:

```julia
wrapped(f::Function) = transformed_value(only, combine(f))
```

This is the foundation for everything in Part B1. Two things to know:

- `combine` is exported as `PlutoUI.combine` (this package already imports it for footnotes).
  `wrapped` and `transformed_value` live in the `PlutoUI.Experimental` submodule and are not
  exported. Import them as `using PlutoUI.Experimental: wrapped`. The compat entry
  `PlutoUI = "0.7.56"` already covers this API, but note in the docstring that it depends on an
  experimental PlutoUI API.
- An implicit label (`<label>Text <input></label>`) gives the input an accessible name with no
  need for an `id` on the input, which matters because we cannot add attributes to the inner
  `<input>` from Julia. Attributes that must sit on the input itself (`aria-describedby`,
  `aria-valuetext`) have to be set by a small `<script>` emitted next to the widget, the same way
  PlutoUI's own Slider script uses `currentScript.previousElementSibling`.

### Part B1: helpers in this package (do this first, no upstream dependency)

Add `src/accessible_controls.jl`, included and exported from `src/PlutoTeachingTools.jl`.

1. `labeled(widget, label; description=nothing, lang=default_language[])`

   Usage: `@bind n labeled(Slider(1:100), "Number of samples"; description="Larger is slower.")`.

   Implement with `wrapped() do Child ... end`. Emit
   `<label class="ptt-labeled"> <span>label</span> $(Child(widget)) </label>` and, when
   `description` is given, a `<span id=...>` after it plus a script that finds the input inside the
   label (`label.querySelector("input, select, textarea, [role]")`) and sets
   `aria-describedby` to that id. Add `lang=` via `get_language_code(lang)` when the label text
   might be localized. Covers 1.3.1 Info and Relationships, 3.3.2 Labels or Instructions, and
   4.1.2 Name, Role, Value.

   Make sure `labeled` works for every widget in the table above. `Radio` and `MultiCheckBox`
   are groups, so for those `labeled` should emit `<fieldset><legend>` instead of `<label>`.
   Dispatch on the widget type (`PlutoUI.Radio`, `PlutoUI.MultiCheckBox`), and document that
   authors should use `fieldset_group` (below) for hand-built groups.

2. `fieldset_group(legend, widgets...)` or a `do`-block form using `combine`, so authors can group
   several related controls under one `<fieldset><legend>`. Return the combined value as a tuple
   or NamedTuple exactly as `combine` does.

3. Slider value announcement. When `labeled` wraps a `PlutoUI.Slider`, emit a script that sets
   `aria-valuetext` on the range input to the formatted value on every `input` event and once at
   load. You can reuse the same approach PlutoUI's Slider uses: interpolate
   `string.(slider.formatter.(slider.values))` into the script as a JS array and index it with
   `input.valueAsNumber - 1`. Do this whether or not `show_value` is set. If `show_value` is set,
   also give the `<output>` a `for=` attribute pointing at a generated input `id`. This is a
   workaround until the upstream fix in Part B2 ships, so mark it clearly in a comment and in
   `accessibility.md`.

4. `accessible_controls_css()`: a function returning a `<style>` block an author places once at
   the top of a notebook. It should provide a `:focus-visible` outline for `input`, `select`,
   `textarea`, and `button` inside `pluto-output` (2.4.7 Focus Visible, matching the existing
   `#4A90E2` outline in `src/other.jl`), and enforce a minimum 24x24 CSS px hit area for
   checkboxes, radio buttons, the `.pui-switch` element, and range-slider thumbs (2.5.8 Target
   Size (Minimum)). Scope every selector under `pluto-output` so Pluto's own chrome is untouched,
   and keep the rules overridable (no `!important` unless a test shows PlutoUI's own inline
   styles win).

5. Docstrings and a short authoring guide section in `accessibility.md` covering: always give a
   control a visible label, do not rely on placeholder text, prefer `labeled` over `@bindname` when
   the variable name is not meaningful to a reader, group related controls with a legend, and
   include `accessible_controls_css()` once per notebook.

6. Tests in `test/accessible_controls.jl`, included from `test/runtests.jl`. Check the rendered
   HTML string for each widget type: the `<label>` or `<fieldset><legend>` is present and contains
   the label text, the widget markup is inside it, `aria-describedby` appears when `description`
   is given, the Slider variant contains `aria-valuetext`, `accessible_controls_css()` contains
   `:focus-visible` and `24px`. Also test that `PlutoTeachingTools.labeled(Slider(1:3), "x")`
   reports the right initial value through `AbstractPlutoDingetjes.Bonds.initial_value`, since
   that is what `@bind` uses outside Pluto. AbstractPlutoDingetjes is already a transitive
   dependency via PlutoUI, so add it to `test/Project.toml` as a direct test dependency.

Verification you cannot do here: real keyboard and screen-reader behaviour inside a running Pluto
session. List in your final report exactly what the user should check by hand (Tab reaches each
control, the announced name matches the label, the slider announces real values, Radio announces
the legend, focus ring visible, no horizontal overflow at 320 px width).

### Part B2: upstream pull request for PlutoUI.jl (prepare it, do not skip it)

Some of these gaps can only be fixed properly inside PlutoUI, because that is where the
`<input>` is built. Prepare the change in the read-only clone at
`/home/user/juliapluto/plutoui.jl` on a new local branch.

Scope of the upstream change, keep it small and backwards compatible:

1. Add a keyword argument to every builtin widget in the table above, so authors can attach
   attributes directly to the input element. Suggested name and type, mirroring `Resource`:
   `html_attributes::NamedTuple = (;)`. Store it as a new struct field with a default, keep every
   existing positional constructor working, and merge it into the existing `@htl` attribute
   NamedTuples (`$(merge((type="range", ...), slider.html_attributes))`). `CheckBox` and `Switch`
   build their markup with plain string interpolation, so convert those to `@htl` first. Read the
   compat branches (`OldSlider`, `OldSelect`) and leave them alone.

2. `Slider`: always emit the script (not only when `show_value=true`), set `aria-valuetext` from
   the formatted value on every `input` event and at load, and give the `<output>` a `for=`
   attribute tied to a generated id on the input.

3. `Radio` and `MultiCheckBox`: accept `label=nothing`. When given, wrap the options in
   `<fieldset><legend>label</legend>...</fieldset>`. For `Radio` the JS `value` property is
   defined on the `<form>`, so put the fieldset inside the form and keep the property where it is.

4. `Switch`: add `role="switch"` and increase the default `--height` so the control meets the
   24 px minimum, or add a transparent padding hit area if the maintainers would object to a
   visual change. Explain the tradeoff in the PR.

5. `MultiSelect`: change the `title` text so it also describes the keyboard method
   (Shift+Arrow or Ctrl+Space, depending on platform) rather than only Ctrl-click.

6. Tests: look at `/home/user/juliapluto/plutoui.jl/test/` first and follow its conventions.

Deliver it like this:

- First try `add_repo` for `JuliaPluto/PlutoUI.jl` with `access: "push"`. If that succeeds, fork
  the repo with the GitHub tools, push the branch to the fork, and open a draft PR against
  `JuliaPluto/PlutoUI.jl` with a description that lists each WCAG criterion addressed, links to
  the W3C understanding pages, and states that nothing changes for notebooks that do not pass the
  new keywords.
- If push access is refused (a cross-owner attach may be), do not stop. Save
  `git diff main` from the PlutoUI clone as `docs/upstream/plutoui-accessibility.patch` in this
  repository, plus `docs/upstream/plutoui-accessibility-PR.md` containing the PR title and body
  ready to paste. Say clearly in the report which of the two happened.
- Once the upstream keyword exists, `labeled` in this package should prefer it over the
  attribute-setting script. Leave a `TODO` with the upstream PR link at the spot in
  `src/accessible_controls.jl` where that switch would go, and add a compat note in
  `accessibility.md`.

## WCAG 2.2 AA criteria to cite

Cite these by number in docstrings, comments, tests, and `accessibility.md`, with links of the
form `https://www.w3.org/WAI/WCAG22/Understanding/<slug>`.

| Criterion | Slug | Where it applies here |
|---|---|---|
| 1.1.1 Non-text Content | `non-text-content` | Plot alt text, decorative images |
| 1.3.1 Info and Relationships | `info-and-relationships` | Label association, fieldset/legend, figure/figcaption |
| 1.4.1 Use of Color | `use-of-color` | Line styles and markers in `accessible_plot_attrs` |
| 1.4.3 Contrast (Minimum) | `contrast-minimum` | Any text you style |
| 1.4.11 Non-text Contrast | `non-text-contrast` | Plot lines and markers, focus ring, slider thumb |
| 2.1.1 Keyboard | `keyboard` | Every control must work without a mouse (native inputs do) |
| 2.4.7 Focus Visible | `focus-visible` | `accessible_controls_css` |
| 2.4.11 Focus Not Obscured (Minimum) | `focus-not-obscured-minimum` | New in 2.2. Check `aside` does not cover focused controls |
| 2.5.7 Dragging Movements | `dragging-movements` | New in 2.2. Range inputs already have arrow-key alternatives, say so |
| 2.5.8 Target Size (Minimum) | `target-size-minimum` | New in 2.2. Checkboxes, radios, Switch, slider thumb |
| 3.1.2 Language of Parts | `language-of-parts` | `lang=` on localized labels and captions |
| 3.3.2 Labels or Instructions | `labels-or-instructions` | `labeled`, `description` |
| 4.1.2 Name, Role, Value | `name-role-value` | Accessible names, `role="switch"`, `aria-valuetext` |
| 4.1.3 Status Messages | `status-messages` | Already handled for feedback boxes, do not regress it |

WCAG 2.2 removed 4.1.1 Parsing, so do not cite it. Reference for what changed between 2.1 and
2.2: https://www.w3.org/WAI/standards-guidelines/wcag/new-in-22/

## Order of work and definition of done

1. Part A core (`plot_with_alt`), with tests, i18n strings, example cells, `accessibility.md`.
2. Part B1 (`labeled`, `fieldset_group`, `accessible_controls_css`), same checklist.
3. Part B2 upstream PR or patch.
4. Part A optional Plots extension, only if 1 to 3 are complete and tested.

Done means: all new functions exported and documented, `Pkg.test()` passes locally (or you
report exactly why you could not run it), every language file has the new strings, `example.jl`
demonstrates each new function, `accessibility.md` lists each change with its criterion, the
version is bumped, and the branch is pushed. Your final report should list what was verified by
running code, what was verified only by reading, and the manual checks left for the user.
