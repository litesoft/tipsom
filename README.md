# tipsom
ACSON - Access Constrained Simple Object Notation

Why is ACSON needed?

  When you switch from a Relational DB to a Document Based NoSQL DB, there are two significant changes in the data that is read per record.  These two changes are: a significant increase in the size of the data per record, and the data is no longer a single list of fields, but is a DOM (Document Object Model) of multiple records, of various types and quantities.  Quite often the Text DOM is represented in JSON, and the JSON has to be parsed as a unit, which could take a significant amount of time and memory as the parsing keeps generating more and more JSON objects, and if you then bind all these JSON objects to domain objects (generating a Domain Object Graph or DOG) even more time and memory is used.

  These problems with the reading of records, pale when you think about the process of updating a large JSON structure from a series of Restful endpoints that attempt to update a sub-section of the JSON via a sub-section of the DOG being updated: parse all the JSON into a large DOG and then update the large DOG from the requested change via a small DOG and then render the updated large DOG back into JSON.  If your going to use Optimistic Record Locking, when the record (mostly JSON based string) is finally attempted to be written,  after doing all the parsing and rendering, the process can fail.

  IMO, the fundamental problem is having to interact with the JSON DOM as a unit.  So, a potential solution is to convert the single large JSON DOM into a selectively managed set of smaller Text DOMs.  The question then is what is the optimal scope/size of these smaller Text DOMs?

  With ACSON, the plan is to treat every attribute of every Text DOM object as a separately parsable entity.  These entities can be "Lazy Parsed" on demand as sections are actually requested (and extending the DOG as needed), and to be able to quickly stitch together all the parsed and unparsed data back into a single Text DOM.

The goals of ACSON (a modified form of JSON) is to have the simplicity of JSON but to support easy of semi-parsing (very simple and fast metadata parsing and deferred data parsing) which will support very rapid hierarchical access (parsing only enough to satisfy current request).  The semi-parsing that occurs as part of the selective extraction, facilitates quick generation by directly utilizing the unparsed data with simple and quick rendering of the parsed metadata.  An additional goal was that it be completely human readable (which means that there should be no "invisible" characters EXCEPT spaces).

When reading the info below, please keep in mind the cleverness of UTF-8: all 7-bit ASCII character byte values (including line feeds) can NOT exist as part of a multibyte UTF-8 Rune!

