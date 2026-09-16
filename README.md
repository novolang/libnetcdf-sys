# libnetcdf-sys

NetCDF is a file format for arrays of measurements and the C library
that reads and writes it. One file holds named arrays over named
dimensions, each array with a type and each with attributes describing
what it means, and a program reads a rectangular piece of an array
without reading the rest. The format and the library are documented in
[the NetCDF-C reference](https://docs.unidata.ucar.edu/netcdf-c/current/).
This package declares forty of that library's entry points to
novo-lang, one declaration each.

**Status: a binding, not a port.** Every function in this package is a
declaration of a function in libnetcdf. The package contains no logic
of its own, and it does nothing without the C library installed. The
forty entry points read and write the classic data model — dimensions,
variables, attributes — in both the classic and the netCDF-4 file
formats; the section "What is not included" says what a program still
cannot do with them alone.

**Unverified.** NetCDF is not installed on the machine where this
package was written, so the test suite has never been linked. See the
"Tests" section.

## What it is

A **dataset** is one file. It holds three kinds of thing, and a program
that understands those three understands the format.

A **dimension** is a named length. `time` might be 8760 and `depth` 40.
One dimension in a classic file may be **unlimited**, which means it
grows as data is written to it rather than being fixed when the file is
made.

A **variable** is a named array over some of those dimensions, with one
element type for all of it. A variable over `time` and `depth` has
8760 by 40 elements. A variable over no dimensions holds exactly one.

An **attribute** is a small named value carried beside a variable, or
beside the dataset itself. `units` is an attribute; so is the range a
reader should expect. An attribute attached to the dataset rather than
to a variable uses the variable identifier -1, which the C header calls
`NC_GLOBAL`.

A classic dataset has two states. In **define mode** dimensions,
variables and attributes may be added but no data may be written; out
of it, the reverse. `nc_enddef` leaves define mode and is where the
layout of the file is decided. A netCDF-4 dataset does not need the
distinction, and accepts the calls anyway.

There are two **file formats** behind one interface. The classic format
is NetCDF's own, a header followed by the arrays. The **netCDF-4**
format is an HDF5 file with a NetCDF layout inside it, and it is what
compression, chunking and a tree of groups need. A program says which
one it wants when it creates a file, and never needs to ask again: the
same calls read both.

The library **converts as it reads and writes**. A variable stored as
four-byte floats read with `nc_get_var_double` arrives as eight-byte
floats. This is why the accessor's name says what the caller's buffer
holds rather than what the file holds.

## Install

```
novo pkg add libnetcdf-sys
```

Adding the package does not install the C library. On Debian and Ubuntu
the library and its header come from the system package
`libnetcdf-dev`:

```
sudo apt install libnetcdf-dev
```

On macOS the Homebrew formula is `netcdf`. On other systems the library
builds from Unidata's source with CMake.

A build with netCDF-4 support links against libhdf5 in turn, and the
loader resolves that for itself.

## Example

Every variable in a file, with its shape:

```novo ignore
use libnetcdf

fn main() [io, ffi]
    let nc_slot = ptr.alloc_word()
    // 0 is NC_NOWRITE.
    let rc = libnetcdf.nc_open("temperature.nc", 0, nc_slot) as i32
    if rc != 0
        println(ptr.read_str(libnetcdf.nc_strerror(rc)))
        return
    let ncid = ptr.read_word(nc_slot)

    let nvars = ptr.alloc_word()
    let _counted = libnetcdf.nc_inq(ncid, 0, nvars, 0, 0)
    let name = ptr.alloc(257)
    let ndims = ptr.alloc_word()
    for varid in 0..ptr.read_word(nvars)
        let _named = libnetcdf.nc_inq_varname(ncid, varid, name)
        let _shaped = libnetcdf.nc_inq_varndims(ncid, varid, ndims)
        println("${ptr.read_str(name)} over ${ptr.read_word(ndims)} dimension(s)")

    ptr.free(ndims)
    ptr.free(name)
    ptr.free(nvars)
    ptr.free(nc_slot)
    let _closed = libnetcdf.nc_close(ncid)
```

The example is fenced as an illustration rather than a compiled block
because `novo doc` compiles the blocks in documentation comments and
not the ones in this file. The same calls are in
`tests/libnetcdf_tests.nv`.

## What the package contains

| Module | Contents |
| --- | --- |
| `libnetcdf` | Every entry point, in seven groups: the dataset, the dimensions, the variables, the data, the attributes, the netCDF-4 storage settings, and errors and version. |

The seven groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| Dataset | 8 | Creates, opens, flushes and closes a file, enters and leaves define mode, and reports the four counts and the format. |
| Dimensions | 4 | Defines a dimension, finds one by name, and reports its length and its name. |
| Variables | 6 | Defines a variable, finds one by name, and reports its name, its element type and its shape. |
| Data | 10 | Moves a whole variable or a rectangular slice of one, as eight-byte floats, four-byte integers or bytes. |
| Attributes | 8 | Writes and reads a text, double or integer attribute, and reports an attribute's type, length and name. |
| netCDF-4 storage | 2 | Sets a variable's chunk shape and its deflate compression. |
| Errors and version | 2 | Turns a status into a sentence, and names the library. |

## How to choose an entry point

`nc_get_var_double` reads a variable of any numeric type, because the
library converts. Reach for it unless the elements are known to be
integers and the conversion is unwanted.

The `vara` calls move a rectangular slice: a starting corner and a
count along each dimension. They are the calls for a file too large to
read whole, and the only way to extend an unlimited dimension a piece
at a time.

`nc_get_var_text` is for a variable of type `NC_CHAR`, which is how a
classic file carries a string: the last dimension is its length.

`nc_def_var_chunking` and `nc_def_var_deflate` need a netCDF-4 file,
which means `nc_create` with the mode 4096. They fail on a classic
dataset.

`nc_inq` with four zeros is how a program that knows nothing about a
file starts: it answers how many dimensions, variables and attributes
the file has, and the identifiers are 0 upwards in each case.

## The rules a user needs

1. **Zero is success and every other status is negative.** The status
   is a C `int`, so write `as i32` before comparing it with a negative
   number. `nc_strerror` turns the status into a sentence and answers
   the address of a C string the library owns; read it with
   `ptr.read_str` and do not free it.
2. **An out-parameter for a C `int` is four bytes in an eight-byte
   slot.** `ptr.alloc_word` gives a slot that is zeroed, so
   `ptr.read_word` reads back the four-byte answer. A dimension's
   length and an attribute's element count are a `size_t` and fill the
   whole eight bytes.
3. **A dataset, a dimension and a variable are each numbered from
   zero**, in the order they were defined. `nc_inq` gives the counts
   and the identifiers follow from them.
4. **`NC_GLOBAL` is -1.** An attribute on the dataset itself is written
   with that as the variable identifier.
5. **The type and mode numbers are numbers**, because the C header
   spells them as macros.

   | Name | Number | What it means |
   | --- | --- | --- |
   | `NC_NOERR` | 0 | success |
   | `NC_NOWRITE` | 0 | open for reading |
   | `NC_WRITE` | 1 | open for reading and writing |
   | `NC_CLOBBER` | 0 | create, overwriting any existing file |
   | `NC_NOCLOBBER` | 4 | create, failing if one exists |
   | `NC_64BIT_OFFSET` | 512 | the classic format with 64-bit offsets |
   | `NC_NETCDF4` | 4096 | the HDF5-backed format |
   | `NC_BYTE` | 1 | a one-byte signed integer |
   | `NC_CHAR` | 2 | a byte of text |
   | `NC_SHORT` | 3 | a two-byte integer |
   | `NC_INT` | 4 | a four-byte integer |
   | `NC_FLOAT` | 5 | a four-byte float |
   | `NC_DOUBLE` | 6 | an eight-byte float |
   | `NC_UNLIMITED` | 0 | as a dimension length, the dimension that grows |
   | `NC_GLOBAL` | -1 | as a variable identifier, the dataset itself |
   | `NC_CONTIGUOUS` | 0 | as a storage setting, one run |
   | `NC_CHUNKED` | 1 | as a storage setting, chunks |

6. **The accessor's name says what the caller's buffer holds.** The
   library converts between it and the variable's own type, and answers
   a range error when a value does not fit. `nc_get_var_double` on an
   `NC_FLOAT` variable is the ordinary way to read one.
7. **A data buffer carries no length.** The library moves as many
   elements as the variable's shape, or the slice's count, implies. Ask
   `nc_inq_dimlen` for the length and multiply by the element width
   before allocating.
8. **A `start` and a `count` are one eight-byte number per dimension**,
   outermost first. `ptr.alloc(8 * rank)` reserves either, and
   `ptr.write_word` fills it.
9. **A name buffer must hold 257 bytes**, which is `NC_MAX_NAME + 1`.
   `nc_inq_dimname`, `nc_inq_varname` and `nc_inq_attname` write a
   terminating zero, so `ptr.read_str` reads the answer back.
10. **A text attribute and a text variable carry no terminating zero.**
    Their length is the attribute's element count or the dimension's
    length. Read them with `ptr.read_bytes_n` rather than
    `ptr.read_str`.
11. **A classic dataset must be in define mode to be given a new
    dimension, variable or attribute.** `nc_create` starts there and
    `nc_enddef` leaves; `nc_redef` goes back. A netCDF-4 dataset does
    not need the distinction.
12. **Compression needs chunks and needs netCDF-4.**
    `nc_def_var_deflate` fails on a classic dataset, and it must be
    called in define mode before any data is written. A compressed
    variable is chunked, and the library chooses a chunk shape when
    `nc_def_var_chunking` has not.
13. **A dataset that is not closed may be missing its last writes.**
    `nc_close` writes them out; `nc_sync` does the same and leaves the
    file open.

## What is not included

- **The `float`, `short`, `schar`, `uchar`, `longlong` and `string`
  accessors.** `nc_put_var_float` and its family are the same calls for
  a different caller buffer, and the double, integer and text forms
  read and write every numeric and character variable because the
  library converts. The `ptr` module also writes a 32-bit float and has
  no call that reads one back, so a buffer of them would be decoded by
  hand. `nc_get_var_string` is a different case: it answers an array of
  pointers to strings the library allocated, which a second call has to
  free.
- **The strided and mapped accessors.** `nc_put_vars_double` takes a
  stride per dimension and `nc_put_varm_double` a memory map as well.
  The whole-variable and slice forms are here.
- **The user-defined types.** `nc_def_compound`, `nc_insert_compound`,
  `nc_def_vlen`, `nc_def_enum` and `nc_def_opaque` are netCDF-4's own,
  and a compound type is described by the byte offsets of its members.
  They are left out of the first release.
- **The groups.** `nc_def_grp` and `nc_inq_grps` give a netCDF-4 file a
  tree rather than a flat list of variables. Every call in this package
  works on the root group, which is the whole of a classic file.
- **The parallel interface.** `nc_create_par` and `nc_open_par` take an
  `MPI_Comm` and an `MPI_Info`, which most MPI implementations pass by
  value.
- **The generic `nc_get_att` and `nc_put_att`.** They take a `void *`
  and the caller's own type number. The three typed pairs say what the
  buffer holds in the name of the call.
- **`nc_set_fill`.** A variable's own fill value is an attribute named
  `_FillValue`, which the typed attribute calls reach; the mode switch
  is left out of the first release.

## Related packages

There is no novo-lang replacement for this package and none is planned.
The classic NetCDF format is small enough to reimplement, but a file in
the netCDF-4 format is an HDF5 file, and HDF5 is a format defined by
its implementation. A reader that handled one format and not the other
would be a reader a user could not rely on.

`libhdf5-sys` binds the library underneath the netCDF-4 format. A
program that has an HDF5 file rather than a NetCDF one wants that
package. A program that has a NetCDF file wants this one, whichever
format it is in: the NetCDF interface is smaller, and it names its
element types with plain integers where HDF5 names them with global
variables a binding cannot reach.

`dataframe-nv` and `ndarray-nv` are where the arrays go once they are
read.

## Tests

`tests/libnetcdf_tests.nv` holds ten tests written against the
signatures. They call the C library, so `novo test` needs NetCDF
installed and linkable:

```
novo test tests/libnetcdf_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

Every file the suite writes is made under a fresh directory from
`fs.temp_dir` and deleted again, and nothing reaches the network. The
dataset test creates an empty classic file, reopens it and asserts that
it has no dimensions, no variables and no attributes, and that its
format is the classic one. The dimension and variable tests assert that
a definition comes back by name with the length, type and shape it was
given. The three round-trip tests write four doubles, four integers and
four bytes of text and read them back. The integer test also reads the
same variable as doubles, which is the conversion the library performs.
The slice test writes two elements into the middle of a variable, reads
them back on their own, and asserts that the element outside the slice
was not touched. The attribute test round-trips a text, a double and an
integer attribute on the dataset itself and lists the first of them
back by number. The last test creates a netCDF-4 file, chunks and
compresses a variable, writes to it, and asserts that the format the
file reports is netCDF-4 rather than classic.

**Unverified.** NetCDF is not installed on the staging machine, so the
suite has never linked: `novo test` stops at `/usr/bin/ld: cannot find
-lnetcdf`. Every assertion above is written from the NetCDF-C reference
and none of them has been observed to pass.

## Implementation status

| Group | State |
| --- | --- |
| Dataset | Complete for the serial interface. |
| Dimensions | Complete. |
| Variables | Complete for the classic data model. |
| Data | Complete for the double, integer and text buffers, whole and by slice. |
| Attributes | Complete for the three typed pairs. |
| netCDF-4 storage | Complete for chunking and deflate. |
| Errors and version | Complete. |
| Other caller buffer types | Absent. The three here read and write every variable, by conversion. |
| Strided and mapped access | Absent. Left out of the first release. |
| User-defined types | Absent. Left out of the first release. |
| Groups | Absent. Every call works on the root group. |
| Parallel interface | Absent. It takes an MPI communicator by value. |

## Licence

Apache-2.0. See [LICENSE](LICENSE).

NetCDF itself is distributed under a BSD-style licence from Unidata,
and installing it is the reader's own step.
