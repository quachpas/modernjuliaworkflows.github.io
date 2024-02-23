+++
title = "Sharing your code"
ignore_cache = true
+++

<!-- Setup -->

```!
# hideall
if isdir(sitepath("MyAwesomePackage"))
    rm(sitepath("MyAwesomePackage"); recursive=true)
end
```

\activate{}

# Sharing your code

In this post, you will learn about tools to initialize, structure and distribute Julia packages.

\toc

## Setup

A vast majority of Julia packages are hosted on [GitHub](https://github.com/) (although less common, other options like [GitLab](https://gitlab.com/) are also possible).
GitHub is a platform for collaborative software development, based on the version control system [Git](https://git-scm.com/).
If you are unfamiliar with these technologies, check out the [GitHub documentation](https://docs.github.com/en/get-started/quickstart).

The first step is therefore [creating](https://github.com/new) an empty GitHub repository.
You should try to follow [package naming guidelines](https://pkgdocs.julialang.org/v1/creating-packages/#Package-naming-guidelines) and add a ".jl" extension at the end, like so: "MyAwesomePackage.jl".
Do not insert any files like `README.md`, `.gitignore` or `LICENSE.md`, this will be done for you in the next step.

Indeed, we can leverage [PkgTemplates.jl](https://github.com/JuliaCI/PkgTemplates.jl) to automate package creation (like `]generate` from Pkg.jl but on steroids).
The following code gives you a basic file structure to start with:

```>pkgtemplates
using PkgTemplates
dir = Utils.path(:site)  # replace with the folder of your choice
t = Template(dir=dir, user="myuser", interactive=false);  
t("MyAwesomePackage")
```

Then, you simply need to push this new folder to the remote repository <https://github.com/myuser/MyAwesomePackage.jl>, and you're ready to go.
The rest of this post will explain to you what each part of this folder does, and how to bend them to your will.

To work on the package further, we develop it into the current environment and import it:

```>using-awesome
using Pkg
Pkg.develop(path=sitepath("MyAwesomePackage"))  # ignore sitepath
using MyAwesomePackage
```

## GitHub Actions

The most useful aspect of PkgTemplates.jl is that it automatically generates workflows for [GitHub Actions](https://docs.github.com/en/actions/quickstart).
These are stored as YAML files in `.github/workflows`, with a slightly convoluted syntax that you don't need to fully understand.
For instance, the file `CI.yml` contains instructions that execute the tests of your package (see below) for each pull request, tag or push to the `main` branch.
This is done on a GitHub server and should theoretically cost you money, but your GitHub repository is public, you get an unlimited workflow budget for free.

More workflows and functionalities are available through optional [plugins](https://juliaci.github.io/PkgTemplates.jl/stable/user/#Plugins-1).
The interactive setting `Template(..., interactive=true)` allows you to select the ones you want for a given package.
Otherwise, you will get the [default selection](https://juliaci.github.io/PkgTemplates.jl/stable/user/#Default-Plugins), which you are encouraged to look at.

## Testing

The purpose of the `test` subfolder in your package is [unit testing](https://docs.julialang.org/en/v1/stdlib/Test/): automatically checking that your code behaves the way you want it to.
For instance, if you write your own square root function, you may want to test that it gives the correct results for positive numbers, and errors for negative numbers.

```>sqrt
using Test

@test sqrt(4) ≈ 2

@testset "Invalid inputs" begin
    @test_throws DomainError sqrt(-1)
    @test_throws MethodError sqrt("abc")
end;
```

Such tests belong in `test/runtests.jl`, and they are executed with the `]test` command (in the REPL's Pkg mode).
Unit testing may seem rather naive, or even superfluous, but as your code grows more complex, it becomes easier to break something without noticing.
Testing each part separately will increase the reliability of the software you write.

At some point, your package may require [test-specific dependencies](https://pkgdocs.julialang.org/v1/creating-packages/#Adding-tests-to-the-package).
This often happens when you need to test compatibility with another package, on which you do not depend for the source code itself.
Or it may simply be due to testing-specific packages like the ones we will encounter below.
For interactive testing work, use [TestEnv.jl](https://github.com/JuliaTesting/TestEnv.jl) to activate the full test environment (faster than running `]test` repeatedly).

\advanced{

If you want to have more control over your tests, you can try

* [ReferenceTests.jl](https://github.com/JuliaTesting/ReferenceTests.jl) to compare function outputs with reference files.
* [ReTest.jl](https://github.com/JuliaTesting/ReTest.jl) to define tests next to the source code and control their execution.
* [TestItemRunner.jl](https://github.com/julia-vscode/TestItemRunner.jl) to leverage the testing interface of VSCode.

}

## Style

To make your code easy to read, it is essential to follow a consistent set of guidelines.
The official [style guide](https://docs.julialang.org/en/v1/manual/style-guide/) is very short, so most people use third party style guides like [BlueStyle](https://github.com/JuliaDiff/BlueStyle) or [SciMLStyle](https://github.com/SciML/SciMLStyle).

[JuliaFormatter.jl](https://github.com/domluna/JuliaFormatter.jl) is an automated formatter for Julia files which can help you enforce the style guide of your choice.
Just add a file `.JuliaFormatter.toml` at the root of your repository, containing a single line like

```toml
style = "blue"
```

Then, the package directory will be formatted in the BlueStyle whenever you call

```>format
using JuliaFormatter
JuliaFormatter.format(MyAwesomePackage)
```

\vscode{

The [default formatter](https://www.julia-vscode.org/docs/stable/userguide/formatter/) falls back on JuliaFormatter.jl.

}

\advanced{

You can format code automatically in GitHub pull requests with the [`julia-format` action](https://github.com/julia-actions/julia-format).

}

## Code quality

Of course, there is more to code quality than just formatting.
[Aqua.jl](https://github.com/JuliaTesting/Aqua.jl) provides a set of routines that examine other aspects of your package, from unused dependencies to ambiguous methods.
It is usually a good idea to include the following in your tests:

```>aqua
using Aqua, MyAwesomePackage
Aqua.test_all(MyAwesomePackage);
```

Meanwhile, [JET.jl](https://github.com/aviatesk/JET.jl) is a complementary tool, similar to a static linter.
Here we focus on its [error analysis](https://aviatesk.github.io/JET.jl/stable/jetanalysis/), which can detect errors or typos without even running the code by leveraging type inference.
You can either use it in report mode (with a nice [VSCode display](https://www.julia-vscode.org/docs/stable/userguide/linter/#Runtime-diagnostics)) or in test mode as follows:

```>jet
using JET, MyAwesomePackage
JET.report_package(MyAwesomePackage)
JET.test_package(MyAwesomePackage)
```

Note that both Aqua.jl and JET.jl might pick up false positives: refer to their respective documentations for ways to make them less sensitive.

## Documentation

Even if your code does everything it is supposed to, it will be useless to others (and pretty soon to yourself) without proper documentation.
Adding [docstrings](https://docs.julialang.org/en/v1/manual/documentation/) everywhere needs to become a second nature.
This way, readers and users of your code can query them through the REPL help mode.

```!docstring
"""
    myfunc(a, b; kwargs...)

One-line sentence describing the purpose of the function,
just below the (indented) signature.

More details if needed.
"""
function myfunc end;
```

[DocStringExtensions.jl](https://github.com/JuliaDocs/DocStringExtensions.jl) provides a few shortcuts that can speed up docstring creation by taking care of the obvious parts.

However, package documentation is not limited to docstrings.
It can also contain high-level overviews, technical explanations, examples, tutorials, etc.
[Documenter.jl](https://github.com/JuliaDocs/Documenter.jl) allows you to design a website for all of this, based on Markdown files contained in the `docs` subfolder of your package.
Unsurprisingly, its own [documentation](https://documenter.juliadocs.org/stable/) is excellent and will teach you a lot.
To build the documentation locally, just run

```julia-repl
julia> using Pkg; Pkg.activate("docs")

julia> include("docs/make.jl")
```

Then, use [LiveServer.jl](https://github.com/tlienart/LiveServer.jl) from your package folder to visualize and automatically update the website as the code changes (similar to Revise.jl):

```julia-repl
julia> using LiveServer

julia> servedocs()
```

\advanced{

[DocumenterCitations.jl](https://github.com/ali-ramadhan/DocumenterCitations.jl) allows you to insert citations inside the documentation website from a BibTex file.

}

To host the documentation online easily, just select the [`Documenter` plugin](https://juliaci.github.io/PkgTemplates.jl/stable/user/#PkgTemplates.Documenter) from PkgTemplates.jl before creation.
Not only will this fill the `docs` subfolder with the right contents: it will also initialize a [GitHub Actions workflow](https://documenter.juliadocs.org/stable/man/hosting/#gh-pages-Branch) to build and deploy your website on [GitHub pages](https://pages.github.com/).
The only thing left to do is to [select the `gh-pages` branch as source](https://documenter.juliadocs.org/stable/man/hosting/#gh-pages-Branch).

\advanced{

Assuming you are looking for an alternative to Documenter.jl, you can try out [Pollen.jl](https://github.com/lorenzoh/Pollen.jl).
In another category, [Replay.jl](https://github.com/AtelierArith/Replay.jl) allows you to replay instructions entered into your terminal as an ASCII video, which is nice for tutorials.

}

## Versions and registration

The Julia community has adopted [semantic versioning](https://semver.org/), which means every package must have a version, and the version numbering follows strict rules.
The main consequence is that you need to specify [compatibility bounds](https://pkgdocs.julialang.org/v1/compatibility/) for your dependencies: this happens in the `[compat]` section of your `Project.toml`.
To initialize these bounds, use the `]compat` command in the Pkg mode of the REPL, or the package [PackageCompatUI.jl](https://github.com/GunnarFarneback/PackageCompatUI.jl).

As your package lives on, new versions of your dependencies will be released.
The [CompatHelper.jl](https://github.com/JuliaRegistries/CompatHelper.jl)  GitHub Action will help you monitor Julia dependencies and update your `Project.toml` accordingly.
In addition, [Dependabot](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuring-dependabot-version-updates#enabling-dependabot-version-updates) can monitor the dependencies... of your GitHub actions themselves.
But don't worry: both are default plugins in the PkgTemplates.jl setup.

\advanced{

It may also happen that you incorrectly promise compatibility with an old version of a package.
To prevent that, the [julia-downgrade-compat](https://github.com/julia-actions/julia-downgrade-compat) GitHub action tests your package with the oldest possible version of every dependency, and verifies that everything still works.

}

If your package is useful to others in the community, it may be a good idea to register it, that is, make it part of the pool of packages that can be installed with `Pkg.add(MyAwesomePackage)`.
Note that unregistered packages can also be installed by anyone, but the command is slightly different: `Pkg.add(url="https://github.com/myuser/MyAwesomePackage")`.

To register your package, check out the [general registry](https://github.com/JuliaRegistries/General) guidelines.
The [Registrator.jl](https://github.com/JuliaRegistries/Registrator.jl) bot can help you automate the process.
Another handy bot is [TagBot](https://github.com/JuliaRegistries/TagBot), which automatically tags new versions of your package following each release: yet another default plugin in the PkgTemplates.jl setup.
If you have performed the [necessary SSH configuration](https://documenter.juliadocs.org/stable/man/hosting/#travis-ssh), TagBot will also trigger documentation website builds following each release.

\advanced{

If your package is only interesting to you and a small group of collaborators, or if you don't want to make it public, you can still register it by setting up a local registry: see [LocalRegistry.jl](https://github.com/GunnarFarneback/LocalRegistry.jl).

}

## Collaboration and literate programming

Once your package grows big enough, you might need to bring in some help.
Working together on a software project has its own set of challenges, which are partially addressed by a good set of ground rules liks [SciML ColPrac](https://github.com/SciML/ColPrac).
Of course, collaboration goes both ways: if you find a Julia package you really like, you are more than welcome to [contribute](https://julialang.org/contribute/) as well, for example by opening issues or submitting pull requests.

Large-scale software is often hard to grasp, and the code alone may not be very enlightening.
Whether it is for package documentation or to write papers and books, you might want to interleave code with texts, formulas, images and so on.
In addition to the [Pluto.jl](https://github.com/fonsp/Pluto.jl) and [Jupyter](https://jupyter.org/) notebooks, take a look at [Literate.jl](https://github.com/fredrikekre/Literate.jl) to enrich your code with comments and translate it to various formats.
[Quarto](https://quarto.org/) is another cross-language notebook system that supports Python, R and Julia, while [Books.jl](https://github.com/JuliaBooks/Books.jl) is more relevant to draft long documents.

## Reproducibility

Obtaining consistent and reproducible results is an essential part of experimental science.
[DrWatson.jl](https://github.com/JuliaDynamics/DrWatson.jl) is a general toolbox for running and re-running experiments in an orderly fashion.
We now explore a few specific issues that often arise.

A first hurdle is [random number generation](https://docs.julialang.org/en/v1/stdlib/Random/), which is not guaranteed to remain stable across Julia versions.
To ensure that the random streams remain exactly the same, you need to use [StableRNGs.jl](https://github.com/JuliaRandom/StableRNGs.jl).
Another aspect is dataset download and management.
The packages [DataDeps.jl](https://github.com/oxinabox/DataDeps.jl) and [ArtifactUtils.jl](https://github.com/JuliaPackaging/ArtifactUtils.jl) can help you bundle non-code elements with your package.
A third thing to consider is proper citation and versioning.
Giving your package a with [Zenodo](https://zenodo.org/) ensures that everyone can properly cite it in scientific publications.
Similarly, your papers should cite the packages you use as dependencies: [PkgCite.jl](https://github.com/SebastianM-C/PkgCite.jl) will help with that.

## Interoperability

Making packages play nice with one another is a key goal of the Julia ecosystem.
Since Julia 1.9, this can be done with [package extensions](https://pkgdocs.julialang.org/v1/creating-packages/#Conditional-loading-of-code-in-packages-(Extensions)), which override specific behaviors based on the presence of a given package in the environment.
To preserve compatibility with earlier Julia versions, [PackageExtensionTools.jl](https://github.com/cjdoris/PackageExtensionTools.jl) is the way to go.

Furthermore, the Julia ecosystem as a whole plays nice with other programming languages too.
[C and Fortran](https://docs.julialang.org/en/v1/manual/calling-c-and-fortran-code/) are natively supported.
Python can be easily interfaced with the combination of [CondaPkg.jl](https://github.com/cjdoris/CondaPkg.jl) and [PythonCall.jl](https://github.com/cjdoris/PythonCall.jl).
Other language compatibility packages can be found in the [JuliaInterop](https://github.com/JuliaInterop) organization, like [RCall.jl](https://github.com/JuliaInterop/RCall.jl) or [Cxx.jl](https://github.com/JuliaInterop/Cxx.jl).

\advanced{

Some package developers may need to define what kind of behavior they expect from a certain type, or what a certain method should do.
When writing it in the documentation is not enough, a formal testable specification becomes necessary.
This problem of "interfaces" does not yet have a definitive solution in Julia, but several options have been proposed: [Interfaces.jl](https://github.com/rafaqz/Interfaces.jl), [RequiredInterfaces.jl](https://github.com/Seelengrab/RequiredInterfaces.jl) and [PropCheck.jl](https://github.com/Seelengrab/PropCheck.jl) are all worth checking out.
    
}

<!-- Clean up -->

```!cleanup
Pkg.rm("MyAwesomePackage")  # hide
```