To achieve the goals above, the following are some of the plans and limitations needed:

  0) To support rapid parsing, all data is UTF-8 "line" based (using line feeds), so no data may contain an unencoded new-line.  Excepting the new-line(s) and spaces, the human readable goal dictates that NO OTHER ASCII CONTROL CHARACTERS will be allowed (nothing less than an ASCII space - decimal 32, except the line feeds - note, there are other countries character sets they may have invisible characters, but filtering for them is not practical).  The first pass in the parsing process is to convert the "file" byte stream into text lines by looking for the line feeds.

  1) There is an implied "root" "object" that starts the "file".

  2) Each line of the "file" is of one of the following types (the first two line types are identified by the single-quoted starting character):
      '#' - Comment line,
      ':' - Member "Value" (Arrays or MLSs - see below) "prefix", or
      Attribute "Name" & "Value" (Objects)

  3) Like JSON, ACSON is a "Name" - "Value" paired format where all "Values" are of one of the following forms: "Objects", "Arrays", and limited "primitives".

  4) The ACSON primitives include all the JSON primitives (with three variations for strings) plus some temporal types:
      Strings - three variations:
        Multi-Line String (MLS),
        Single-Line Strings - two variations:
          JSON Encoded String (JES), and
          Unencoded ASCII String (UAS)
            To ensure that these UASs can be quickly determined to NOT need any encoding the strings may only contain ASCII values from (inclusive) space (32) thru (inclusive) byte value 126!
      Numbers,
      Booleans, and
      temporal types (see Note - temporal formats):
        Instance - ISO8601plusZuluRelative
        CalendarDate - ISO8601dateOnlyNoTZ
        Time - ISO8601timeOnlyPlus
        Duration - ISO8601likeHoursMins (Human "Daily" Durations)

  5) Since "Strings" could violate the "rules" for parsing, every string added will be evaluated and stored in a combination of the three string variations.  Note: the multi-line string form's subsequent lines consist of the other two forms (in the Array Member "Value" form).  Note: Trailing white space for each line is removed!

  6) Since the hierarchical nature of the ACSON (and JSON for that matter) is created by the use of attribute values that are Objects, Arrays, and MLSs, the key to rapid parsing (and deferring the majority of the detailed parsing) is the easy determination of the length (for skipping parsing) of the Objects, Arrays, and MLSs.  To that end, each "Value" contains a metadata structure that indicates (implicitly or directly) how many additional lines make up the "body" of the Object, Array, or MLS.

  7) All Attribute "Names" will be legitimate Java/Go (and most other languages) variable names, BUT be limited to 7-bit ASCII.  This means that they must start with a Letter and then be optionally followed by: letters, digits, and underscores.  Attribute "Names" are terminated by the "Value's" "prefix".

  8) A "Value" (whether it is a Member "Value" or part of an Attribute's "Name"/"Value" pair) consists of parts (on a single line):
    a) ':' the "Value's" "prefix",
    b) A single character that indicates the "Value's" "form" (see "Strategy dispatch characters"), and
    c) Either:
      1) Actual data of the "Value", or
      2) For multi-line values (Objects, Arrays, & MLSs), an integer indicating the number of additional lines.

  9) Strategy dispatch characters for the various "Value" "forms" (the character is the single letter before the '-', alphabetically ordered for the letters):
    { - Objects,
    [ - Arrays,
    " - JESs,
    ' - UASs,
    b - Booleans,
    c - CalendarDate,
    d - Duration,
    i - Instance,
    n - Numbers,
    s - MLSs, and
    t - Time

  10) The "Zero" value (see Go re null/nil and empty strings) for primitives will NOT be stored with the exception of the empty string in a MLS.

Note - temporal formats ("[...]" indicates that whatever is between the '[' and the ']' is optional, "{...}" indicates a group where one of the options must be used, and "|" is an option separator):

  Since most, if not all, of the forms below will be used for sorting or ordering, there are no leading optional sections and hard limits on some values.

  ISO8601likeHoursMins - used for Duration (Human "Daily")
                  format: "hh:mm"
      notes:
        hh - hours, 2 digits, max: 99 (for situations where people are "working" for multiple days)
        mm - minutes, 2 digits, max: 59

  ISO8601timeOnlyPlus - used for Time (within a day)
                  format: "hh:mm[{W|Z}]"   'W' is the default
      notes:
        hh - hour, 2 digits, max: 40 (see  Note - 24hr clock)
        mm - minute, 2 digits, max: 59
        Z - relative to Zulu time
        W - relative to Wall time


  ISO8601dateOnlyNoTZ - used for CalendarDate (always relative to Wall time)
                  format: "YYYY-MM-DD"
      notes:
        YYYY - year (see Note - Sorting Years)
        MM - month, 2 digits, min: 01, max: 13 (see Note - Accounting Months)
        DD - day of month, 2 digits, min: 01, max: see Note - Max Days in Month.

  ISO8601plusZuluRelative - used for Instance - ISO8601plusZuluRelative
                  format: "YYYY-MM-DDThh:mm[:ss[.fs]]Z[{+|-}oh[:om]]"
      notes:
        YYYY - year (see Note - Sorting Years)
        MM - month, 2 digits, min: 01, max: 13 (see Note - Accounting Months)
        DD - day of month, 2 digits, min: 01, max: see Note - Max Days in Month.
        hh - hour, 2 digits, max: 40 (see  Note - 24hr clock)
        mm - minute, 2 digits, max: 59
        ss - second, 2 digits, max: 59
        fs - fraction of second, min digits: 1, max digits: open
        {+|-} - local time zone offset from Zulu direction indicator
        oh - local time zone offset from Zulu hours, 2 digits, max: 15 (I think)
        om - local time zone offset from Zulu minutes, 2 digits, max: 59

        If there is a local time zone offset from Zulu, then the offset is from the Instance creators delta between Zulu time and Wall time at the time of creation!


Note - Max Days in Month:

  In addition to variation on month length (30 or 31 or February at 28 and leap year rules), there is the addition of the concept of Accounting 13th month (see Note - Accounting Months).

Note - Sorting Years:

  Due to the need to sort CalendarDate and Instances, the year is always represented as 4 digits.  This means there is a hard limit on the year at 9999 and NO support for any non-AD years.

Note - Accounting Months:

  Many firms, for accounting reasons, treat the year as having 12 uniform months and an extra "short" 13th month.

Note - 24hr clock:

  Since 00:00 means the first minute of the day, 24:00 means the upcoming midnight minute.
  Any values over 24:00 would be used for events or shifts that start before midnight and are scheduled to end after midnight.
