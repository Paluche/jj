# Better Bookmark coloration

Authors: [Hubert Lefevre](mailto:hubelefevre@gmail.com)

**Summary:** This document aims at resolving a mess with the color configuration
surrounding the display of the different types of bookmarks. Which is currently
pretty bad, but invisible since all configuration to color bookmarks is set to
be either `magenta` or `bright magenta` in the case of a bookmark at the working
copy. But once you aim at having different colors for the different types of
bookmarks, you then realize that you cannot obtain the proper coloration for the
proper type of bookmark displayed.

## The different types of bookmarks

We distinguish the following types of bookmarks:

- **Local bookmark**: Bookmark, which exist locally which you can directly
  modify. Currently the associated color settings are `local_bookmarks` and
  `"working_copy local_bookmarks"`.
- **Remote bookmark**: Bookmark, which exists on a remote, it is the
  representation of what is the current known state of the bookmark on a remote.
  Often displayed as `<bookmark-name>@<remote-name>`. Currently the associated
  color settings are `remote_bookmarks` and `"working_copy remote_bookmarks"`.
  Note that in a Git colocated repository, if the bookmark exists as a git local
  branch then jj will show it as a remote bookmark `<bookmark-name>@git` while
  this is something purely local.
- **Tracked bookmark**: A bookmark is tracked when a local bookmark is
  associated with at least one bookmark from one remote. You can have one local
  bookmark being tracked with several remote bookmarks, but each of those remote
  bookmarks has to come from different remotes. The local and remote bookmarks
  share the same name (e.g. `<bookmark-name>`), the remote one being appended
  the remote name to distinguish it from the local one.
  (e.g. `<bookmark-name>@<remote-name>`). Usually we will prefer showing the
  local bookmark only, and when there is divergence between the local and the
  remote bookmarks, then the bookmark name is appended with a `*` character,
  showing the user there is a push to be done for that bookmark.
  The remote bookmark is still showed only if it is currently set at a different
  change than the remote one. Currently the associated color settings are
  `bookmarks` and `working_copy bookmarks`, but that is an over-simplification
  as we will see later.

There is an additional color configuration related to the bookmarks which
`bookmark` and the associated `"working_copy bookmark"`. The first one is used
for coloring the bookmark listed by the `jj bookmark list` command through the
default value of the template `templates.bookmark_list =
'format_commit_ref(self, "bookmark") ++ "\n"'`. The second one is not used but
here to be aligned with the other color configurations.

You find the same logic being the color configurations `tags`/`"working_copy
tags"` and `tag` / `"working_copy tag"`. The color defined by `tags` is used to
color tags in the logs for instance, the `working_copy` variant being used if
the tag is at the commit associated with the current state of the working copy.
And `tag` is used by the command `jj tag list` through the default template
`template.tag_list = 'format_commit_ref(self, "tag") ++ "\n"'`, its
`working_copy` variant is not used.


## Issues with the current color system for bookmarks

Basically the selected color depends on what information the template says what
to display and not what the type of the bookmark is.

To display only local bookmarks, you will use `commit.local_bookmarks()` for
instance in the template of `builtin_log_detailed(commit)` we find
```toml
'builtin_log_detailed(commit)' = '''
  ...
  surround("Bookmarks: ", "\n", separate(" ",
    commit.local_bookmarks(),
    commit.remote_bookmarks(),
  )),
  ...
'''
```

Which will display both local and remote bookmarks in there configured colors.
If a bookmark `<bookmark-name>` is tracked with a local and remote part on the
remote named `<remote-name>`, then this template will show both
`<bookmark-name>` with the `local_bookmarks` color, and
`<bookmark-name>@<remote-name>` with the `remote_bookmarks` color. This is
exactly what we would like to have as result.

But always listing local and remote bookmarks is overly verbose in the case of
tracked bookmarks, and more if there is several remotes tracked. So here comes
the `commit.bookmarks()` template function. Which will display:

- if the bookmark is local then display the name, if it is tracked and has a
  diverged from any of the associated remote, append a `*` character if the
  remote part is not at the same state. So we would see either `<bookmark-name>`
  for local only and up-to-date tracked bookmarks or `<bookmark-name>*` for
  divergent local tracked bookmarks.
- if the bookmark is remote. If it is tracked and the associated local bookmark
  is at the same commit, then do not show it otherwise, show it as
  `<bookmark-name>@<remote-name>` for remote bookmarks or tracked remote
  bookmarks.

And all this is displayed with the same color, the one set by `bookmarks` and
`"working_copy bookmarks"`. This representation is the one used by the default
template for `jj log` command `builtin_log_compact`, which uses
`builtin_log_compact(commit)` which uses `format_short_commit_header(commit)`

