---
layout: post
title: Scala and Scalding Overview
categories: [userguides, express, 2.0.4]
tags : [express-ug]
version: 2.0.4
order : 4
description: Scala and Scalding Overview.
---

This section walks through KijiExpress examples with the assumption that the reader is not
familiar with Scala or Scalding. If you are already comfortable with the syntax of Scala and
Scalding, you can skip this page.

A typical KijiExpress job includes reading data, manipulating the data structure and perhaps
the data itself, then writing the resulting data. Scala and Scalding are already optimized
to manage data in pipelines; KijiExpress gives you some additional power by taking advantage
of the entity-centric way data is stored in Kiji tables.

After describing some commonly used [Scala syntax conventions](#scala_syntax_conventions), we'll
describe the components of managing basic data flow through a pipeline with KijiExpress:

* [Reading Data](#reading_data)
* [Mapping Data](#mapping_data)

The next section will contain more examples of Scalding operations that are more complex than maps,
such as grouping and joining operations.

## Scala Syntax Conventions

### Newlines and Breaking Lines

Scala statements are read as single lines: they are terminated by newlines or by semi-colons.
A semi-colon in the middle of a line delimits two statements on a single line.
Scala statements can span lines; a newline doesn’t end a statement after a curly brace,
as in a class definition:

    def getMostPopularSong(songs: Seq[FlowCell[String]]): String = {
        val songRecord = songs.getFirstValue
        ...
        return mostPopularSong("song_id").asString
    }

Scala statements cannot include a newline between square brackets.

>Note that more than one newline in a row is not allowed in the Express REPL.

### Declaring Types

In Scala, you set types by following an object by a colon and then the type or trait.
A trait, like a class, lets you define object types, but unlike classes, you can't pass
arguments to a trait. Because everything is an object in Scala, you’ll see typing in all
sort of places. For example, there are two places in this example where types are set
explicitly:

    def getMostPopularSong(songs: Seq[FlowCell[String]]): String = {
        val songRecord = songs.getFirstValue
        ...
        return mostPopularSong("song_id").asString
    }

**Parameter type:**

`getMostPopularSong(songs: Seq[FlowCell[String]])`

The function `getMostPopularSong` takes songs as its input; the syntax `songs:
Seq[FlowCell[String]]` tells you that the next identifier is the type that this
function expects the input songs to be in. In this case, songs is of type
`Seq[FlowCell[String]]`, which is a KijiExpress trait that allows you to easily
handle data from a cell in a Kiji table. More on this later in [Managing
Data](#managing_data).

**Function type:**

`def getMostPopularSong(songs: Seq[FlowCell[String]]): String`

The return type for the function is also explicitly typed: `getMostPopularSong(…): String`
tells you that the return value of this function is of String type.

### Defining Functions

In explicit function definitions in Scala, the syntax of the definition depends on whether
or not the function has a return value.

If the function has a return value, the syntax includes “=”:

    def getMostPopularSong(songs: Seq[FlowCell[String]]): String = {
        ...
    }

In this example, `getMostPopularSong` returns a value of type String that is generated
from the (omitted) statements between curly braces.

If the function does not have a return value (as would be specified by “void” in Java),
the syntax omits the “=”:

    def printFile(fileName: String) {
      val source = scala.io.Source.fromFile(fileName)
      val str = source.mkString()
      println(str)
      source.close()
    }

You’ll also see many places where functions are defined inline in another function call.
For example, when performing a map operation, you can define a function that describes the
transformation between the map source and target fields:

    Need Better Example
    .map('entityId -> 'songId) { (eId: EntityId) => eId(0) }
    Need explanation of function literal.

## Tuples and Pipelines

KijiExpress views a data set as a collection of named tuples. A named tuple can be thought
of as an ordered list where each element has a name. When using KijiExpress with data
stored in a Kiji table, a row from the Kiji table corresponds to a single tuple, where
columns from the Kiji table correspond to named elements in the tuple.

Data processing occurs in pipelines, where the input and output from each pipeline is a
stream of named tuples represented by a data object. Each operation you described in your
KijiExpress program, or job, defines the input, output, and processing for a pipe.

Data enters a pipe from a source. Sources can be such places as Kiji tables, text files, or Sequence
files. At the end of the pipe, tuples are written to a sink. Again, sinks can be Kiji tables, text
files, or SequenceFiles.

### Operating on Pipe Data

When data is read in KijiExpress it is implicitly converted into a KijiPipe, which is an
extension of Scalding RichPipe. The data is in the form of a stream of tuples, where one tuple
includes a row of the input. The data in each tuple is organized in fields: if the data came from a
Kiji table, each tuple contains a field for each column or map family in the table, including the
entity ID. If the data comes from the output of a TextLine source, each tuple contains a single
field named “line”.

One of the syntax constructions you will see frequently in Express samples are invocations
of functions applied to the “pipe” object that is the result of reading data from a source.
Each of the functions map and write in this example operate on an implicit pipe object that
is the result of the TextLine source.

    TextLine("input")
      .map('line ->
          ('userId, 'playTime, 'songId)) { parseJson }
      .map('userId -> 'entityId) { userId: String => EntityId("output")(userId) }
      .write(KijiOutput(args("table-uri"), 'playTime)('songId -> "info:track_plays"))

Scalding pipes operate like composable functions. That is,

    Pipe.functionA().functionB().functionC()

performs

    functionC(functionB(functionA(Pipe)))

where functionA is the first to executed, then functionB, then functionC.

## Reading Data

Most Express operations involve reading data from some data source, whether it's a KijiTable
or a file. In Scala, reading data can be immediately paired with mapping the data into tuples
for data pipeline operations.

A basic statement to read from a text file would use the Scalding type TextLine to handle
the I/O operations:

    TextLine("import-customers-by-state.txt")
        .read

Line-by-line, this statement does the following:

* `TextLine` is a Scadling "source" type. It has one input: the name of the file to treat
as the source. The path to the file is relative to the place the Express job is run.
`TextLine` produces a data stream that has two fields: `line` includes a row of text from
the input file; `num` includes the line number from the file.

* `.read` is an anonymous function operating on the output of `TextLine`; it is on a
separate line by convention. The read method explicitly indicates that the source should
be extracted and streamed into the data pipeline.

    For data in a Kiji table, Express includes the source KijiInput that understands the
    special characteristics of a Kiji table, such as entity IDs.

        KijiInput("kiji/table/uri/users")
            .read

    `KijiInput` is an Express extension of a Scalding source.

* It takes one input, a string that indicates the URI of the Kiji table to treat as the
source. When the Express operation is run as a job, it is typical that this value is
provided on the command line; in that case, the call to `KijiInput` would use an instance
of the Scala Args class to collect the command-line parameters:

        KijiInput(args("users-table"))
        .read

    The command line would include an option `--users-table` where the URI of the Kiji
    table would follow that option, often employing an environment variable (in this case,
    `{KIJI}`) to indicate the location of the table:

        express job <jarfile> <class> --users-table ${KIJI}/users

* It produces a data stream that includes each row in the Kiji table as a tuple; each
column (and map-type column family) appears as an entry in the tuple. Because the source
is a Kiji table, we know that the first entry in the tuple is the entity ID. We also know
that each entry in the tuple potentially includes more than one timestamped version.

## Mapping Data

With data streaming in the processing pipeline, the KijiExpress operation can begin to
manipulate the data. The most common operation to apply to data is to restructure the
arrangement of the data using a map operation. Scala provides more than one map-type operation;
we'll go through the four most common here:

* [Map](#map)
* [FlatMap](#flatmap)
* [MapTo](#mapto)
* [FlatMapTo](#flatmapto)

### Map

Map operations in Scala use the following syntax:

    map('existing_field -> 'additional_field) { mapping_function }

This syntax is simplified: it shows single field mapped one-to-one to another single field.
A map function can operation on any number of fields at once, and the mapping doesn't
have to be one-to-one.

The mapping function can be something defined elsewhere (in an included library or elsewhere
in the file) or written out between the curly braces. The mapping function needs to be told
the types of the input it operates on.

The output of a map operation on the data stream always augments the input stream: the
output is a tuple that includes the entire input stream plus whatever additional target
fields are specified.

>The keyword `map` also appears as a constructor for a map datatype. For example, `Map`
>appears as a KijiInput parameter, it maps columns from the table into tuples in the
>output data stream. See KijiSources ?link.

Here are some examples of map functions:

#### Parsing row content into tuple fields

This example takes the line field (output from `TextLine`) and uses a mapping function
`parseJson` to transform the JSON record into tuple fields.

Notice that the map target includes multiple fields: the `parseJson` function has to
produce these fields as its output.

    .map('line -> ('userId, 'playTime, 'songId)) { parseJson}

The map function does not need to indicate the types of the output fields because they
are specified in the definition of `parseJson`:

    def parseJson(json: String): (String, Long, String) = {
        ...
    }

The anonymous map function augments the input data stream with the new material specified
in the target; each "row" of the the output data stream is a tuple that consists of four
fields: `line`, `userId`, `playTime`, and `songId`.

#### Inserting a value as an Entity ID

This example takes the values of one field, transforms them by applying the mapping
function, then puts the result in another field.

    .map('firstSong -> 'entityId) {
        firstSong: String => EntityId(firstSong) }

This example has the following points:

* Data from one field is mapped to another single field.

* The mapping function specification includes the data type of the source field
(`firstSong: String`).

* The operation that the mapping function performs is defined by EntityId, which is a
KijiExpress method that creates an entity ID from the objects passed to it.
(`EntityId(firstSong)`).

* It augments the input stream so that the output stream includes the entire content of
the input stream augmented with the new field entityId. The result is a stream that's
ready to be written to a Kiji table.

#### Processing Kiji Table Columns

A map statement can include the transformation in line instead of specifying a function
defined elsewhere. This example takes a column from a Kiji table and returns only the most recent
value of the column.  It uses the fact that the content of the column can be manipulated as a
`Seq[FlowCell[...]]`: Kiji tables can hold many timestamped values (versions) for each column; data
is returned in a `Seq[FlowCell[...]]`.  We can access the first element of a Seq with `seq.head`,
and we can access the datum contained in a FlowCell with `flowCell.datum`.

    .map('trackPlays -> 'lastTrackPlayed) {
        slice: Seq[FlowCell[String]] => slice.head.datum}
    .mapTo(('entityId, 'name, 'desc) -> ('id, 'name, 'desc)) {
        cols: (EntityId, Seq[FlowCell[String]], Seq[FlowCell[String]]) =>
          val (entityId, name, desc) = cols
          (entityId(0), name.head.datum, desc.head.datum)

This example has the same basic structure as previous examples:

* Data from one field is mapped to another single field.
* The mapping function includes the data type of the source field.  Note that the parameter name
  `slice` does not need to correspond to the field name that the data comes from, in this case,
  `'trackPlays`.
* It augments the input stream so that the output stream includes the entire content of
  the input stream (`trackPlays`) augmented with the new field (`lastTrackPlayed`).

### FlatMap

FlatMap operations in Scala use the same syntax as map operations:

    .flatMap('existing_field -> 'additional_field) { mapping_function }

Just like a map operation, you can specify multiple existing or additional fields and the
mapping function can be defined in-line or elsewhere in the file or in an included library.

The `flatMap` operation performs the map to generate a list of new values to fill the
additional fields, then flattens that list into individual tuples. For each tuple of input,
the `flatMap` generates a tuple for each value produced by the mapping function. Each output
tuple includes the entire input tuple plus the additional field value.
Here are some examples of `flatMap` functions:

#### Document to Words

The quintessential use of `flatMap` makes an index of values such as turning a document
into a list of words. The `flapMap` statement takes a string, splits it into "words"
delimited by spaces, and then produces a tuple for each word is as follows:

    .flatMap('doc_content -> 'word) {
        doc_content : String => doc_content.split("\s+")
    }

As with a `map` operation, the `flatMap` operation has the following characteristics:

* Data from one field is mapped to another single field.

* The mapping function includes the data type of the source field: `(doc_content : String)`.

* Scala includes a number of methods that can be applied against a string value that are
defined for the Scala class "StringLike". One is `split`; others include `append`, `capitalize`,
`compare`, `count`, `distinct`, `isEmpty`, `last`, `sortBy`, and `toSet`. In this case, the
`split` method takes a character or a regular expression; this example uses the regular expression
`\s+` to specify one or more spaces.

* The mapping function splits the input into words; if this were a `map` function, the
output would be the input tuple with an additional field that included a list of all the
words in the `doc_content` field of the original tuple. Because it is a `flatMap` function,
the output is a tuple for each word in `doc_content`. Each output tuple contains the fields
from the input tuple with an additional field `word` with one of the words from `doc_content`.
?Is the list ordered in any way?

#### FlatMap With Multiple Fields in the Output

The Document to Words example is simple in that it pivots on a single field value; what
happens when there's more payload in the output?

The following example includes multiple "additional fields" grouped within parentheses:

    .flatMap('playlist -> ('firstSong, 'songId)) { bigrams }

The interesting pieces of this statement are:

* The fields added to the output data stream are grouped in parentheses to indicate that
they together compose the new value that is the seed of the new output tuple.

* The mapping function `bigrams` converts the list of songs from playlist (organized as a
set of versions in a single column of a Kiji table, see `trackPlays` in Processing Kiji
Table Columns) into a collection of tuples, each tuple including two song names of songs
that were played one after the other. The input in this case is a stream of tuples from a
Kiji table. The field `playlist` has been produced as an output of a `KijiInput` source
and the data in it is of type `Seq[FlowCell[String]]`:

![KijiSlice][express_playlist]

[express_playlist]: ../../../../assets/images/express_playlist.png

The mapping function bigram turns playlist into a collection of tuples:

![Bigram][express_bigram]

[express_bigram]: ../../../../assets/images/express_bigram.png

The output of the `flatMap` is a tuple for each value of the collection of first song and
its ID; the tuple includes the original playlist values: ?how do the song IDs come back?

![FlatMap][express_flatmap]

[express_flatmap]: ../../../../assets/images/express_flatmap.png

The details of how to define the mapping function to generate bigrams is described in
?tutorial_link?

### MapTo

MapTo operations perform a map operation followed by a "project" operation; using a
`mapTo` function is more efficient that performing the two operations separately. The
project operation drops the original input fields and retains only the fields indicated
in the map operation target field list.

    .mapTo((existing_field_list) -> (output_field_list)) { mapping_function }

After this oepration, the only fields of the tuples in the pipe are those in output_field_list.


### FlatMapTo

FlatMapTo is similar to MapTo: it performs a flatMap, but keeps only the output fields. This snippet
comes from the PlayCounter in the KijiExpress music tutorial:

    .flatMapTo('playlist -> 'song) { slice : Seq[FlowCell[[String] =>
      slice.cells.map {cell => cell.datum}
    }

After this step in the flow, the only field in the pipe is 'song.

### Other Scalding Operations

In the next section, you will see examples of other Scalding operations that alter the structure of
data, instead of simply being transformative maps on a pipe.  These include grouping and joining
operations.

## Building from Examples

There are a number of examples provided in the KijiExpress tutorial. The following list
indexes some of the functionality shown in the tutorial examples with the specific
example file:

| Functionality | Example |
| ------------- | ------- |
| Pass arguments from the command line | all |
| Read data from a Kiji table | all |
| Call a function to operate on the pipeline data | all |
| Create an EntityId from a column in the source | SongRecommender.scala |
| Read a column with multiple values and produce a tuple | SongRecommender.scala |
| Filters the pipeline input on specific columns (project) | SongRecommender.scala |
| Extract a single data value from a column | SongRecommender.scala |
| Join data from two pipelines | SongRecommender.scala |
| Breaks a list into individual values | SongPlayCounter.scala |
| Groups values, counts instances | SongPlayCounter.scala |
| Create a sorted key-value store | TopNextSongs.scala |
| Fill an Avro record | TopNextSongs.scala |
| Write data to a Kiji table | all |
| Write data to a HFDS file | SongPlayCounter.scala |


These Scala files are here:

    ${KIJI_HOME}/examples/express-music/src/main/scala/org/kiji/express/music/
