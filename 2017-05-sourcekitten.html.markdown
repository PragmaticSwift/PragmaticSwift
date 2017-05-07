---
title: SourceKitten
date: 2017-03-27 20:39 UTC
issue: may-2017
tags:
author: Seán Labastille
author_twitter: flufffel
editor: Patrick Balestra
editor_twitter: balestrapatrick
order: 5
---

# SourceKitten

[SourceKitten](https://github.com/jpsim/SourceKitten) is a command line utility wrapping [SourceKit](https://github.com/apple/swift/tree/master/tools/SourceKit).
SourceKit supports editor functionality to work effectively with Swift such as code completion, indexing or syntax colouring. Xcode's editor relies on SourceKit for this functionality, for example.

SourceKitten enables you to use these abilities yourself by providing high-level commands to extract documentation comments, get structure or syntax information as well as look up code completion suggestions.

Let's walk through these step by step and see how you can work with the results of these commands, using [Alamofire](https://github.com/Alamofire/Alamofire) as a sample project.

## Commands

### Code Completion

In order to look up code completion suggestions you need to provide a source file path as well as an offset within this file for which completions should be looked up for.

For example the following invocation points at the start of a switch case statement over an enumeration. Here the completions are restricted to the relevant enumeration cases.

```swift
                boundaryText = "--\(boundary)\(EncodingCharacters.crlf)"
            case .encapsulated:
                boundaryText = "\(EncodingCharacters.crlf)--\(boundary)\(EncodingCharacters.crlf)"
            case .final:
                  ^
                boundaryText = "\(EncodingCharacters.crlf)--\(boundary)--\(EncodingCharacters.crlf)"
            }
            return boundaryText.data(using: String.Encoding.utf8, allowLossyConversion: false)!
```

```shell
$ sourcekitten complete --file Source/MultipartFormData.swift --offset 3068
```

```json
[{
  "descriptionKey" : "encapsulated",
  "kind" : "source.lang.swift.decl.enumelement",
  "name" : "encapsulated",
  "sourcetext" : "encapsulated",
  "numBytesToErase" : 0,
  "typeName" : "MultipartFormData.BoundaryGenerator.BoundaryType",
  "associatedUSRs" : "s:FOVC17MultipartFormData17MultipartFormData17BoundaryGenerator12BoundaryType12encapsulatedFMS2_S2_",
  "moduleName" : "MultipartFormData",
  "context" : "source.codecompletion.context.exprspecific"
}, {
  "descriptionKey" : "final",
  "kind" : "source.lang.swift.decl.enumelement",
  "name" : "final",
  "sourcetext" : "final",
  "numBytesToErase" : 0,
  "typeName" : "MultipartFormData.BoundaryGenerator.BoundaryType",
  "associatedUSRs" : "s:FOVC17MultipartFormData17MultipartFormData17BoundaryGenerator12BoundaryType5finalFMS2_S2_",
  "moduleName" : "MultipartFormData",
  "context" : "source.codecompletion.context.exprspecific"
}, {
  "descriptionKey" : "initial",
  "kind" : "source.lang.swift.decl.enumelement",
  "name" : "initial",
  "sourcetext" : "initial",
  "numBytesToErase" : 0,
  "typeName" : "MultipartFormData.BoundaryGenerator.BoundaryType",
  "associatedUSRs" : "s:FOVC17MultipartFormData17MultipartFormData17BoundaryGenerator12BoundaryType7initialFMS2_S2_",
  "moduleName" : "MultipartFormData",
  "context" : "source.codecompletion.context.exprspecific"
}]
```

Elsewhere you may receive methods and operators applicable to strings:

```swift
        var headerText = ""

        for (key, value) in bodyPart.headers {
            headerText += "\(key): \(value)\(EncodingCharacters.crlf)"
                      ^
        }
        headerText += EncodingCharacters.crlf
        return headerText.data(using: String.Encoding.utf8, allowLossyConversion: false)!
```

```shell
$ sourcekitten complete --file Source/MultipartFormData.swift --offset 18149
```

```json
["...", {
  "descriptionKey" : "+=",
  "kind" : "source.lang.swift.decl.function.operator.infix",
  "name" : "+=",
  "sourcetext" : "+= <#T##String#>",
  "numBytesToErase" : 0,
  "typeName" : "Void",
  "moduleName" : "Swift",
  "context" : "source.codecompletion.context.othermodule"
},
"..."
]
```

For more details on what kind of contexts code completion operates in you can review [the implementation](https://github.com/apple/swift/blob/master/lib/IDE/CodeCompletion.cpp).

### Syntax

This command simply returns an array of tokens contained in the given file, indicating their type and range (offset and length).

```shell
$ sourcekitten syntax --file Source/Result.swift
```

```json
[
  "...",
  {
    "type" : "source.lang.swift.syntaxtype.identifier",
    "offset" : 3514,
    "length" : 5
  },
  "..."
]
```

For a full list of types this might return, see the [SourceKit source code](https://github.com/apple/swift/blob/master/tools/SourceKit/lib/SwiftLang/SwiftLangSupport.cpp#L402-L419).

### Structure & Documentation

Describing both the structure and documentation for source files is closely related.
The following invocation will return the structure of the given source file.

```shell
$ sourcekitten structure --file Source/Result.swift
```

```json
{
  "key.offset" : 2869,
  "key.nameoffset" : 2873,
  "key.accessibility" : "source.lang.swift.accessibility.public",
  "key.typename" : "String",
  "key.length" : 23,
  "key.name" : "description",
  "key.bodyoffset" : 2894,
  "key.kind" : "source.lang.swift.decl.var.instance",
  "key.namelength" : 11,
  "key.bodylength" : 141
},
```

Meanwhile, in order to extract documentation SourceKitten will invoke `xcodebuild` under the hood, so you should run it in a folder containing an Xcode project.
If you're just experimenting it can be helpful to create a small sample project rather than waiting for a full build of your production app.

The simplest invocation of the documentation command will yield documentation for each source file in the project.
The output below is a snippet focused on the same source code as the structure extracted above.
Notice how the basic attributes are shared, but plenty of additional keys have been added such as various views of the documentation comments.

Another key benefit of compiling as part of generating documentation is receiving accurate type information or identifiers for entities like _Unified Symbol Resolution_ (USR) (See the [Swift lexicon](https://raw.githubusercontent.com/apple/swift/7e68e02b4eaa1cf44037a383129cbef60ea55d67/docs/Lexicon.rst)).


```shell
$ sourcekitten doc
```

```json
{
  "key.accessibility" : "source.lang.swift.accessibility.public",
  "key.length" : 23,
  "key.overrides" : [
    {
      "key.usr" : "s:vPs23CustomStringConvertible11descriptionSS"
    }
  ],
  "key.doc.type" : "Other",
  "key.parsed_scope.start" : 79,
  "key.kind" : "source.lang.swift.decl.var.instance",
  "key.doc.full_as_xml" : "<Other file=\"./Source/Result.swift\" line=\"79\" column=\"16\"><Name>description</Name><USR>s:vO9Alamofire6Result11descriptionSS</USR><Declaration>public var description: String { get }</Declaration><Abstract><Para>The textual representation used when written to an output stream, which includes whether the result was a success or failure.</Para></Abstract></Other>",
  "key.nameoffset" : 2873,
  "key.typename" : "String",
  "key.doc.column" : 16,
  "key.doc.comment" : "The textual representation used when written to an output stream, which includes whether the result was a\nsuccess or failure.",
  "key.bodyoffset" : 2894,
  "key.filepath" : "./Source/Result.swift",
  "key.doc.file" : "./Source/Result.swift",
  "key.annotated_decl" : "<Declaration>public var description: <Type usr=\"s:SS\">String</Type> { get }</Declaration>",
  "key.fully_annotated_decl" : "<decl.var.instance><syntaxtype.keyword>public</syntaxtype.keyword> <syntaxtype.keyword>var</syntaxtype.keyword> <decl.name>description</decl.name>: <decl.var.type><ref.struct usr=\"s:SS\">String</ref.struct></decl.var.type> { <syntaxtype.keyword>get</syntaxtype.keyword> }</decl.var.instance>",
  "key.doc.declaration" : "public var description: String { get }",
  "key.typeusr" : "_TtSS",
  "key.offset" : 2869,
  "key.doc.line" : 79,
  "key.doc.name" : "description",
  "key.name" : "description",
  "key.usr" : "s:vO9Alamofire6Result11descriptionSS",
  "key.parsed_declaration" : "public var description: String",
  "key.namelength" : 11,
  "key.parsed_scope.end" : 86,
  "key.bodylength" : 141
}
```

If you normally need to pass additional arguments to `xcodebuild` you can also use these with SourceKitten by adding them to the invocation following a double dash.

```shell
$ sourcekitten doc -- -scheme App
```

## Feral SourceKittens: Examples of usage in the wild

### In other projects

A prominent user of SourceKitten is Realm's documentation generator [Jazzy](https://github.com/realm/jazzy) which provides a great way to generate documentation for your codebase.

### In your own projects

A quick way to start using SourceKitten yourself is by combining an invocation with [`jq`](https://stedolan.github.io/jq/) which offers `sed`-like command line processing for JSON.

For example you can compare the relative amount of comments in a source file:

```source
$ sourcekitten syntax --file Source/Result.swift | jq 'map(select(.type != "source.lang.swift.syntaxtype.doccomment")) | length'
239

$sourcekitten syntax --file Source/Result.swift | jq 'map(select(.type == "source.lang.swift.syntaxtype.doccomment")) | length'
78
```

Once you have prototyped an interesting query you could of course use this as part of a more complex workflow, such as the following Ruby snippet below.

```ruby
#!/usr/bin/env ruby

require 'json'

contents = File.read('Source/Result.swift')
syntax = JSON.parse`sourcekitten syntax --file Source/Result.swift`

syntax.each do |token|
  puts "#{token['type']} #{contents.slice(token['offset'], token['length'])}"
end
```

## Natural Predators

While SourceKitten is a great way to gain better insights into your Swift projects today, the Swift team and contributors are also working on providing such support as part of the tooling shipped with the language, under the umbrella of the [Swift Syntax Structured Editing Library](https://lists.swift.org/pipermail/swift-dev/Week-of-Mon-20170206/004066.html) which lives at [`lib/Syntax` in the Swift repository](https://github.com/apple/swift/tree/master/lib/Syntax).
