OEP comments: !!!

**Summary:**

Separate public and internal classes/interfaces/enums on a package level to achieve a cleaner API. Move everything that is not a part of the public API to internal packages. An internal package is a package which name contains ".internal." infix compared to its public counterpart, if any.

**Goals:**

* Obvious separation between public and internal APIs on a per class/interface/enum basis.

**Non-Goals:**

* Packaging reorganization for the existing public APIs.
* Remove, deprecate or lower visibility of anything.
* Add API annotations/javadocs to existing code base.

**Success metrics:**

* Public-internal separation clearly visible to everyone.
* Public things sit in their unchanged public packages.
* Internal things sit in their new internal packages.
* Nothing public is broken.

**Motivation:**

It's really hard to understand what is considered public and what is not in the current ODB code base. This is bad. ODB developers may break public things unexpectedly. Developers of the ODB-based solutions may introduce dependencies on things that are considered internal by ODB developers.

**Description:**

Package-based separation will provide clean indication of a public API. For users, if you don't see "internal" in a package name, it's public, you may use anything from it safely. If you see "internal" in the package name, you still may use anything from it, but at your own risk. Java API documentation should be updated to mention this fact.

For ODB contributors, if you see "internal" in a package name, you may change anything without a fear of breaking things. If you don't see "internal", be careful while changing something, it's a public API. Think twice before removing, changing or deprecating something, think triple before adding somehthing new, we have to support it.

For each Java file in the ODB code base following steps should be taken:

1. Decide is it public or not.
2. If it's public, skip it.
3. Create a new internal package to host it, if package doesn't exist.
4. Move it to the new package.

For consistency, we must choose only one internal package naming scheme:

1. $project.$module.internal.$subpackage: `com.orientechnologies.orient.etl.internal.source`, `com.orientechnologies.orient.core.internal.command.script`, `com.orientechnologies.lucene.internal`.

2. $project.$module.$subpackage.internal: `com.orientechnologies.orient.etl.source.internal`, `com.orientechnologies.orient.core.command.script.internal`, `com.orientechnologies.lucene.internal`.

3. $project.internal.$module.$subpackage: `com.orientechnologies.orient.internal.etl.source`, `com.orientechnologies.orient.internal.core.command.script`, `com.orientechnologies.internal.lucene`.

**Alternatives:**

* Leave is as it is.
* Apply API annotations/javadocs, leave packages alone.
* Declare internal things as package-private. May introduce problems with visibility/inheritance/testability.

**Risks and assumptions:**

* No one may really know what is public/internal at this point.
* Too many ODB users may already have dependencies on internal things.
* Too much team communication may be required to establish the separation.
* Too many classes/interfaces may have a mixture of public-internal methods.

**Impact matrix**

- [ ] Storage engine
- [ ] SQL
- [ ] Protocols
- [ ] Indexes
- [ ] Console
- [x] Java API
- [ ] Geospatial
- [ ] Lucene
- [ ] Security
- [ ] Hooks
- [ ] EE
