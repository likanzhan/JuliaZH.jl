# 代码加载

!!! note
    这一章包含了加载包的技术细节。如果要安装包，使用 Julia 的内置包管理器[`Pkg`](@ref Pkg)将包加入到你的活跃环境中。如果要使用已经在你的活跃环境中的包，使用 `import X` 或 `using X`，正如在[模块](@ref 模块)中所描述的那样。

## 定义

Julia加载代码有两种机制：

1. **代码包含：**例如 `include("source.jl")`。包含允许你把一个程序拆分为多个源文件。表达式 `include("source.jl")` 使得文件 `source.jl` 的内容在出现 `include` 调用的模块的全局作用域中执行。如果多次调用 `include("source.jl")`，`source.jl` 就被执行多次。`source.jl` 的包含路径解释为相对于出现 `include` 调用的文件路径。重定位源文件子树因此变得简单。在 REPL 中，包含路径为当前工作目录，即 [`pwd()`](@ref)。
2. **加载包：**例如 `import X` 或 `using X`。`import` 通过加载包（一个独立的，可重用的 Julia 代码集合，包含在一个模块中），并导入模块内部的名称 `X`，使得模块 `X` 可用。 如果在同一个 Julia 会话中，多次导入包 `X`，那么后续导入模块为第一次导入模块的引用。但请注意，`import X` 可以在不同的上下文中加载不同的包：`X` 可以引用主工程中名为 `X` 的一个包，但它在各个依赖中可以引用不同的、名称同为 `X` 的包。更多机制说明如下。

代码包含是非常直接和简单的：其在调用者的上下文中解释运行给定的源文件。包加载是建立在代码包含之上的，它具有不同的[用途](@ref modules)。本章的其余部分将重点介绍程序包加载的行为和机制。

一个 *包（package）* 就是一个源码树，其标准布局中提供了其他 Julia 项目可以复用的功能。包可以使用 `import X` 或 `using X` 语句加载，名为 `X` 的模块在加载包代码时生成，并在包含该 import 语句的模块中可用。`import X` 中 `X` 的含义与上下文有关：程序加载哪个 `X` 包取决于 import 语句出现的位置。因此，处理 `import X` 分为两步：首先，确定在此上下文中是**哪个**包被定义为 `X`；其次，确定到**哪里**找特定的 `X` 包。

这些问题可通过查询各项目文件（`Project.toml` 或 `JuliaProject.toml`）、清单文件（`Manifest.toml` 或 `JuliaManifest.toml`），或是源文件的文件夹列在[`LOAD_PATH`](@ref) 中的项目环境解决。


## 包的联合

大多数时候，一个包可以通过它的名字唯一确定。但有时在一个项目中，可能需要使用两个有着相同名字的不同的包。尽管你可以通过重命名其中一个包来解决这个问题，但在一个大型的、共享的代码库中被迫做这件事可能是有高度破坏性的。相反，Julia的包加载机制允许相同的包名在一个应用的不同部分指向不同的包。

Julia 支持联合的包管理，这意味着多个独立的部分可以维护公有包、私有包以及包的注册表，并且项目可以依赖于一系列来自不同注册表的公有包和私有包。您也可以使用一组通用工具和工作流（workflow）来安装和管理来自各种注册表的包。Julia 附带的 `Pkg` 软件包管理器允许安装和管理项目的依赖项，它会帮助创建并操作项目文件（其描述了项目所依赖的其他项目）和清单文件（其为项目完整依赖库的确切版本的快照）。

