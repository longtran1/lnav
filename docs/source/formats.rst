
.. _log-formats:

Log Formats
===========

Log files are parsed based on formats defined in configuration files.  Many
formats are already built in to the **lnav** binary and you can define your own
using a JSON file.  When loading files, each format is checked to see if it can
parse the first few lines in the file.  Once a match is found, that format will
be considered that files format and used to parse the remaining lines in the
file.  If no match is found, the file is considered to be plain text and can
be viewed in the "text" view that is accessed with the **t** key.

The following log formats are built into **lnav**:

.. csv-table::
   :header: "Name", "Table Name", "Description"
   :widths: 8 5 20
   :file: format-table.csv

Defining a New Format
---------------------

New log formats can be defined by placing JSON configuration files in
subdirectories of the :file:`~/.lnav/formats/` directory.  The directories and
files can be named anything you like, but they must have the '.json' suffix.  A
sample file containing the builtin configuration will be written to this
directory when **lnav** starts up.  You can consult that file when writing your
own formats or if you need to modify existing ones.

The contents of the format configuration should be a JSON object with a field
for each format defined by the file.  Each field name should be the symbolic
name of the format.  This value will also be used as the SQL table name for
the log.  The value for each field should be another object with the following
fields:

  :title: The short and human-readable name for the format.
  :description: A longer description of the format.
  :url: A URL to the definition of the format.

  :file-pattern: A regular expression used to match log file paths.  Typically,
    every file format will be tried during the detection process.  This field
    can be used to limit which files a format is applied to in case there is
    a potential for conflicts.

  :regex: This object contains sub-objects that describe the message formats
    to match in a plain log file.  Log files that contain JSON messages should
    not specify this field.

    :pattern: The regular expression that should be used to match log messages.
      The `PCRE <http://www.pcre.org>`_ library is used by **lnav** to do all
      regular expression matching.

  :json: True if each log line is JSON-encoded.

  :line-format: An array that specifies the text format for JSON-encoded
    log messages.  Log files that are JSON-encoded will have each message
    converted from the raw JSON encoding into this format.  Each element
    is either an object that defines which fields should be inserted into
    the final message string and or a string constant that should be
    inserted.  For example, the following configuration will tranform each
    log message object into a string that contains the timestamp, followed
    by a space, and then the message body::

    [ { "field": "ts" }, " ", { "field": "msg" } ]

    :field: The name of the message field that should be inserted at this
      point in the message.
    :default-value: The default value to use if the field could not be found
      in the current log message.  The built-in default is "-".

  :timestamp-field: The name of the field that contains the log message
    timestamp.  Defaults to "timestamp".

  :level-field: The name of the regex capture group that contains the log
    message level.  Defaults to "level".

  :body-field: The name of the field that contains the main body of the
    message.  Defaults to "body".

  :level: A mapping of error levels to regular expressions.  During scanning
    the contents of the capture group specified by *level-field* will be
    checked against each of these regexes.  Once a match is found, the log
    message level will set to the corresponding level.  The available levels,
    in order of severity, are: **fatal**, **critical**, **error**,
    **warning**, **info**, **debug**, **trace**.

  :value: This object contains the definitions for the values captured by the
    regexes.

    :kind: The type of data that was captured **string**, **integer**,
      **float**, **json**, **quoted**.
    :collate: The collation function for this value.
    :identifier: A boolean that indicates whether or not this field represents
      an identifier and should be syntax colored.
    :foreign-key: A boolean that indicates that this field is a key and should
      not be graphed.  This should only need to be set for integer fields.

  :sample: A list of objects that contain sample log messages.  All formats
    must include at least one sample and it must be matched by one of the
    included regexes.  Each object must contain the following field:

    :line: The sample message.

Example format::

    {
        "example_log" : {
            "title" : "Example Log Format",
            "description" : "Log format used in the documentation example.",
            "url" : "http://example.com/log-format.html",
            "regex" : {
                "basic" : {
                    "pattern" : "^(?<timestamp>\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}\\.\\d{3}Z)>>(?<level>\\w+)>>(?<component>\\w+)>>(?<body>.*)$"
                }
            },
            "level-field" : "level",
            "level" : {
                "error" : "ERROR",
                "warning" : "WARNING"
            },
            "value" : {
                "component" : {
                    "kind" : "string",
                    "identifier" : true
                }
            },
            "sample" : [
                {
                    "line" : "2011-04-01T15:14:34.203Z>>ERROR>>core>>Shit's on fire yo!"
                }
            ]
        }
    }

Modifying an Existing Format
----------------------------

When loading log formats from files, **lnav** will overlay any new data over
previously loaded data.  This feature allows you to override existing value or
append new ones to the format configurations.  For example, you can separately
add a new regex to the example log format given above by creating another file
with the following contents::

    {
        "example_log" : {
            "regex" : {
                "custom1" : {
                    "pattern" : "^(?<timestamp>\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}\\.\\d{3}Z)<<(?<level>\\w+)--(?<component>\\w+)>>(?<body>.*)$"
                }
            },
            "sample" : [
                {
                    "line" : "2011-04-01T15:14:34.203Z<<ERROR--core>>Shit's on fire yo!"
                }
            ]
        }
    }

Installing Formats
------------------

File formats are loaded from subdirectories in :file:`/etc/lnav/formats` and
:file:`~/.lnav/formats/`.  You can manually create these subdirectories and
copy the format files into there.  Or, you can pass the '-i' option to **lnav**
to automatically install formats from the command-line.  For example::

    $ lnav -i myformat.json
    info: installed: /home/example/.lnav/formats/installed/myformat_log.json

Formats installed using this method will be placed in the :file:`installed`
subdirectory and named based on the first format name found in the file.

Format files can also be made executable by adding a shebang (#!) line to the
top of the file, like so::

    #! /usr/bin/env lnav -i
    {
        "myformat_log" : ...
    }

Executing the format file should then install it automatically::

    $ chmod ugo+rx myformat.json
    $ ./myformat.json
    info: installed: /home/example/.lnav/formats/installed/myformat_log.json
