---
layout: post
title:  "Version numbers for data structures"
date:   2009-03-03 11:00:00
tags: versionning
language: EN
---
Data structures can evolve with time, either during the development of a project or between versions of the project. It is crucial to know which structure you are dealing with. In order to do that, you should store the version number of the data structure along with your data.

The place where to put the version number highly depends on the type of storage you are using:

- in a SQL database, it can be stored in a table called Metadata, having 2 columns, a key and a value. The version number would be stored in a row of a given key.
- in an XML file, it can be an attribute on the root element.
- in a remote procedure call (RPC: RMI, .Net Remoting...), it can be a getDataVersionNumber() method.
- in a web-service (REST-style web-service, SOAP web-service...), it can be a DataVersionNumber resource which only contains the version number, or a Metadata resource containing the version number as well as other information.
- in a set of data files, it can be a properties file with a data.versionNumber property.
- in a Lucene index, it can be metadata Document with a Field storing the version number.
- etc.

The version number should be composed of 2 numbers:

- a major version number: you should increment this number if, when you update the structure, the compatibility with older versions of your software is no longer guaranteed.
- a minor version number: you should increment this number if, when you update the structure, the compatibility with older versions is kept. This allows new versions to take advantage of structure changes.

Your software should be declared compatible with either a given major version number, or a range of major version numbers. For instance, an application compatible with versions 4 to 6 should be compatible with versions 4.0, 4.12, 5.3, 6.10, etc, but not with 3.5 or 7.12.

You should make sure this is ensured as soon as possible:

- for a web-application connected to a SQL database: at startup time.
- for XML files: as soon as the file is read.
- etc.

Also, you should make sure both major and minor version numbers are converted as integers before making any comparison (you should not compare the strings "1.9" and "1.10", or convert the values to floats, both methods being incorrect).

Finally, don't make any confusion, for instance:

- with a data version number: the data version number changes when users change data, whereas the data structure version number changes when developers change the data storage structure.
- with a product version number: your product may offer great new features, justifying a change from version 1 to 2, and in the same time, your data structure version number may not change at all.