# ADIF 4 - A Proposal

By Sebastián Delmont • KI2D • <ki2d@ham2k.com>

# TL;DR

### The problem:

* Users want to use "extended characters" in their logs, and exchange the with other applications, services and users. The standard does not allow it, but many applications do this in many incompatible ways.
* ADIF is "text based", but does not specify the encoding to use in this "text", since this didn't matter for 7-bit ASCII.
* ADIF uses "data lengths" to determine the amount of the data in each field, but does not specify whether these should be counted in bytes or characters, since this didn't matter for US-ASCII.
* [Developers have used both](survey-results/README.md) 8-bit encodings and UTF-8 in their applications, and while both are equivalent for 7-bit ASCII, once you add "extended characters", the two encodings are not equivalent, and applications can break not only by showing the wrong characters, but by truncating data, including extraneous data, or even dropping fields or entire records.
* ***So, we have, at the very least, three different ways to interpret the standard, based on which encoding you pick: ASCII, ISO-8859-1 and UTF-8.*** And these are incompatible with each other if you want to consider including "extended characters" in the standard.

### The proposal:

* Produce a [new "epoch"](https://www.adif.org/316/ADIF_316.htm#Header_Field_ADIF_VER) of the standard: **ADIF 4**.
* Make all ambiguous details regarding encodings in the standard more explicit, including that data lengths should be counted in characters as defined by the encoding used in the file.
* Define a new `ENCODING` header so that applications can explicitly state which encoding they are using. Any encoding [identified by IANA](https://www.iana.org/assignments/character-sets/character-sets.xhtml) can be used, as long as it is a superset of US-ASCII.
* Declare "US-ASCII" as the default encoding, and require applications to support importing and exporting it in order to be considered compliant.
* Declare that applications can reject encodings they don't support, and still be considered compliant, but explain to the user that the rejection was due to an unexpected encoding, provide a list of supported encodings, and point the user to the page at https://tools.adif.org/, which provides users with tools to fix encoding issues and convert between encodings.
* Suggest that applications should support "UTF-8" imports and exports.
* Suggest that applications should support "ISO-8859-1" imports, and might even consider this the default encoding when importing pre-ADIF 4 files.
* Allow applications to attempt to detect, and correct, possible mis-encodings in files that do not have an "encoding" header.

### More context:

* There's a [summary of other proposals discussed](#other-proposals) at the end of this document.
* There's a [survey of existing applicationss](survey-results/README.md) that were tested to see how they handled different types of encodings and field data lengths.

---

# Why do we need to talk about changes to the standard?

The [ADIF Standard](https://www.adif.org/316/ADIF_316.htm) was defined from the very beginning as a ["text format"](https://www.adif.org/316/ADIF_316.htm#:~:text=ADI%20files%20are%20text%20files). It mentions line breaks, 7-bit ASCII, and the use of ASCII digits for any numeric values.

The standard does explicitly state that `String` fields should be limited to ["ASCII [...] in the range of 32 through 126"](https://www.adif.org/316/ADIF_316.htm#:~:text=an%20ASCII%20character%20whose%20code%20lies%20in%20the%20range%20of%2032%20through%20126%2C%20inclusive). This is also known officially as "US-ASCII".

However, in the real world, most applications alllowed users to enter "extended characters", such as accented letters like the "ñ" in "Muñoz", the "ü" in "Müller" or the "ß" in "Straße".

And many applications include these "extended characters" in the ADIF files they export. This is clearly not compliant with the standard, but definitely prevalent (see [survey results](survey-results/README.md)).

There is a problem with this: once you get past the basic english alphabet and into "extended characters", there is no such thing as a "text file" anymore. The text has to be characterized by the encoding used to convert the characters to bytes.

> There are two popular encodings: "ISO-8859-1" (actually, to be precise, most of the time this is interpreted to mean "Windows-1252", which is a superset of "ISO-8859-1" but not an official standard, yet it is broadly used) which encodes the most common characters in what is known as the "Latin alphabet", and "UTF-8" which encodes all characters in the Unicode standard.
>
> There are other useful encodings, including other 8-bit code page encodings in the ISO 8859 family, covering alphabets like greek, cyrillic and more, or Shift-JIS for Japanese characters.
>
> But for simplicity, we will keep the discussion around the main two, and use the terms "ISO-8859-1" and "UTF-8" to refer to each one of them.

Both "ISO-8859-1" and "UTF-8" are supersets of "US-ASCII" (plain english alphabet, numbers, punctuation, etc.) so either is perfectly compatible with the ADIF standard, strictly speaking.

But the two encodings are not compatible once you consider "extended characters". More importantly, **"ISO-8859-1" assumes that each character is encoded as a single byte**, while **"UTF-8" uses multi-byte sequences** to represent characters outside of "US-ASCII".

Now, the ADIF standard uses "data length" counts to determine the amount of data in each field. **But the standard never defined whether these lengths should count bytes or characters**. For "US-ASCII", the two are equivalent, but for "extended characters", they are not.

And developers have implemented their applications using logic that assumes either encoding, [with major applications on either side of the fence](survey-results/README.md). Most likely this was the result of just using the "text primitives" available in their programming language or environment of choice. Old Win32 applications would have used the Windows API, which uses "Windows-1252" by default, while new .NET applications would have used the .NET Framework, which uses "UTF-8" by default. Newer languages like Java, Python and Javascript all use Unicode internally too.

So we have a "Schrödinger's Standard", that can be either US-ASCII, ISO-8859-1 or UTF-8 depending on who you ask, and you cannot really tell which it is until you bring extended characters into the mix.

This has caused extended discussions in the ADIF community, and has led to a lot of confusion and frustration. With different parties arguing that their interpretation of "what text is" is the correct one.

This proposal starts from the premise that all three encodings (and even any other standard encoding)should be considered valid, but that not everybody has to be required to support all three.

---

# What is the proposal?

First, **we should make this a new major version of the standard, ADIF 4**. In order to make it easier to communicate with users, and between developers, that from this clearly marked point on we are talking about a standard that supports multiple encodings and thus requires specificity about which encoding is being used.

Second, we should make all ambiguous points in the standard explicit. There's a section bellow that lists the details. This means **reiterating that the standard is "text based"**, with the text using the encoding specified in an "encoding" header, and that **data lengths should be in "characters"** as defined by the encoding used in the file.

> One point of extensive discussion has been that we should allow developers to use "bytes" as the unit of measurement for data lengths, and use this for UTF-8 encoded files.
>
> Our counterargument is that if you mix UTF-8 with byte encodings, you end up with something that no longer is "text based" can be considered a binary format, because implementing this requires applications to open files as binary data in order to read bytes and not characters. This means the "encoding" applies only to certain fields and not the whole file.
>
> Another point that can be made is that this is not UTF-8 with byte encodings, but rather ISO-8859-1 text with character data lengths, where the strings include characters that happen to match UTF-8 sequences. Something known as ["mojibake"](https://en.wikipedia.org/wiki/Mojibake).

Third, to ensure backwards compatibility, we should declare that **"US-ASCII" is the default encoding**, and require applications to support importing and exporting it in order to be considered "compliant". This is because this is what the current standard defines; and the most prevalent scenario, by far, in the real world is that regardless of which encoding a developer used, the data is still limited to US-ASCII.

Fourth, we should suggest that **applications should support "UTF-8" imports and exports**. This broadens "extended character" support to the entire Unicode standard, and not just a small set like ISO-8859-1 or Greek. New applications, and updated applications, should aim to support UTF-8 as their main encoding for interoperability purposes. For reference, UTF-8 is used by [98.8% of the world's web sites](https://w3techs.com/technologies/cross/character_encoding/ranking).

Fifth, while it is desirable, it is not reasonable to demand all developers to update their applications to support Unicode and UTF-8. Therefore, we should **allow applications to reject encodings they don't support, and still be considered "compliant"**.

At the same time, we don't want to leave users stranded, so the standard should provide a page at https://tools.adif.org/, that applications can refer their users to, which provides information and tools to help users address encoding issues and incompatible applications.

Sixth, we should suggest that **applications should support "ISO-8859-1" imports and exports**, and might even **consider looking for ISO-8859-1 characters mis-encoded in US-ASCII** files without a header. Many existing applications use ISO-8859-1 as their internal encoding, it's mostly backwards-compatible with applications that can deal with US-ASCII, and it's the most prevalent encoding currently used for "extended characters" in the real world of amateur radio. So allowing applications to do this to help users keep their data exchanges working the way they are today, even if this is not fully compliant with the existing standard. I believe it is a reasonable compromise to minimize impact on end users.

It **would not be unreasonable to actually declare "ISO-8859-1" as the official default encoding** for pre-ADIF 4 as long as we're ok with the implication that such a change would **make the new standard not be strictly backwards-compatible**. This would increase the likelihood, but not ensure, of a file exported from an existing application being interpreted correctly by an encoding-aware application. If we prefer to keep the standard strictly backwards-compatible, we can just leave it as a suggestion and keep US-ASCII as the default encoding.

And Seventh, in order to ensure full backwards compatibility, and prevent existing compliant applications from misinterpreting a file encoded as UTF-8, but opened by an application that assumes that characters are bytes, **any encoding that uses more than one byte per character** (such as UTF-8) should perform one extra step, of **escaping any "<" character using the sequence "&lt;"** when exporting ADIF data, and unescaping it when importing such ADIF data. This prevents any application from mistaking unexpected data in a field for the start of the next field.

---

# What does this mean for existing applications?

**Most applications can become fully compliant with ADIF 4 just by adding an "encoding" header to their output files.**

**Any currently-compliant applications can remain unchanged and be able to interoperate with applications updated to ADIF 4** in all cases that were valid when the application was written. These applications should not miss any records or fields, and should be able to display all data correctly, **as long as this data is limited to US-ASCII**.

**These applications can also remain unchanged and still be able to interoperate, with minimal data loss on non-mandatory fields with string data containing extended characters, when importing from ADIF 4 files with encodings other than US-ASCII.** These applications would possibly display a truncated version of these strings, and/or replace any non-US-ASCII characters with "?" or similar. Since the standard has never specified what is the behavior of applications in the case of unexpected data in strings, this is not considered as breaking backwards compatibility, in the same way that an older application not displaying a newly defined enum value is not considered as breaking backwards compatibility.

**Most of these applications can also remain unchanged and still be able to interoperate with _most_ cases that are _prevalent_ in real world usage.** In many cases they can exchange data containing extended characters with other applications that happen to use the same internal encodings, **even if this is a non-compliant scenario.**

Most applications can improve their support of ADIF4 by looking at the "encoding" header in the input file, and if it's not supported, they can offer their users a link to https://tools.adif.org/.

Most applications can achieve full compliance by offering their users different encoding options, including at the very minimum US-ASCII.

Most applications can improve their support for ADIF 4 by using mechanisms to transcode input files to whatever encoding they support internally. We intend to [provide tools to help with this](https://github.com/ham2k/transadif/), and the standard might provide guidance on this respect.

Most applications can improve their support even further by offering their users multiple encoding options when exporting, ideally including UTF-8 and ISO-8859-1 at the very least.

----

# Detailed changes to the standard

### New `ENCODING` header

Add a new header, `ENCODING`, where the generating application can specify any text encoding, using the names or aliases defined by IANA at https://www.iana.org/assignments/character-sets/character-sets.xhtml as long as such encoding is a superset of US-ASCII.

In the absence of an `ENCODING` header, it is assumed that the file is encoded as US-ASCII.

### Change the types of fields that can contain Unicode

Change the types of any field that currently has a matching `_INTL` field, from "String" to "IntlString", or "MultilineString" to "InltMultilineString" as appropriate. Also include a few fields that never got an _INTL counterpart but should have: MORSE_KEY_INFO, QSLMSG_RCVD, QSL_VIA.

Declare `_INTL` fields as deprecated.

Change the definition of "IntlCharacter" to remove `...  (encoded with UTF-8) ...`

Change the definition of "IntlString" (and "IntlStringMultiline" in a similar way) to:

> A sequence of IntlCharacter, encoded according to the encoding specified in the ENCODING header, or US-ASCII if no such header is present.
>
> If the encoding requires multiple bytes to represent a character, and the data contains such a multi-byte sequence, then it must perform the following transformations in this order: On export, any occurrence of the US-ASCII character "&" must be replaced with "&amp;" and then any occurrence of the character "<" must be replaced with "&lt;". On import, any occurrence of the character sequence "&lt;" must be replaced with "<" and any occurence of the character sequence "&amp;" must be replaced with "&". These transformed sequences would be counted as part of the data length, so "á<é" would be transformed to "á&lt;é" and have a field length of 6 characters. But "a<e" would not have any transformations applied and have a field length of 3 characters.

### Clarify field data length

Change section "IV.A.1. ADI Data-Specifiers" so that

> ... followed by data D of length L: ...

is updated to read

> ... followed by data D consisting of L characters as defined by the file encoding specified in the ENCODING header, or as defined by US-ASCII if no ENCODING header is present.

### Clarify file encodings and handling

Change section "IV.A. ADI File Format" so that where it currently says:

> ADI files are text files that are typically exported with a .adi file name extension. Applications should additionally accept files with a .adif file name extension.

it should now say:

> ADI files are text files, encoded according to the encoding specified in the ENCODING header, or US-ASCII if no such header is present.
>
>Compliant applications must accept files encoded as US-ASCII. Applications should also accept files encoded as UTF-8.
>
>An application might reject a file encoded with an encoding it does not recognize, but in that case it should display a message to the user indicating that the encoding was not recognized, listing what encodings are acceptable by the application, and pointing the user at https://tools.adif.org/" for more information.
>
>An application might attempt to correct mis-encoded data when appropriate, such as detecting ISO-8859-1 or UTF-8 sequences in a file supposed to be US-ASCII.
>
>ADI files are typically exported with a .adi file name extension. Applications should additionally accept files with a .adif file name extension.



---

# Other Proposals

Note: This list is comprehensive, but certainly not exhaustive. It represents the interpretation and opinions of the author of this document,Sebastian KI2D. Feel free to follow the links to the original discussions. And of course, feel free to contribute any additional information or comments you consider relevant.

It is our understanding that, besides the ADIF 4 proposal in this document, this short list are the proposals still in consideration:
* [New ADIF container: ADU](#new-adif-container-adu)
* [Use original fields, with HTML entity encoding](#use-original-fields-with-html-entity-encoding)
* [Replacement encoding, using the data between fields](#replacement-encoding-using-the-data-between-fields)

The other proposals all were determined to be incompatible, abandoned, or superceded by other proposals. But reading through their evolution helps understand how we got to the current set being discussed.

### Encoding headers, with data lengths in bytes, and ISO-8859-1 as the default encoding

This was the initial proposal by Sebastian KI2D [in this discussion](https://groups.io/g/adifdev/message/10315)

It would look like `<NAME:10>Sebastián <NOTES:36>이건 예시예요` encoded in UTF-8 (note the `10` and `36` because of multi-byte UTF-8 sequences).

Or like `<NAME:9>Sebastián <NOTES:7>?? ????` encoded in ISO-8859-1 (note the `9`).

The discussion started around how it would be hard for users manually editing a file to properly count the bytes in a UTF-8 string.

This made us realize that indeed, the unit of counting was not necesarily bytes, and that an existing compliant application might misinterpret a field data length for an UTF-8 string and read fewer bytes than expected, and then if the subsequent bytes contained something that looks like a field, such application would potentially skip the field or even the entire record. But also an existing compliant application might use Unicode text primitives to read the data and then misinterpret the field data length as characters and read more characters than the original file intended, thus potentially causing the next field to be skipped or the entire record to be misinterpreted.

This discussison made Thomas VA2NW and Sebastian KI2D start a survey of existing applications to test how they handled different types of encodings and field data lengths. These [survey results](survey-results/README.md) pointed out that there were significant numbers of existing, popular applications on both camps, and that the original assumption that "ISO-8859-1 is the most common implementation in the real world" was not supported by the data.

We also realized that implementing a format with data lengths in bytes in applications that use Unicode primitives would require them to rebuild their entire parsing logic to use raw bytes, and convert certain strings in certain ways, which in turn would mean the ADIF format is not "a text file" but rather a binary format.

Conclusion:
* Breaks backwards compatibility.
* Implementation in Unicode applications would be more complex than currently.

Learnings:
* The ambiguity in the standard created a problem where applications implemented it using two mutually-incompatible interpretations. There is no way to solve the issue of Unicode support without accepting both interpretations as valid.


### Encoding headers, with data lengths in bytes, but default to UTF-8

Proposed by Greg K9CTS [in this discussion](https://groups.io/g/adifdev/message/10322) on Sep 14, 2025.

While this suffered from the same problems as the previous proposal, it showcased the need to focus on Unicode support as the end goal.

Conclusion:
* Same as above.


### Change main encoding to ISO-8859-1, use separate _INTL fields with UTF-8 and data lengths in bytes

Proposed by Dave AA6YQ [in this discussion](https://groups.io/g/adifdev/message/10332) on Sep 14, 2025.

It would look like `<NAME:9>Sebastián <NAME_INTL:10>Sebastián <NOTES:7>?? ???? <NOTES_INTL:36>이건 예시예요` encoded in ISO-8859-1.

Having a new field to hold the Unicode data means that the problem of providing Unicode support is only solved when a significant number of applications are updated to support it, and until then data loss would be likely. Applications might not prioritize implementation until most other applications have also done so, which means we have a chicken-and-egg problem.

Also, having two fields introduces a problem of vague semantics. What happens if only the _INTL field is present? What happens if an application allows the user to update `NAME` but just passes `NAME_INTL` through?

And finally, it turns the standard from "text based" to a "binary format" where different parts of the file use different encodings.

Conclusion:
* Would work, but only after most applications are updated to support it.
* Causes data loss when doing round-trips with existing applications.

Learnings:
* Solutions that use separate fields are bound to suffer from round-trip problems.


### Use original fields, with HTML entity encoding

Proposed by Sebastian KI2D [in this discussion](https://groups.io/g/adifdev/message/10393) on Sep 15, 2025, and later clarified and proposed again [in this discussion](https://groups.io/g/adifdev/message/10872) on Sep 28, 2025.

It would look like `<NAME:16>Sebasti&aacute;n <NOTES:49>&#xC774;&#xAC74; &#xC608;&#xC2DC;&#xC608;&#xC694;` encoded in US-ASCII.

By using HTML entity encoding (formally, ["character reference encoding"](https://en.wikipedia.org/wiki/Character_encodings_in_HTML#Character_references)) we ensure that an ADIF file remains limited to US-ASCII characters, but provide a way to encode and decode any Unicode character.

Implementation would require some changes, but uses an encoding mechanism that has broad support in the industry (it's used by XML, HTML and more) and for which there are well-tested libraries available on every language, framework and platform.

In this discussion, and other threads, some people, starting with Jack W6FB in [this message](https://groups.io/g/adifdev/message/10416), expressed the opinion that an existing application displaying something like "Sebasti&aacute;n" could be considered as breaking backwards compatibility.

Conclusion:
* Works, but requires broad adoption.
* No data loss when doing round-trips with existing applications.

Learnings:
* Some people consider extraneous characters in existing applications to be a backwards compatibility issue.


### Use separate _INTL fields, with Base64 encoding

There was no explicit proposal for this, but Thomas VA2NW suggested using base64 in [this message](https://groups.io/g/adifdev/message/10433) and was later discussed that this would only be acceptable if used on separate fields.

It would look like `<NAME:9>Sebasti?n <NAME_INTL:16>U2ViYXN0acOhbg== <NOTES:7>?? ???? <NOTES_INTL:28>7J2E6rG0IOyYiOyEnOyYieyWtA==` encoded in US-ASCII.

Conclusion:
* Same problems as other _INTL proposals.


### Use separate _INTL fields, with HTML entity encoding

This was mentioned by Sebastian KI2D as an alternative to base64, potentially providing more legibility in cases where the data is limited to ISO-8859-1.

It would look like `<NAME:9>Sebastián <NAME_INTL:16>Sebasti&aacute;n <NOTES:7>?? ???? <NOTES_INTL:49>&#xC774;&#xAC74; &#xC608;&#xC2DC;&#xC608;&#xC694;` encoded in ISO-8859-1.

Conclusion:
* Same problems as other _INTL proposals.


### Encoding headers, only allow ISO 8859 encodings

Proposed by Dave AA6YQ [in this discussion](https://groups.io/g/adifdev/message/10730) on Sep 24, 2025. Dave also suggested it for a vote [in this message](https://groups.io/g/adifdev/topic/request_for_vote/115441785) a day later. No vote happened.

It would look like `<NAME:9>Sebastián <NOTES:7>?? ????` encoded in ISO-8859-1 (note the `9`). It would not be able to encode `이건 예시예요`.

This proposal is similar to the original proposal, but tries to sidestep the bytes-vs-character debate by only allowing ISO 8859 encodings.

It would preclude the use of UTF-8 and limit files to use a single code page at a time.

It would also seem to declare that ADIF is not just "a text file" but a "text file in a limited set of 8-bit encodings", thus limiting future options.

Conclusion:
* Breaks backwards compatibility, limits future options, does not solve the problem of extended characters.


### New ADIF container: ADU

Proposed by Thomas VA2NW [in this discussion](https://groups.io/g/adifdev/message/10843) on Sep 28, 2025.

This proposal allows for full Unicode support, but in files that use a new file extension, `.adu`.

It would look like `<NAME:9>Sebastián <NOTES:7>이건 예시예요` encoded in UTF-8.

It keeps the same base format, field definitions, etc, which makes implementation a lot easier.

Its use of a different container and file extension prevents any compatibility issues with existing applications.

But even if it's easy to implement, it requires applications to somehow present the choice of container to their users. And these users don't necesarily have the context to make the decision.

Users can still rename files between .adi and .adu and things would mostly work, but that requires manual intervention, and understanding what is going on.

Conclusion:
* Works, but might limit adoption and cause some confusion.

Learnings:
* Maybe it makes sense to make these proposed changes very explicit, such as the new extension proposed here, so that users are more aware of the fact that they are using a new format. This is what led Sebastian KI2D to include the idea of changing the epoch and call it ADIF 4, as a "softer hint that things have changed".


### Replacement encoding, using the data between fields

Also known as "Plan 10 for Inner Space", proposed by Graham G3ZOD [in this discussion](https://groups.io/g/adifdev/message/10949) on Sep 30, 2025.

Trying to find a different path, Graham proposed an innovative approach, using the extra data between fields, which the standard allows, to include replacement data for a simplified string.

It would look like `<NAME:9>Sebasti?n {{7=225}} <NOTES:7>?? ???? {{0=51060,1=44148,3=50696,4=49884,5=50696,6=50836}}`

There were several other variants discussed, where the values in `{{...}}` could include the actual UTF-8 sequence, or the replacements could be represented as single UTF-8 characters, etc.

Sebastian KI2D also pointed out that creating a novel encoding is not a trivial task, that it would be likely to introduce bugs (someone pointed out that you could insert a null byte with something like `{{3=0,...}}` and potentially break parsing, for example). And that using an existing encoding like Base64 or HTML entities would be a better choice. There was no agreement on this.

But it was pointed out that ultimately, it suffers from the same problem that the "_INTL Field" proposals have: data is lost when doing a round trip in an existing application.

Also, we could not reach agreement on whether "Sebasti&aacute;n" or "Sebasti?n" or even "Sebastian" were acceptable replacements for "Sebastián". But perhaps the distinction is clearer with "???????" being unacceptable replacement for "이건 예시예요".

Conclusion:
* Works, but only if everybody adopts it, and implementation is not trivial. Cannot do round-trip interoperation with existing applications.

Learnings:
* It was in this discussion that we realized that the data between fields could be used, as long as we did not include a "<" character. This is where the idea of using &lt; to escape UTF-8 strings and prevent breakage first popped up (see [this message](https://groups.io/g/adifdev/message/11113)) leading to the final proposal.
* It was also in the discussion around what was an acceptable replacement that Sebastian KI2D first noted that any string with extended characters was a "new type of value" being introduced in a new version of the standard, and thus how an existing application represented it, or even failure to represent it, was not a backwards compatibility issue, in the same way that an older application failing to dislay `<MODE:4>MFSK<SUBMODE:3>FT4` as "FT4" was not a backwards compatibility issue.


