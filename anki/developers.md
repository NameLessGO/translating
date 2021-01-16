# For Developers

When adding user-visible strings to Anki's codebase, extra work is required
to make the strings translatable.

Anki's codebase has recently migrated away from the old gettext system,
and now only uses Fluent's `col.tr()` and `aqt.utils.tr()`.

Please start by taking a look at the [core documentation](/anki/core.md) to see
how strings are presented to translators. Note how the strings do not
appear in the context of code, so comments are often required to help the
translators understand what they are translating.

## Adding New Strings

As an example, imagine we want to add the string "You have x add-ons".

To add a new translatable string to the codebase, we first need to identify
whether it belongs in the `core` module (text likely to be used by all Anki clients),
or whether it is specific to the computer version (the `desktop` module).
Add-ons are only supported by the computer version, so we'll want to use
the `desktop` module in this example.

- The English `core` files are stored in `ftl/core`
- The English `desktop` files are stored in `ftl/qt`

We'll look for a file like `ftl/qt/addons.ftl`, and add one if no appropriate
one exists. Then we need to add the string to the file.

Each string needs a key that uniquely identifies it. It should start with
the same text as the filename, and then contain at least a few words separated
by hyphens. For example, we might add the following to the file:

```
addons-you-have-count = You have {$count} add-ons.
```

The problem with this is that it will look strange when count is 1. We had
better use a different string in the single add-on case:

```
addons-you-have-count = You have { $count ->
    [one] 1 add-on
   *[other] {$count} add-ons
   }.
```

While an improvement, this can make it a bit hard for translators when the
"you have" part needs to change depending on the number. So while more verbose,
it is better to list out the alternatives in full where possible.

```
addons-you-have-count = { $count ->
    [one] You have 1 add-on.
   *[other] You have {$count} add-ons.
   }
```

This leaves the most control in the hands of the translators to be able
to translate the sentence with a natural structure.

Please note that the bulk of the strings in the .ftl files were imported
from the older gettext system, so some of them may not demonstrate best
practice. The older system also encouraged constructs like:

```
msg = "%s %d %s" % (_("Studied"), card_count, _("today"))
```

Please avoid constructing strings in Python like this, as it makes it hard
for translators to translate, and not all languages use English spaces.
The above example would be better made into a single translatable string
that takes a count argument.

Finally, we should also add one or more comments for translators:

```
### Lines starting with three hashes are shown in the translation system
### for every string in the file.
### You can use it for a general summary of what a file is dealing with. Not
### all files will need this.

## Lines starting with two hashes are shown for every following string, until
## another set of two hashes is encountered, or the file ends.

# This is a comment that will only apply to the following string. Eg:
# Shown in the add-on management screen (Tools>Add-ons), in the title bar.
addons-you-have-count = { $count ->
    [one] You have 1 add-on.
   *[other] You have {$count} add-ons.
   }
```

## Accessing the New String

Once you've added one or more strings to the .ftl files, run Anki in the source
tree as usual, which will compile the new strings and make them accessible in
Python/Typescript. To resolve a string, you can use the following code:

```
from aqt.utils import tr, TR

msg = tr(TR.ADDONS_YOU_HAVE_COUNT, count=3)
```

- Note how a SCREAMING_CASE constant has been automatically defined as part of
  the build process, allowing for type checking.
- You can provide as many arguments as you need, as either strings,
  integers, or floats.
- Code in qt can use the `aqt.utils.tr()` function, but code in pylib should
  use `col.tr()` instead.
- See ts/congrats for how strings can be looked up in Typescript.

If you'd like to test out the strings in the Python repl, make sure to
call set_lang() first.

```
>>> from anki.lang import set_lang
>>> from aqt.utils import tr, TR
>>> set_lang("en", "")
>>> tr(TR.ADDONS_YOU_HAVE_COUNT, count=3)
'You have \u20683\u2069 add-ons.'
```

Note the unicode characters that were inserted. These ensure that when left-to-right
and right-to-left languages are mixed, the text flows correctly. For debugging,
you can strip the characters off:

```
>>> from anki.lang import without_unicode_isolation as nosep
>>> nosep(tr(TR.ADDONS_YOU_HAVE_COUNT, count=3))
'You have 3 add-ons.'
>>> nosep(tr(TR.ADDONS_YOU_HAVE_COUNT, count=1))
'You have 1 add-on.'
```

## Repeated Content

Sometimes you will need multiple translations that have some text shared between
them. For example, the search code has a list of separate error message, such as:

```
search-invalid-added = Invalid search: 'added:' must be followed by a positive number of days.
search-invalid-edited = Invalid search: 'edited:' must be followed by a positive number of days.
```

Since there are a lot of separate messages with the same prefix, we can split it off
into a separate string:

```
search-invalid-search = Invalid search: { $reason }
search-invalid-added = 'added:' must be followed by a positive number reason.
search-invalid-edited = 'edited:' must be followed by a positive number of days.
```

The code can then compose the message from two separate entries. Note that we're
using a Fluent variable in the outer string, as some languages may want to place
the reason first, not incude a space after the colon, and so on.

## Avoid HTML where possible

Translators may not have any development experience, and HTML can be difficult to read
and translate correctly. Prefer plain text where possible, and when HTML is required
or would make things much clearer, consider using markdown instead (see the search
translations for an example of how this is done).

## Avoid Strings That Will Change

Avoid doing things like listing out a series of options in a string, if there's any
chance that list will change in the future. When the string later gets updated
(and assuming people don't forget to update it), it will need to be given a new ID
so that translators become aware of it, and doing so will mean all the existing
translations get invalidated until a translator has a chance to update them, which
may take months and sometimes even years.

## Avoid Excess Strings

Please try to be conservative with the number of new strings you add, as translator time
is precious, and the more strings included in Anki, the more overwhelming it can be
for translators. If you need to display an error message for some obscure error that most
people will never hit, it probably doesn't need a translation.

## Add-ons

Add-ons can make use of existing strings in Anki, but if you wish to add new
strings, you'll need to add them to your add-on rather than Anki. How you approach
translations in your add-on(s) is up to you:

- If you don't care about translations, the simplest solution is to use plain strings.
- The next-easiest option is to place all strings in a dict or python module,
  and accept pull requests from users that add a new language. You can then
  select the appropriate dict by looking at what anki.lang.currentLang is set
  to.
- If you want proper plural support, you'll want to consider using gettext,
  though it is considerably more involved than the above solution. It requires
  separate build steps to extract strings from your source and build compiled
  versions of the strings. You can have translators email you a completed .po
  file, or send you a PR. There are online translation services like Launchpad
  and Crowdin that support gettext, but you would need to either manually upload
  and download files, or spend some time setting up scripts to do so.

Fluent is not currently recommended for add-ons. There is limited tooling
support for it, and you would need to bundle the Fluent Python libraries with
your add-on.
