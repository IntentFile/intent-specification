# A print template renders an image from the tenant's files

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7024
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7031

## The problem

A [print template](../versions/1.6.md#printable-documents) can interpolate text. Every layout tag the
1.6 text lists - `page`, `header` / `footer`, `section` / `stack`, `row`, `field`, `text`, `table`,
`total`, `line`, `if` - places text or rules text; none places a picture. So the one thing every
printed business document carries above its title cannot be expressed: **the issuer's logo** on an
invoice, a proforma, a credit note, a delivery note. A suite that prints those without a logo is not
one a small company will issue documents from, and a suite-side "company logo" requirement had been
parked on this gap since the format's print template existed.

The obvious place for the picture is already in the application. A deployment holds files in two
forms: files the tenant uploads beside its print templates (a logo, a stamp, a signature), and files a
record carries through a [`function: Attachment`](../versions/1.6.md#attachments-and-snapshots) child,
whose rows hold the stored file's path. Neither could reach a rendered document.

The problem is sharper than "add a tag". A rendered document is produced on three paths - the
interactive print action, the copy a `function: Snapshot` mints, and the document a notify block's
`attach: print` mails - and the last two run with no browser and no caller. An image reference left in
the template for a viewer to fetch later would render on one path and not the others, or would require
the stored file to be readable without the caller's authorization. The image has to be **part of the
render**.

## The proposed shape

An `image` layout tag. Its `src` is a reference to a stored file, and it is resolved when the document
is rendered:

```text
<document id="sales-invoice-print">
    <page>
        <header>
            <row>
                <stack width="*">
                    <image src="Templates/Print/logo.png" width="120"/>
                    <text style="title">Sales Invoice</text>
                </stack>
                <stack width="*">
                    <text align="right" style="subtitle">{{document.Number}}</text>
                </stack>
            </row>
            <line/>
        </header>
        ...
        <image src="{{document.Signature.StoragePath}}" height="60" align="right"/>
    </page>
</document>
```

`src` takes one of three forms, and the shape of the value says which:

| `src` | what it names |
| --- | --- |
| `Templates/Print/logo.png` - a bare path | a file in **the application's file store**, relative to the store's root; on a shared deployment, the tenant's own store |
| `{{document.<Relation>.StoragePath}}` - a placeholder | a path held by the record: the stored-path field of a `function: Attachment` row reached through a to-one relation, resolved by the ordinary placeholder rules and then read as a bare path |
| `data:image/png;base64,...` - or any `scheme:` URI | carried as written: a `data:` URI already holds the image inline, any other scheme is an address the renderer is left to resolve itself |

`width` and `height` are the resize hints, in the units every layout tag already accepts. `align`
applies as on any block.

No modeling key is added. The issue that motivated the construct sketched two - an
`attachments: { roles: [logo] }` on a master, and a `kind: image` to-one - and this proposal takes
neither. A logo is **per-tenant branding**, not record data: it belongs in the tenant's file store
beside the print templates the tenant already customizes, a value nothing in a model can carry and the
same reason the template itself is create-if-absent. A file *of the record* already has a handle: a
relation to an Attachment child is an ordinary to-one, and its stored path reaches the template through
the placeholder rules that already exist. A `kind: image` would be `manyToOne` under another name.

## Expected behaviour

- **The image is resolved server-side, at render time, and inlined into the render.** A conforming
  generator reads the referenced file while the caller's own scope and authorization still apply and
  embeds its bytes into the rendered document. The output is self-contained: nothing in it is fetched
  later, by anyone, from anywhere.
- **All three render paths carry the same image from the same template.** The interactive print, the
  versioned copy a `function: Snapshot` mints and the document `attach: print` attaches embed the image
  identically, because each is a render and the image is part of the render.
- **A missing image renders nothing** - no block, no reserved space, no placeholder, no broken-image
  box - and the document is produced as if the tag were absent. A tenant that has not uploaded its
  logo yet is the everyday state of a fresh deployment, and a logo that cannot be read must never cost
  the invoice.
- **The generated scaffold carries one shared logo slot**, `Templates/Print/logo.png` at width 120,
  in the header of every document template it writes and in the mailed report template. One path for
  the whole application, not one per document: a company has a logo, not an invoice-logo and a
  separate statement-logo, so branding a deployment is a single upload - or a single
  `doc/Templates/Print/logo.png` shipped with the project and seeded into the store like any other file
  under `doc/`. The slot is emitted unconditionally, which is safe exactly because a missing image
  renders nothing: a deployment that never uploads a logo prints as before, and one that does needs no
  regeneration of a template it may already have adapted by hand.
- **Sizing.** `width` alone or `height` alone scales the image proportionally; both together set both
  dimensions; neither sizes the image to its content. Only an absolute measurement (`120`, `120px`) is a
  hint - a percentage or a fraction weight on an image is ignored.

## Edge rules

- **`src` is trimmed** before the shape is read. An absent or blank `src` renders nothing, and is not
  an error.
- **What counts as a URI** is a leading scheme - a letter followed by letters, digits, `+`, `-` or `.`
  up to a colon. Everything else is a store path. A URI source is emitted exactly as written and is
  **never read** by the generator.
- **A store path is relative to the store's root**, which is the tenant's boundary. A path carrying a
  `..` segment is refused: it renders nothing, and is reported in the implementation's diagnostics,
  never on the document.
- **Only an image is embedded.** The stored file's media type must be `image/<subtype>` - matched in
  full, not by prefix, because the type is carried into the render and an uploaded attachment's type is
  whatever the uploading client claimed. A file of any other type renders nothing.
- **There is a size ceiling** the deployment sets (the reference implementation's default is 2 MB). A
  file over it renders nothing. The bound is checked against the store's declared length before the
  file is read and again while reading, since a store may report no length or a stale one.
- **An unreadable store** - the file is there but cannot be opened - renders nothing. A print never
  fails on its picture.
- **Every failure above is soft and looks the same from the document**: the image is omitted, the
  render succeeds. The implementation reports the cause in its own diagnostics.
- **A placeholder in `src` obeys the placeholder rules.** `{{document.<Relation>.StoragePath}}` resolves
  as any `{{document.<Relation>.<Field>}}` does - including
  [alternative operands](../versions/1.6.md#alternative-operands-in-a-placeholder) and row scope inside
  a table - and the resolved string is then read as a `src` of its own shape. An unresolved placeholder
  is a blank `src` and renders nothing.
- **The image follows the render's language.** A language folder holds its own template, and each
  template names its own images; a per-language logo is a per-language file. Nothing else about
  [render language](0029-render-language-governs-its-data.md) changes.
- **A versioned copy freezes its image.** Because the bytes are embedded at render time, the copy a
  `function: Snapshot` mints carries the logo as it was when the copy was minted; replacing the file in
  the store changes later renders, never an existing version - which is what an immutable copy means.
- **The current version's text is wrong by omission.** The 1.6 sentence listing the layout tags does
  not name `image`; the Specification text below replaces it.
- **Not part of this proposal:** a role or key that names a logo in the model (`attachments.roles`,
  `kind: image`) - both MUST be rejected as unknown keys, as any unknown key is; a per-document logo
  slot; a fallback or placeholder image; fetching a remote `http(s):` source on the generator's behalf.

## Prior art / workarounds

Before the construct shipped there was no way to print a picture at all. The available workarounds
were all outside the format: a hand-written print endpoint that composed a PDF with the logo and
bypassed the template; an image baked into a custom stylesheet the template could not reference; or
issuing documents on pre-printed letterhead. Every one of them lost the property the template exists
for - one authored document, rendered the same on screen, in the archive and in the mail.

The reference implementation's earlier layout language did carry an `image` tag whose `src` was
emitted verbatim for a downstream renderer to fetch. That is the shape this proposal rules out: a
verbatim reference to a stored file is fetchable only by opening the file store to an unauthenticated
read, and is not fetchable at all on the two render paths that have no viewer.

## Specification text

**Anchor:** Printable documents — the sentence introducing the layout tags is replaced; the subsection
below is added after "Naming the rendered file — `fileName`".

The replaced sentence:

The template is a tree of layout tags (`page`, `header` / `footer`, `section` / `stack`, `row`,
`field`, `text`, `image`, a `table` bound to the items, `total`, `line`, and `if`), with values as
placeholders:

#### Images — `<image src>`

A template places a picture with `image`. Its `src` names a stored file, and the file is read and
embedded when the document is rendered - so the issuer's logo, a stamp or a signature reaches every
rendered copy of the document:

```text
<image src="Templates/Print/logo.png" width="120"/>
<image src="{{document.Signature.StoragePath}}" height="60" align="right"/>
```

The shape of `src` says what it is:

| `src` | resolution |
| --- | --- |
| a bare path | a file in the application's file store, relative to the store's root - on a shared deployment, the tenant's own store |
| a placeholder | resolved by the ordinary placeholder rules, then read as a `src` of the resulting shape - `{{document.<Relation>.StoragePath}}` renders the file of a [`function: Attachment`](#attachments-and-snapshots) row reached through a to-one relation |
| a `scheme:` URI | emitted as written and never read - a `data:` URI already carries the image inline |

`width` / `height` are resize hints; one alone scales the image proportionally, neither sizes it to
its content. `align` applies as on any block. A generated template carries one shared logo slot,
`Templates/Print/logo.png`, in its header - one path for the whole application, since a company has
one logo, not one per document - so branding a deployment is a single upload to the file store, or
a single `doc/Templates/Print/logo.png` shipped with the project. A logo is per-tenant branding, not
record data: the format has no key that names one.

> **Normative.**
> A conforming implementation MUST resolve `src` at render time, within the scope and authorization
> of the caller the render runs for, and MUST embed the file's bytes into the rendered document. The
> rendered document MUST be self-contained: no image in it may be fetched after the render, by any
> party. The three renders of a document - the interactive print, a [versioned copy](#attachments-and-snapshots)
> and the document a [notify block attaches](#the-notify-block--and-attach-print-sending-the-document-itself) -
> MUST embed the same image from the same template.
> `src` MUST be trimmed. A value beginning with a URI scheme (a letter followed by letters, digits,
> `+`, `-`, `.`, up to a colon) MUST be emitted unchanged and MUST NOT be read. Any other value is a
> path relative to the file store's root; a path with a `..` segment MUST be refused.
> Only a file whose media type is `image/<subtype>`, matched in full, may be embedded. An
> implementation MUST bound the size of an embedded file, checking the bound before reading and while
> reading.
> Every failure - an absent or blank `src`, an unresolved placeholder, a missing file, a refused path,
> a file that is not an image, a file over the bound, a store that cannot be read - MUST render
> nothing: no block, no space, no placeholder, and the render MUST succeed as if the tag were absent.
> The cause MAY be reported in the implementation's diagnostics, never on the document.
> Only an absolute measurement is a resize hint; a percentage or fraction weight on an `image` MUST be
> ignored.
> A placeholder in `src` MUST resolve by the same rules as a placeholder in text, including
> alternative operands and row scope.
> The generated scaffold MUST carry the shared logo slot unconditionally; a generator MUST NOT make
> its presence depend on the file existing at generation time.
> The format has no key that names a logo or an image on a model: `attachments.roles` and
> `kind: image` MUST be rejected as unknown.
> A template using `image` rendered by an implementation that predates it MUST NOT fail to parse.

## DSL index

| Construct | What it does |
| --- | --- |
| [print `<image src>`](#images--image-src) | embed a stored file - the tenant's logo, an Attachment row's file, an inline `data:` URI - into a rendered document; a missing image renders nothing |