```toml
'format_short_commit_header(commit)' = '''
separate(" ",
  format_short_change_id_with_change_offset(commit),
  format_short_signature(commit.author()),
  format_timestamp(commit_timestamp(commit)),
  commit.bookmarks(),
  commit.tags(),
  commit.working_copies(),
  format_short_commit_id(commit.commit_id()),
  format_commit_labels(commit),
  if(config("ui.show-cryptographic-signatures").as_boolean(),
    format_short_cryptographic_signature(commit.signature())
  ),
)
'''
```

So with that representation you cannot distinguish local bookmarks from remote
bookmarks based on the color representation.

### Issues with the `jj bookmark list` command

This command uses a single configuration color `bookmark`, which is blocking the
possibility to differentiate the type of the displayed bookmark based on the
color.

Although the displayed information (more or less complete depending on the used
option) could make possible to distinguish the different type of bookmarks,
there is often no way to properly distinguish a local or remote bookmark between
one which is tracked and one which is not tracked.

## Proposed solutions

### Better documentation about colors

First of all, to get to the conclusion of where each of the colors were used and
how, I had to experiment because there is no proper documentation. So we need to
review the documentation regarding the colors, in both the default color
configuration file `cli/src/config/colors.toml` and in `docs/configs.md` (or a
new dedicated page `docs/colors.md`,

### What do we want to display?

My initial goal, when diving into this rabbit hole was that I wanted to have a
way to differ from tracked local bookmark and non-tracked local bookmarks. We
basically can distinguish the following types of bookmarks:

- local bookmarks
- remote bookmarks
- tracked local or remote bookmarks

And that in all the output which concerns bookmarks, so the color configurations
`bookmark` and `"working_copy bookmark` should be deprecated.

The idea of having the same color for all tracked bookmarks not matter is local
or remote, is that the representation of a local and remote bookmarks is already
different, what matters is to differentiate if it is a tracked or not

Resulting in a default color configuration which would look like:

```toml
[colors]
local_bookmarks = "magenta"  # Local only bookmark
remote_bookmarks = "magenta"  # Remote only bookmark
tracked_bookmark = "magenta"  # Tracked bookmark, local or remote

# Working copy variants
"working_copy local_bookmarks" = "bright magenta"
"working_copy remote_bookmarks" = "bright magenta"
"working_copy tracked_bookmarks"  = "bright magenta"
```

If we want to be thorough and offer the possibility to fine tune all the case,
we can add specific configurations to differentiate the tracked local bookmarks
from the tracked remote bookmarks, and another one for the divergent tracked
local bookmark.

```toml
[colors]
local_bookmarks = "magenta"  # Local only bookmark
remote_bookmarks = "magenta"  # Remote only bookmark
tracked_local_bookmark = "magenta"
tracked_remote_bookmark = "magenta"
divergent_tracked_local_bookmark = "magenta"

"working_copy local_bookmarks" = "bright magenta"  # Local only bookmark
"working_copy remote_bookmarks" = "bright magenta"  # Remote only bookmark
"working_copy tracked_local_bookmark" = "bright magenta"
"working_copy tracked_remote_bookmark" = "bright magenta"
"working_copy divergent_tracked_local_bookmark" = "bright magenta"
```

This starts to make a lot of different colors to configure if one wants to be
able to differentiate each of the types. And at some point we could see to offer
the user a way to format each type of bookmarks as they see fit.

This could come under multiple template configurations, with the defaults being
the following if we respect the current default settings.

```toml
[templates]
local_bookmarks(bookmark) = "bookmark"
remote_bookmarks(bookmark, remote) = "bookmark ++ '@' ++ remote"
tracked_local_bookmark(bookmark) = "bookmark"
tracked_remote+bookmark(bookmark, remote) = "bookmark ++ '@' ++ remote"
```

### Review how bookmarks are displayed

We need a single function which should be used to format a bookmark, it would
take into account the following properties of the bookmark to display:

- Name of the bookmark (`&str`)
- if the bookmark is a remote one (`BookmarkType::Remote(Option<&str>, bool)`):
    - Name of the remote if the bookmark is a remote one (`Option<&str>`)
    - Is the remote bookmark tracked? (`bool`)
- if the bookmark is a local one (`BookmarkType::Local(Vec<&str>, bool)`):
    - Names of the associated remotes the local bookmark tracked (`Vec<&str>`)
    - Does the local bookmark diverge from any of the associated remote?
      (`bool`).
- Is the format to give information about the state of the change the working
  copy is currently at? (`bool`)

So we could start by reviewing how the bookmark are represented.

## Expanding the logic to the tags

We observe somehow the similar issues with the tag color configurations:

- No distinction possible between the different types of tags:
    - The ones that exists only locally (which should not be immutable IMO)
    - The ones that represents the current known location of the tag for a
      specific remote.
- No distinction between the tags and the bookmarks
