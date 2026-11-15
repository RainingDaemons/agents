---
title: Libraries
source: https://help.penpot.app/user-guide/design-systems/libraries/
scraped: 2026-11-15
---

# Libraries

Libraries may include components, colors and typographies. Learn how to create and manage them to better organize the pieces of your designs and speed your workflow.

## File libraries

Each file has its own file library which is where the assets that belong to this file are stored.

You have two ways to access the file library from the file workspace:

- Click the assets tab icon at the left sidebar.
- Press Alt/⌥ + i.

## Shared libraries

### Publish as shared library

You can publish any regular file as a shared library. This means that the file library of this file will be available to be connected to other files that exist in the same team, so its library assets can be reused.

There are two ways to publish a library:

- Using the file main menu.
- From the libraries panel, that you can launch from the button at the top of the assets tab. It reads "Add library" when the file has no connected libraries and "Manage library" when it has.

### Unpublish a shared library

You can unpublish any library anytime the same way you can publish it, both from the file menu and the libraries panel.

Unpublishing a library will disconnect it from the files where it was connected. The assets that have already been used in other files will remain, but no longer linked with the now unpublished library.

### Connect shared libraries

To add a Shared Library from another file, launch the libraries panel, then search and select the available libraries. If you see the message "There are no Shared Libraries available", start by publishing other files as a shared library or add from Penpot's Libraries & templates.

### Disconnect shared library

You can disconnect any library anytime from the libraries panel just by clicking on the disconnect button.

### Use shared libraries

Shared libraries will be listed at the assets panel, at the workspace left sidebar. You can expand and collapse them to access the assets of each connected shared library.

### Open shared library file

Click on the arrow icon at the right of a shared library name to go to the file where the library is and edit its contents.

### Use tokens from a shared library

A file can use the design tokens of a connected shared library instead of its own. This keeps one source of truth: tokens are defined once in the library, and every file that uses it gets the same tokens, sets and themes.

A file uses one tokens source at a time: its own tokens or the tokens of one connected library.

1. Connect the library to the file (see Connect shared libraries).
2. Open the **Libraries** panel from the assets tab. The **This file** tab lists the file library and the connected libraries, with how many tokens, sets and themes each one has.
3. Click **Set as tokens source** next to the library. The current source is labelled **Tokens source**.

The button only appears for libraries that have tokens of their own. A library that already uses tokens from another library cannot be a tokens source.

You can also change the source from the top of the **Tokens** tab, which shows **Source:** followed by **This file** or the library name. In the **Assets** tab, a tokens icon marks the library that is the current source.

#### What you can do with library tokens

- **Apply** them to layers, as with local tokens.
- **Activate themes and sets.** The active themes and sets are saved in each file, so two files can use the same library with different themes. When you set a library as source, the file starts with the themes and sets active in the library.
- **Edit them in the library.** From a file that uses library tokens you cannot create, edit, rename or delete tokens, sets or themes. Use the open-in-new-tab button next to the source name, or **Edit themes** in the theme menu, to open the library on its Tokens tab.

#### How changes reach your file

Token changes in the library reach the files that use it right away. You do not need to accept them in **Updates**, which only lists components, colors and typographies. When sets or themes are added, removed or renamed in the library, Penpot shows a notice in the file.

If a token applied in your file is renamed or deleted in the library, the layer keeps its value but the token appears as missing. Apply the new token again.

#### Switching back to the file's own tokens

Click **Set as tokens source** on the **File library** row. The file's own tokens were kept while the library was the source, so they come back. Tokens applied from the library appear as missing unless the file has tokens with the same names. Disconnecting a library that is the tokens source also switches the file back to its own tokens.
