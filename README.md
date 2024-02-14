# Resource materials for the subject Development of web applications in the Cloud environment translated into English

__English summary__: This repository contains the source materials for the book "Vývoj webových aplikácii v prostredí Cloud" (Development of web applications in the Cloud environment) originaly written in Slovak language. The original book is available at http://wac-fiit.milung.eu/ translated book is avalable at TODO.

Materials are licensed under the [Creative Commons Attribution 4.0 International (CC BY 4.0) license](https://creativecommons.org/licenses/by/4.0/).

Markdown files and images for generating study materials for "Development of web applications in the Cloud environment" (WAC).

## Working with the Repository

In the `book-src` directory, all the materials are stored. Each section is stored in a separate directory. The `book-src` directory also contains the file `_toc.md` with the book's content. It includes relative links to individual chapters. Each section then has its own `_toc.md` file with links to chapters in that section. The content of a subsection can be included in the book or the parent section using the command `[#include section/_toc.md]`. This command also ensures that the chapters of the subsection are visible in the navigation panel only when the subsection is displayed in the main panel.

In the `book-src` directory, there is also the file `_links.md` with links to resources. This file is automatically added to each chapter, and the listed links can be used using symbolic names. It is recommended to use it in places where references to external sources are made, unless these sources are very specific. This allows generating a bibliography list and modifying references in one place.

## Contributing to the Book

The most suitable way is to use [Development Container](https://containers.dev/overview) when working with the book, such as creating a new [Github Codespaces](https://github.com/features/codespaces) or opening the Development Container using [Visual Studio Code Remote - Containers](https://code.visualstudio.com/docs/remote/containers).

The development container creates a `postStartCommand` process that creates an HTTP server listening on port `3380`. After opening the port (in the _Ports_ tab), you can see the current version of the script in the browser. To immediately reflect your changes, disable caching in the browser in the _Network_ tab of the _Developer Tools_.

In case the process fails—for example, if generating new content is not automatically reflected—terminate the `postStartCommand` process, open a new PowerShell terminal, navigate to the `wac-textbook` folder, and execute the command:

```ps
./scripts/run.ps1 devc-start
```

## Generating the Book

A new version of the book can be generated by committing changes to the `main` branch and creating a new release with a tag in the format `v1.*.*`. This tag triggers a GitHub Action that generates a new version of the book and publishes it on the website TODO.

## Specific Markdown Extension:

In the `_toc.html` files, you can define custom chapter icons using the following syntax: `[$icon-name> Label](<link-url>)`. The icon name is the name of the icon from the [fontawesome](https://fontawesome.com/icons?d=gallery) library. The icon name is followed by the `>` character, and before it is the `$` character. `label` is the text of the hyperlink. If the icon name is not prefixed with the `$` character, the icon is taken from the [Material Symbols](https://fonts.google.com/icons) library. The icon name uses a hyphen `-` instead of an underscore `_`, as in the fontawesome library.

Example `_toc.md`:

```markdown
[$graduation-cap> Introduction](./README.md)
[Prologue](./prologue.md)

<hr />
## [language> Chapter 1: Web Development](dojo/web/000-README.md)

[#include dojo/web/_toc.md]
<hr />
```

### Notes with Icons

You can place a block quote with an icon using the following syntax: `>$icon-name:> Block quote text`. The icon name is the name of the icon from the [fontawesome](https://fontawesome.com/icons?d=gallery) library. The icon name is followed by the characters `:>` and before it is the `$` character. If the icon name is not prefixed with the `$` character, the icon is taken from the [Material Symbols](https://fonts.google.com/icons) library.

Use these icons:

- `>info:>` for additional information
- `>warning:>` for important information
- `>build_circle:>` for addressing potential issues
- `>$apple:>` for Mac OS-specific information
- `>homework:>` for marking individual work

### Highlighted Code Blocks

Inside a block, you can mark a line as inserted by placing the text `@_add_@` on that line; or you can mark a line as removed by placing the text `@_remove_@` on that line; or you can mark a line as important by placing the text `@_important_@` on that line. The rendering of this line will be highlighted with the respective color. The markers as well as the removed lines are not inserted into the clipboard when copying.