联合管理的一个可能后果是没有包命名的中央权限。不同组织可以使用相同的名称来引用不相关的包。这并不是没有可能的，因为这些组织可能没有协作，甚至不知道彼此。由于缺乏中央命名权限，单个项目可能最终依赖着具有相同名称的不同包。Julia 的包加载机制不要求包名称是全局唯一的，即使在单个项目的依赖关系图中也是如此。相反，包由[通用唯一标识符](https://en.wikipedia.org/wiki/Universally_unique_identifier) （UUID）进行标识，它在每个包创建时进行分配。通常，您不必直接使用这些有点麻烦的 128 位标识符，因为 `Pkg` 将负责生成和跟踪它们。但是，这些 UUID 为问题*「`X` 所指的包是什么？」*提供了确定的答案

由于去中心化的命名问题有些抽象，因此可以通过具体情境来理解问题。假设你正在开发一个名为 `App` 的应用程序，它使用两个包：`Pub` 和 `Priv`。`Priv` 是你创建的私有包，而 `Pub` 是你使用但不控制的公共包。当你创建 `Priv` 时，没有名为 `Priv` 的公共包。然而，随后一个名为 `Priv` 的不相关软件包发布并变得流行起来，而且 `Pub` 包已经开始使用它了。因此，当你下次升级 `Pub` 以获取最新的错误修复和特性时，`App` 将依赖于两个名为 `Priv` 的不同包——尽管你除了升级之外什么都没做。`App` 直接依赖于你的私有 `Priv` 包，以及通过 `Pub` 在新的公共 `Priv` 包上的间接依赖。由于这两个 `Priv` 包是不同的，但是 `App` 继续正常工作依赖于他们两者，因此表达式 `import Priv` 必须引用不同的 `Priv` 包，具体取决于它是出现在 `App` 的代码中还是出现在 `Pub` 的代码中。为了处理这种情况，Julia 的包加载机制通过 UUID 区分两个 `Priv` 包并根据它（调用 `import` 的模块）的上下文选择正确的包。这种区分的工作原理取决于环境，如以下各节所述。

## 环境（Environments）

**环境**决定了 `import X` 和 `using X` 语句在不同的代码上下文中的含义以及什么文件会被加载。Julia 有两类环境（environment）：

1. **项目环境（project environment）**是包含项目文件和清单文件（可选）的目录，并形成一个*显式环境*。项目文件确定项目的直接依赖项的名称和标识。清单文件（如果存在）提供完整的依赖关系图，包括所有直接和间接依赖关系，每个依赖的确切版本以及定位和加载正确版本的足够信息。
2. **包目录（package directory）**是包含一组包的源码树子目录的目录，并形成一个*隐式环境*。如果 `X` 是包目录的子目录并且存在 `X/src/X.jl`，那么程序包 `X` 在包目录环境中可用，而 `X/src/X.jl` 是加载它使用的源文件。

这些环境可以混合并用来创建**堆栈环境（stacked environment）**：是一组有序的项目环境和包目录，重叠为一个复合环境。然后，结合优先级规则和可见性规则，确定哪些包是可用的以及从哪里加载它们。例如，Julia 的负载路径是一个堆栈环境。

这些环境各有不同的用途：

* 项目环境提供**可迁移性**。通过将项目环境以及项目源代码的其余部分存放到版本控制（例如一个 git 存储库），您可以重现项目的确切状态和所有依赖项。特别是，清单文件会记录每个依赖项的确切版本，而依赖项由其源码树的加密哈希值标识；这使得 `Pkg` 可以检索出正确的版本，并确保你正在运行准确的已记录的所有依赖项的代码。
* 当不需要完全仔细跟踪的项目环境时，包目录更**方便**。当你想要把一组包放在某处，并且希望能够直接使用它们而不必为之创建项目环境时，包目录是很实用的。
* 堆栈环境允许向基本环境**添加**工具。您可以将包含开发工具在内的环境堆到堆栈环境的末尾，使它们在 REPL 和脚本中可用，但在包内部不可用。

从更高层次上，每个环境在概念上定义了三个映射：roots、graph 和 paths。当解析 `import X` 的含义时，roots 和 graph 映射用于确定 `X` 的身份，同时 paths 映射用于定位 `X` 的源代码。这三个映射的具体作用是：

- **roots:** `name::Symbol` ⟶ `uuid::UUID`

   环境的 roots 映射将包名称分配给UUID，以获取环境可用于主项目的所有顶级依赖项（即可以在 `Main` 中加载的那些依赖项）。当 Julia 在主项目中遇到 `import X` 时，它会将 `X` 的标识作为 `roots[:X]`。

- **graph:** `context::UUID` ⟶ `name::Symbol` ⟶ `uuid::UUID`

   环境的 graph 是一个多级映射，它为每个 `context` UUID 分配一个从名称到 UUID 的映射——类似于 roots 映射，但专一于那个 `context`。当 Julia 在 UUID 为 `context` 的包代码中运行到 `import X` 时，它会将 `X` 的标识看作为 `graph[context][:X]`。正是因为如此，`import X` 可以根据 `context` 引用不同的包。

- **paths:** `uuid::UUID` × `name::Symbol` ⟶ `path::String`

   paths 映射会为每个包分配 UUID-name 对，即该包的入口点源文件的位置。在 `import X` 中，`X` 的标识已经通过 roots 或 graph 解析为 UUID（取决于它是从主项目还是从依赖项加载），Julia 确定要加载哪个文件来获取 `X` 是通过在环境中查找 `paths[uuid,:X]`。要包含此文件应该定义一个名为 `X` 的模块。一旦加载了此包，任何解析为相同的 `uuid` 的后续导入只会创建一个到同一个已加载的包模块的绑定。

每种环境都以不同的方式定义这三种映射，详见以下各节。

!!! note
    为了清楚地说明，本章中的示例包括 roots、graph 和 paths 的完整数据结构。但是，为了提高效率，Julia 的包加载代码并没有显式地创建它们。相反，加载一个给定包只会简单地计算所需的结构。

### 项目环境（Project environments）

项目环境由包含名为 `Project.toml` 的项目文件的目录以及名为 `Manifest.toml` 的清单文件（可选）确定。这些文件也可以命名为 `JuliaProject.toml` 和 `JuliaManifest.toml`，此时 `Project.toml` 和 `Manifest.toml` 被忽略——这允许项目与可能需要名为 `Project.toml` 和 `Manifest.toml` 文件的其他重要工具共存。但是对于纯 Julia 项目，名称 `Project.toml` 和 `Manifest.toml` 是首选。

项目环境的 roots、graph 和 paths 映射定义如下：

**roots 映射** 在环境中由其项目文件的内容决定，特别是它的顶级 `name` 和 `uuid` 条目及其 `[deps]` 部分（全部是可选的）。考虑以下一个假想的应用程序 `App` 的示例项目文件，如先前所述：

```toml
name = "App"
uuid = "8f986787-14fe-4607-ba5d-fbff2944afa9"

[deps]
Priv = "ba13f791-ae1d-465a-978b-69c3ad90f72b"
Pub  = "c07ecb7d-0dc9-4db7-8803-fadaaeaf08e1"
```

如果将它表示为 Julia 字典，那么这个项目文件意味着以下 roots 映射：

```julia
roots = Dict(
    :App  => UUID("8f986787-14fe-4607-ba5d-fbff2944afa9"),
    :Priv => UUID("ba13f791-ae1d-465a-978b-69c3ad90f72b"),
    :Pub  => UUID("c07ecb7d-0dc9-4db7-8803-fadaaeaf08e1"),
)
```

基于这个 root 映射，在 `App` 的代码中，语句 `import Priv` 将使 Julia 查找 `roots[:Priv]`，这将得到 `ba13f791-ae1d-465a-978b-69c3ad90f72b`，也就是要在这一部分加载的 `Priv` 包的 UUID。当主应用程序解释运行到 `import Priv` 时，此 UUID 标识了要加载和使用的 `Priv` 包。

**依赖图（dependency graph）** 在项目环境中其清单文件的内容决定，如果其存在。如果没有清单文件，则 graph 为空。清单文件包含项目的直接或间接依赖项的节（stanza）。对于每个依赖项，该文件列出该包的 UUID 以及源码树的哈希值或源代码的显式路径。考虑以下 `App` 的示例清单文件：

```toml
[[Priv]] # 私有的那个
deps = ["Pub", "Zebra"]
uuid = "ba13f791-ae1d-465a-978b-69c3ad90f72b"
path = "deps/Priv"

[[Priv]] # 公共的那个
uuid = "2d15fe94-a1f7-436c-a4d8-07a9a496e01c"
git-tree-sha1 = "1bf63d3be994fe83456a03b874b409cfd59a6373"
version = "0.1.5"

[[Pub]]
uuid = "c07ecb7d-0dc9-4db7-8803-fadaaeaf08e1"
git-tree-sha1 = "9ebd50e2b0dd1e110e842df3b433cb5869b0dd38"
version = "2.1.4"

  [Pub.deps]
  Priv = "2d15fe94-a1f7-436c-a4d8-07a9a496e01c"
  Zebra = "f7a24cb4-21fc-4002-ac70-f0e3a0dd3f62"

[[Zebra]]
uuid = "f7a24cb4-21fc-4002-ac70-f0e3a0dd3f62"
git-tree-sha1 = "e808e36a5d7173974b90a15a353b564f3494092f"
version = "3.4.2"
```

这个清单文件描述了 `App` 项目可能的完整依赖关系图：

- 应用程序使用两个名为 `Priv` 的不同包，一个作为根依赖项的私有包，以及一个通过 `Pub` 作为间接依赖项的公共包。它们通过不同 UUID 来区分，并且有不同的依赖项：
  * 私有的 `Priv` 依赖于 `Pub` 和 `Zebra` 包。
  * 公有的 `Priv` 没有依赖关系。
- 该应用程序还依赖于 `Pub` 包，而后者依赖于公有的 `Priv` 以及私有的 `Priv` 包所依赖的那个 `Zebra` 包。


此依赖图以字典表示后如下所示：

```julia
graph = Dict(
    # Priv——私有的那个:
    UUID("ba13f791-ae1d-465a-978b-69c3ad90f72b") => Dict(
        :Pub   => UUID("c07ecb7d-0dc9-4db7-8803-fadaaeaf08e1"),
        :Zebra => UUID("f7a24cb4-21fc-4002-ac70-f0e3a0dd3f62"),
    ),
    # Priv——公共的那个:
    UUID("2d15fe94-a1f7-436c-a4d8-07a9a496e01c") => Dict(),
    # Pub:
    UUID("c07ecb7d-0dc9-4db7-8803-fadaaeaf08e1") => Dict(
        :Priv  => UUID("2d15fe94-a1f7-436c-a4d8-07a9a496e01c"),
        :Zebra => UUID("f7a24cb4-21fc-4002-ac70-f0e3a0dd3f62"),
    ),
    # Zebra:
    UUID("f7a24cb4-21fc-4002-ac70-f0e3a0dd3f62") => Dict(),
)
```

给定这个依赖图，当 Julia 看到 `Pub` 包中的 `import Priv` ——它有 UUID`c07ecb7d-0dc9-4db7-8803-fadaaeaf08e1` 时，它会查找：

```julia
graph[UUID("c07ecb7d-0dc9-4db7-8803-fadaaeaf08e1")][:Priv]
```

会得到 `2d15fe94-a1f7-436c-a4d8-07a9a496e01c`，这意味着 `Pub` 包中的内容，`import Priv` 指代的是公有的 `Priv` 内容，而非应用程序直接依赖的私有包。这也是为何 `Priv` 在主项目中可指代不同的包，而不像其在某个依赖包中另有含义。在包生态中，该特性允许重名的出现。

如果在 `App` 主代码库中 `import Zebra` 会如何？因为`Zebra` 不存在于项目文件，即使它 *确实* 存在于清单文件中，其导入会是失败的。此外，`import Zebra` 这个行为若发生在公有的 `Priv` 包——UUID 为 `2d15fe94-a1f7-436c-a4d8-07a9a496e01c` 的包中，同样会失败。因为公有的 `Priv` 包未在清单文件中声明依赖，故而无法加载包。仅有在清单文件：`Pub` 包和一个 `Priv` 包中作为显式依赖的包可用于加载 `Zebra`。

项目环境的 **路径映射** 从 manifest 文件中提取得到。而包的路径 `uuid` 和名称 `X` 则 (循序) 依据这些规则确定。

1. 如果目录中的项目文件与要求的 `uuid` 以及名称 `X` 匹配，那么可能出现以下情况的一种：
  - 若该文件具有顶层 `路径` 入口，则 `uuid` 会被映射到该路径，文件的执行与包含项目文件的目录相关。
  - 此外，`uuid` 依照包含项目文件的目录，映射至与`src/X.jl`。
2. 若非上述情况，且项目文件具有对应的清单文件，且该清单文件包含匹配 `uuid` 的节（stanza），那么：
  - 若其具有一个 `路径` 入口，则使用该路径（与包含清单文件的目录相关）。
  - 若其具有一个 `git-tree-sha1` 入口，计算一个确定的 `uuid` 与 `git-tree-sha1` 函数——我们把这个函数称为 `slug`——并在每个 Julia `DEPOT_PATH` 的全局序列中的目录查询名为 `packages/X/$slug` 的目录。使用存在的第一个此类目录。

若某些结果成功，源码入口点的路径会是这些结果中的某个，结果的相对路径+`src/X.jl`；否则，`uuid` 不存在路径映射。当加载 `X` 时，如果没找到源码路径，查找即告失败，用户可能会被提示安装适当的包版本或采取其他纠正措施（例如，将 `X` 声明为某种依赖性）。

在上述样例清单文件中，为找到首个 `Priv` 包的路径——该包 UUID 为 `ba13f791-ae1d-465a-978b-69c3ad90f72b`——Julia 寻找其在清单中的节（stanza）。发现其有 路径` 入口，查看 `App` 项目目录中相关的 `deps/Priv`——不妨设`App` 代码在 `/home/me/projects/App` 中—则 Julia 发现 `/home/me/projects/App/deps/Priv` 存在，并因此从中加载  `Priv`。

If, on the other hand, Julia was loading the *other* `Priv` package—the one with UUID `2d15fe94-a1f7-436c-a4d8-07a9a496e01c`—it finds its stanza in the manifest, see that it does *not* have a `path` entry, but that it does have a `git-tree-sha1` entry. It then computes the `slug` for this UUID/SHA-1 pair, which is `HDkrT` (the exact details of this computation aren't important, but it is consistent and deterministic). This means that the path to this `Priv` package will be `packages/Priv/HDkrT/src/Priv.jl` in one of the package depots. Suppose the contents of `DEPOT_PATH` is `["/home/me/.julia", "/usr/local/julia"]`, then Julia will look at the following paths to see if they exist:

1. `/home/me/.julia/packages/Priv/HDkrT`
2. `/usr/local/julia/packages/Priv/HDkrT`

Julia uses the first of these that exists to try to load the public `Priv` package from the file `packages/Priv/HDKrT/src/Priv.jl` in the depot where it was found.

Here is a representation of a possible paths map for our example `App` project environment,
as provided in the Manifest given above for the dependency graph,
after searching the local file system:

```julia
paths = Dict(
    # Priv – the private one:
    (UUID("ba13f791-ae1d-465a-978b-69c3ad90f72b"), :Priv) =>
        # relative entry-point inside `App` repo:
        "/home/me/projects/App/deps/Priv/src/Priv.jl",
    # Priv – the public one:
    (UUID("2d15fe94-a1f7-436c-a4d8-07a9a496e01c"), :Priv) =>
        # package installed in the system depot:
        "/usr/local/julia/packages/Priv/HDkr/src/Priv.jl",
    # Pub:
    (UUID("c07ecb7d-0dc9-4db7-8803-fadaaeaf08e1"), :Pub) =>
        # package installed in the user depot:
        "/home/me/.julia/packages/Pub/oKpw/src/Pub.jl",
    # Zebra:
    (UUID("f7a24cb4-21fc-4002-ac70-f0e3a0dd3f62"), :Zebra) =>
        # package installed in the system depot:
        "/usr/local/julia/packages/Zebra/me9k/src/Zebra.jl",
)
```

This example map includes three different kinds of package locations (the first and third are part of the default load path):

1. The private `Priv` package is "[vendored](https://stackoverflow.com/a/35109534)" inside the `App` repository.
2. The public `Priv` and `Zebra` packages are in the system depot, where packages installed and managed by the system administrator live. These are available to all users on the system.
3. The `Pub` package is in the user depot, where packages installed by the user live. These are only available to the user who installed them.


### 包目录

包目录提供一种更简单的，不能处理名称冲突的环境。

- `X.jl`
- `X/src/X.jl`
- `X.jl/src/X.jl`

Which dependencies a package in a package directory can import depends on whether the package contains a project file:

* If it has a project file, it can only import those packages which are identified in the `[deps]` section of the project file.
* If it does not have a project file, it can import any top-level package—i.e. the same packages that can be loaded in `Main` or the REPL.

**The roots map** is determined by examining the contents of the package directory to generate a list of all packages that exist.
Additionally, a UUID will be assigned to each entry as follows: For a given package found inside the folder `X`...

1. If `X/Project.toml` exists and has a `uuid` entry, then `uuid` is that value.
2. If `X/Project.toml` exists and but does *not* have a top-level UUID entry, `uuid` is a dummy UUID generated by hashing the canonical (real) path to `X/Project.toml`.
3. Otherwise (if `Project.toml` does not exist), then `uuid` is the all-zero [nil UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier#Nil_UUID).

**The dependency graph** of a project directory is determined by the presence and contents of project files in the subdirectory of each package. The rules are:

- If a package subdirectory has no project file, then it is omitted from graph and import statements in its code are treated as top-level, the same as the main project and REPL.
- If a package subdirectory has a project file, then the graph entry for its UUID is the `[deps]` map of the project file, which is considered to be empty if the section is absent.

As an example, suppose a package directory has the following structure and content:

```
Aardvark/
    src/Aardvark.jl:
        import Bobcat
        import Cobra

Bobcat/
    Project.toml:
        [deps]
        Cobra = "4725e24d-f727-424b-bca0-c4307a3456fa"
        Dingo = "7a7925be-828c-4418-bbeb-bac8dfc843bc"

    src/Bobcat.jl:
        import Cobra
        import Dingo

Cobra/
    Project.toml:
        uuid = "4725e24d-f727-424b-bca0-c4307a3456fa"
        [deps]
        Dingo = "7a7925be-828c-4418-bbeb-bac8dfc843bc"

    src/Cobra.jl:
        import Dingo

Dingo/
    Project.toml:
        uuid = "7a7925be-828c-4418-bbeb-bac8dfc843bc"

    src/Dingo.jl:
        # no imports
```

Here is a corresponding roots structure, represented as a dictionary:

```julia
roots = Dict(
    :Aardvark => UUID("00000000-0000-0000-0000-000000000000"), # no project file, nil UUID
    :Bobcat   => UUID("85ad11c7-31f6-5d08-84db-0a4914d4cadf"), # dummy UUID based on path
    :Cobra    => UUID("4725e24d-f727-424b-bca0-c4307a3456fa"), # UUID from project file
    :Dingo    => UUID("7a7925be-828c-4418-bbeb-bac8dfc843bc"), # UUID from project file
)
```

Here is the corresponding graph structure, represented as a dictionary:

```julia
graph = Dict(
    # Bobcat:
    UUID("85ad11c7-31f6-5d08-84db-0a4914d4cadf") => Dict(
        :Cobra => UUID("4725e24d-f727-424b-bca0-c4307a3456fa"),
        :Dingo => UUID("7a7925be-828c-4418-bbeb-bac8dfc843bc"),
    ),
    # Cobra:
    UUID("4725e24d-f727-424b-bca0-c4307a3456fa") => Dict(
        :Dingo => UUID("7a7925be-828c-4418-bbeb-bac8dfc843bc"),
    ),
    # Dingo:
    UUID("7a7925be-828c-4418-bbeb-bac8dfc843bc") => Dict(),
)
```

A few general rules to note:

1. A package without a project file can depend on any top-level dependency, and since every package in a package directory is available at the top-level, it can import all packages in the environment.
2. A package with a project file cannot depend on one without a project file since packages with project files can only load packages in `graph` and packages without project files do not appear in `graph`.
3. A package with a project file but no explicit UUID can only be depended on by packages without project files since dummy UUIDs assigned to these packages are strictly internal.

Observe the following specific instances of these rules in our example:

* `Aardvark` can import on any of `Bobcat`, `Cobra` or `Dingo`; it does import `Bobcat` and `Cobra`.
* `Bobcat` can and does import both `Cobra` and `Dingo`, which both have project files with UUIDs and are declared as dependencies in `Bobcat`'s `[deps]` section.
* `Bobcat` cannot depend on `Aardvark` since `Aardvark` does not have a project file.
* `Cobra` can and does import `Dingo`, which has a project file and UUID, and is declared as a dependency in `Cobra`'s  `[deps]` section.
* `Cobra` cannot depend on `Aardvark` or `Bobcat` since neither have real UUIDs.
* `Dingo` cannot import anything because it has a project file without a `[deps]` section.

**The paths map** in a package directory is simple: it maps subdirectory names to their corresponding entry-point paths. In other words, if the path to our example project directory is `/home/me/animals` then the `paths` map could be represented by this dictionary:

```julia
paths = Dict(
    (UUID("00000000-0000-0000-0000-000000000000"), :Aardvark) =>
        "/home/me/AnimalPackages/Aardvark/src/Aardvark.jl",
    (UUID("85ad11c7-31f6-5d08-84db-0a4914d4cadf"), :Bobcat) =>
        "/home/me/AnimalPackages/Bobcat/src/Bobcat.jl",
    (UUID("4725e24d-f727-424b-bca0-c4307a3456fa"), :Cobra) =>
        "/home/me/AnimalPackages/Cobra/src/Cobra.jl",
    (UUID("7a7925be-828c-4418-bbeb-bac8dfc843bc"), :Dingo) =>
        "/home/me/AnimalPackages/Dingo/src/Dingo.jl",
)
```

Since all packages in a package directory environment are, by definition, subdirectories with the expected entry-point files, their `paths` map entries always have this form.

### Environment stacks

The third and final kind of environment is one that combines other environments by overlaying several of them, making the packages in each available in a single composite environment. These composite environments are called *environment stacks*. The Julia `LOAD_PATH` global defines an environment stack—the environment in which the Julia process operates. If you want your Julia process to have access only to the packages in one project or package directory, make it the only entry in `LOAD_PATH`. It is often quite useful, however, to have access to some of your favorite tools—standard libraries, profilers, debuggers, personal utilities, etc.—even if they are not dependencies of the project you're working on. By adding an environment containing these tools to the load path, you immediately have access to them in top-level code without needing to add them to your project.

The mechanism for combining the roots, graph and paths data structures of the components of an environment stack is simple: they are merged as dictionaries, favoring earlier entries over later ones in the case of key collisions. In other words, if we have `stack = [env₁, env₂, …]` then we have:

```julia
roots = reduce(merge, reverse([roots₁, roots₂, …]))
graph = reduce(merge, reverse([graph₁, graph₂, …]))
paths = reduce(merge, reverse([paths₁, paths₂, …]))
```

The subscripted `rootsᵢ`, `graphᵢ` and `pathsᵢ` variables correspond to the subscripted environments, `envᵢ`, contained in `stack`. The `reverse` is present because `merge` favors the last argument rather than first when there are collisions between keys in its argument dictionaries. There are a couple of noteworthy features of this design:

1. The *primary environment*—i.e. the first environment in a stack—is faithfully embedded in a stacked environment. The full dependency graph of the first environment in a stack is guaranteed to be included intact in the stacked environment including the same versions of all dependencies.
2. Packages in non-primary environments can end up using incompatible versions of their dependencies even if their own environments are entirely compatible. This can happen when one of their dependencies is shadowed by a version in an earlier environment in the stack (either by graph or path, or both).

Since the primary environment is typically the environment of a project you're working on, while environments later in the stack contain additional tools, this is the right trade-off: it's better to break your development tools but keep the project working. When such incompatibilities occur, you'll typically want to upgrade your dev tools to versions that are compatible with the main project.

## 总结

Federated package management and precise software reproducibility are difficult but worthy goals in a package system. In combination, these goals lead to a more complex package loading mechanism than most dynamic languages have, but it also yields scalability and reproducibility that is more commonly associated with static languages. Typically, Julia users should be able to use the built-in package manager to manage their projects without needing a precise understanding of these interactions. A call to `Pkg.add("X")` will add to the appropriate project and manifest files, selected via `Pkg.activate("Y")`, so that a future call to `import X` will load `X` without further thought.
