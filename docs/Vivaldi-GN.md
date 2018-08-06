# Vivaldi GN extensions

This branch contains a number of extensions to Chromium's GN project generator
system. Vivaldi use these extensions to make maintenance of its adaptions of
the Chromium project easier, and reduce the number of edits we have to make in the
Chromium source.

These extensions are mostly useful for projects that embed Chromium as a
submodule, and when the extensions are used in GN files hosted in the parent
module.

The extensions:
* declare_overrides(): Set GN arguments from GN files
* Update variables in targets.
* Vendor specific label aliases like //vendor/foo

Bug fixes:
* Allow inlining of files in BUILDCONFIG.gn files

# [declare_overrides()](reference.md#declare_overrides)

When using GN to configure a build it is possible to add arguments to enable
or disable features, or specify the target system parameters (OS, CPU, etc.),
or specific build properties (e.g. Official build, debug/release, Jumbo build).

Many of the feature configurations will either be static for a given project,
or depend on which platform or other build or features are set.

The static settings can easily be set in the default_arguments section of the
dotgn file, but arguments set based on conditionals (such as OS and CPU) cannot
be set that way.

While there are other ways around this, such as importing a configuration file,
this depends on the developer actively selecting the configuration file and
including it on the GN command line.

declare_overrides() provides an alternative approach, allowing the GN project
files to specify the arguments as part of the BUILDCONFIG.gn file (usually at
the end), before the variables are declared by a declare_arguments() section.

Example:

  declare_overrides() {
    if (is_win && !is_official) {
      is_win_fastlink = true
    }
  }
   
declare_overrides() have the same restrictions regarding reuse of variables as
declare_arguments().


# [update_target](reference.md#update_target) and [update_template_instance](reference.md#update_template_instance)

The update_target and update_template_instance extensions allow modification
of variables in GN targets and template instantiations after the GN
target/template instantiation have been set up by the original GN code.

The primary usage areas are to update (remove or add) dependencies and sources,
but can also be used to update the output name, input and output lists, and
script arguments lists.

Please note that these functions cannot update the logic of the target prior to
being executed as part of the target.

  update_target example:

    in //foo/source_updates.gni:

      update_target("//bar:bar") {
        sources += ["//foo/foo.cc"]
      }

    in //bar/BUILD.gn

      #import would normally go in top level BUILD.gn
      import("//foo/source_updates.gni")

      executable("bar") {
        sources = ["bar.cc"]
      }

    The result would become

      executable("bar") {
        sources = [
          "bar.cc",
          "//foo/foo.cc",
        ]
      }

  update_template_instance example:

    in //foo/source_updates.gni:

      update_template_instance("//bar:bar") {
        sources += ["//foo/foo.cc"]
      }

    in //bar/BUILD.gn

      #import would normally go in top level BUILD.gn
      import("//foo/source_updates.gni")

      template("baz") {
        executable(target_name) {
          sources = invoker.sources
        }
      }

      baz("bar") {
        sources = ["bar.cc"]
      }

    The result would become

      baz("bar") {
        sources = [
          "bar.cc",
          "//foo/foo.cc",
        ]
      }

# [Label aliases](reference.md#set_path_map)

A problem when combining two or more projects that have independent GN label
namespaces is how to combine them.

While it is possible to have all such projects as submodules and labels
relative to the root of the top project, this only works when the projects
are assuming that they will be a submodule with a given path prefix.

This GN variant has added label functionality map //vendor/ labels to a
specific root directory, while allowing a different root directory, e.g. a
submodule to specify most other paths.

  Example (Chromium as submodule of vendor):

    set_path_map([
      # Prefix, actual path
      # Most specific prefixes first
      [
        "//vendor",
        "//",
      ],
      [
        "//out",
        "//out",
      ],
      [
        "//",
        "//chromium",
      ],
    ])

  In this configuration //vendor/foo will map to /foo in the main source dir,
  and //out will map to /out in the main source dir, while (e.g) //chrome will
  map to /chromium/chrome under the main source dir.

# Inlining of files in BUILDCONFIG.gn 

When a project specifies its own BUILDCONFIG.gn file, but want to also import
a subproject's BUILDCONFIG.gn file, that is not possible in normal GN due to
how variable scopes are handled by GN.

This version of GN modifies the handling of import during BUILDCONFIG.gn
processing, changing the import to an inline operation.
