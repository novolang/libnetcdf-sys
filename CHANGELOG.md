# Changelog

All notable changes to libnetcdf-sys are recorded here. The format
is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.1 — 2026-09-24

The documentation and comments in plain prose; no declaration changed.

## 0.1.0 — 2026-09-16

The first release: forty entry points of the NetCDF-C library, one
`@ffi` declaration each, and no logic.

### Added

- `libnetcdf` — the whole surface, in seven groups.
  - The dataset: `nc_create`, `nc_open`, `nc_close`, `nc_sync`,
    `nc_redef`, `nc_enddef`, `nc_inq` and `nc_inq_format`.
  - The dimensions: `nc_def_dim`, `nc_inq_dimid`, `nc_inq_dimlen` and
    `nc_inq_dimname`.
  - The variables: `nc_def_var`, `nc_inq_varid`, `nc_inq_varname`,
    `nc_inq_vartype`, `nc_inq_varndims` and `nc_inq_vardimid`.
  - The data: `nc_put_var_double`, `nc_get_var_double`,
    `nc_put_var_int`, `nc_get_var_int`, `nc_put_var_text`,
    `nc_get_var_text`, and the four `vara` calls that move a
    rectangular slice.
  - The attributes: the text, double and integer pairs, `nc_inq_att`
    and `nc_inq_attname`.
  - The netCDF-4 storage settings: `nc_def_var_deflate` and
    `nc_def_var_chunking`.
  - Errors and version: `nc_strerror` and `nc_inq_libvers`.
- `tests/libnetcdf_tests.nv` — ten tests over the signatures. Every
  file the suite writes is made under a fresh directory from
  `fs.temp_dir` and deleted again.

### Three element types, and the library converts to the rest

NetCDF has twelve element types and it converts between the type a
variable is stored as and the type the caller asks for. The accessor's
name says what the caller's buffer holds, not what the file holds.

So three kinds of accessor cover all of them. `nc_get_var_double` reads
a variable of any numeric type into eight-byte floats — including an
`NC_FLOAT` variable, which is the common one in this format.
`nc_get_var_int` does the same for the integer types, and
`nc_get_var_text` is for `NC_CHAR`.

The `float` accessors are absent for a second reason as well: the `ptr`
module writes a 32-bit float with `ptr.write_f32` and has no call that
reads one back, so a program holding a buffer of them would decode the
bytes itself. Reading as doubles is the shorter road, and it is the one
the package leaves open.

### The out-parameter slot

Most of the library's answers arrive through an out-parameter, and most
of those are a C `int`, which is four bytes. `ptr.alloc_word` gives an
eight-byte slot that is zeroed, so the four-byte answer is read back
with `ptr.read_word` on a little-endian machine. A `size_t`
out-parameter — a dimension's length, an attribute's element count —
fills the whole eight bytes and needs no such care.

### Not a `0.0.x` interface release

An interface release is the shape whose every `pub fn` body is a
`todo()`. Every `pub fn` here is an `@ffi` declaration with no body, so
`novo pkg publish` reads the package as a release with bodies and
refuses a `0.0.x` version for it. The first release of a bindings
package is therefore `0.1.0`.

### The wrapped library

`wraps = "libnetcdf"`. A netCDF-4 build of it links against libhdf5 in
turn, because the netCDF-4 format is HDF5 underneath, and the loader
resolves that for itself. `libnetcdf-sys` and `libhdf5-sys` are
separate packages because they are separate libraries, and a program
that reads a netCDF-4 file through this package never sees the HDF5
one.

### Unverified

NetCDF is not installed on the staging machine, so the suite has never
linked: `novo test` stops at `/usr/bin/ld: cannot find -lnetcdf`. The
declarations were checked against the NetCDF-C reference, and
`novo pkg build` type-checks them, which is the whole of what has been
measured. Treat the package as unmeasured until someone runs the suite
against a real library.

### Named as missing

**The `float`, `short`, `schar`, `uchar`, `longlong` and `string`
accessors.** `nc_put_var_float` and its family are the same calls for a
different caller buffer. The double, integer and text forms read and
write every numeric and character variable, because the library
converts. `nc_get_var_string` is a different case: it answers an array
of pointers to strings the library allocated, and freeing them needs
`nc_free_string`, which is a second call over a layout the caller lays
out by hand.

**The strided and mapped accessors.** `nc_put_vars_double` takes a
stride per dimension and `nc_put_varm_double` takes a memory map as
well. The whole-variable and slice forms are here.

**The user-defined types.** `nc_def_compound`, `nc_insert_compound`,
`nc_def_vlen`, `nc_def_enum` and `nc_def_opaque` are netCDF-4's own,
and a compound type is described by the byte offsets of its members.
They are left out of the first release.

**The groups.** `nc_def_grp` and `nc_inq_grps` give a netCDF-4 file a
tree rather than a flat list. They are left out of the first release;
every call in this package works on the root group, which is the whole
of a classic file.

**The parallel interface.** `nc_create_par` and `nc_open_par` take an
`MPI_Comm` and an `MPI_Info`, which most MPI implementations pass by
value.

**The generic `nc_get_att` and `nc_put_att`.** They take a `void *` and
the caller's own type number. The three typed pairs say what the buffer
holds in the name of the call, which is what a binding can check.

**`nc_set_fill` and the fill values.** A variable's fill value is
written with `nc_put_att` of the variable's own type under the name
`_FillValue`, so the typed attribute calls here reach it; the mode
switch is left out of the first release.
