---
title: Processing 40 TB of code from ~10 million projects with a dedicated server and Go for $100
date: 2019-09-30
---

<link rel="stylesheet" href="/static/an-informal-survey/table.css" />
<link rel="stylesheet" href="/static/an-informal-survey/jquery.dataTables.min.css" />
<script src="/static/an-informal-survey/jquery-3.1.1.min.js"></script>
<script src="/static/an-informal-survey/jquery.dataTables.min.js"></script>
<script src="/static/an-informal-survey/table.js"></script>

The command line tool I created [Sloc Cloc and Code (`scc`)](https://github.com/boyter/scc/) (which is now modified and maintained by many other excellent people) counts lines of code, comments and makes a complexity estimate for files inside a directory. The latter is something you need a good sample size to make good use of. The way it works is that it counts branch statements in code. However what does that actually mean? For example "This file has a complexity of 10" is not very useful without some context. To solve this issue I thought I would try to run scc at all the source code I could get my hands on. This would also allow me to see if there are any edge cases I didn't consider in the tool itself. A brute force Q/A trial by fire.

However if I am going to run it over all that code, which is going to be computationally expensive I may as well try to get some use out of it. As such I decided to record everything as I went and see if I could get something interesting in the end, hence this post.

In short I downloaded and processed a lot of code using `scc`. The raw numbers include,

 - **9,985,051** total repositories
 - **9,100,083** repositories with at least 1 identified file
 - **884,968** empty repositories (those with no files)
 - **58,389,641** files in all repositories
 - **40,736,530,379,778** bytes processed (40 TB)
 - **1,086,723,618,560** lines identified
 - **816,822,273,469** code lines identified
 - **124,382,152,510** blank lines identified
 - **145,519,192,581** comment lines identified
 - **71,884,867,919** complexity count according to scc rules
 - **2** new bugs raised in scc

Lets get the elephant out of the room first. It was not 10 million projects as the "click bait" title indicates. I was shy by 15,000 so I rounded up. Please forgive me.

It took about 5 weeks to download and run `scc` over the collection of repositories saving all of the data. It took just over 49 hours to crunch the 1 TB of JSON and produce the results below.

## Quicklinks

 - [Methodology](#methodology)
 - [Presenting and Computing Results](#presenting-and-computing-results)
 - [Cost](#cost)
 - [Data Sources](#data-sources)
 - [How many files in a repository?](#how-many-files-in-a-repository)
 - [Whats the project breakdown per language?](#whats-the-project-breakdown-per-language)
 - [How many files in a repository per language?](#how-many-files-in-a-repository-per-language)
 - [How many lines of code are in a typical file per language?
](#how-many-lines-of-code-are-in-a-typical-file-per-language)
 - [Average complexity for file in each language?](#average-complexity-for-file-in-each-language)
 - [Average comments for file in each language?](#average-comments-for-file-in-each-language)
 - [What are the most common filenames?](#what-are-the-most-common-filenames)
 - [How many repositories appear to be missing a license?](#how-many-repositories-appear-to-be-missing-a-license)
 - [Which languages have the most comments?](#which-languages-have-the-most-comments)
 - [How many projects use multiple .gitignore files?](#how-many-projects-use-multiple-gitignore-files)
 - [Which language developers have the biggest potty mouth?](#which-language-developers-have-the-biggest-potty-mouth)
 - [Longest files by lines per language](#longest-files-by-lines-per-language)
 - [Whats the most complex file in each language?](#whats-the-most-complex-file-in-each-language)
 - [Whats the most complex file weighted against lines?](#whats-the-most-complex-file-weighted-against-lines)
 - [Whats the most commented file in each language?](#whats-the-most-commented-file-in-each-language)
 - [How many "pure" projects](#how-many-pure-projects)
 - [Projects with TypeScript but not JavaScript](#projects-with-typescript-but-not-javascript)
 - [Anyone using CoffeeScript and TypeScript?](#anyone-using-coffeescript-and-typescript)
 - [What's the typical path length, broken up by language](#what-s-the-typical-path-length-broken-up-by-language)
 - [YAML or YML?](#yaml-or-yml)
 - [Upper, lower or mixed case?](#upper-lower-or-mixed-case)
 - [Java Factories](#java-factories)
 - [Ignore files](#ignore-files)
 - [Future ideas](#future-ideas)
 - [So why bother?](#so-why-bother)
 - [Raw / Processed Files](#raw-processed-files)

## Methodology

Since I run [searchcode.com](https://searchcode.com/) I already have a collection of over 7,000,000 projects across git, mercurial, subversion and such. So why not try processing them? Working with git is usually the easiest solution so I ignored mercurial and subversion this time and exported the full list of git projects. Turns out I actually had 12 million git repositories being tracked, and I should probably update the page to reflect that.

So now I have 12 million or so git repositories which I need to download and process.

When you run `scc` you can choose to have it output the results in JSON optionally saving this file to disk like so ` scc --format json --output myfile.json main.go` the results of which look like the following (done for a single file),

{{<highlight json>}}
[
  {
    "Blank": 115,
    "Bytes": 0,
    "Code": 423,
    "Comment": 30,
    "Complexity": 40,
    "Count": 1,
    "Files": [
      {
        "Binary": false,
        "Blank": 115,
        "Bytes": 20396,
        "Callback": null,
        "Code": 423,
        "Comment": 30,
        "Complexity": 40,
        "Content": null,
        "Extension": "go",
        "Filename": "main.go",
        "Hash": null,
        "Language": "Go",
        "Lines": 568,
        "Location": "main.go",
        "PossibleLanguages": [
          "Go"
        ],
        "WeightedComplexity": 0
      }
    ],
    "Lines": 568,
    "Name": "Go",
    "WeightedComplexity": 0
  }
]
{{</highlight>}}

As a larger example here are the results as JSON for the redis project, [redis.json](/static/an-informal-survey/redis.json). All of the results below come from this output without any supporting additional data.

One thing to keep in mind is that `scc` generally categories languages based on extension (except where extension is shared such as Verilog and Coq). As such if someone puts a HTML file with a java extension it will be counted as a java file. Usually this isn't a problem, because why would you ever do that? But of course at scale it is. It is something I discovered later where some files were masquerading as another.

A while back I wrote code to create github badges using `scc` https://boyter.org/posts/sloc-cloc-code-badges/ and since part of that included caching the results, I modified it slightly to cache the results as JSON in AWS S3.

With the badge code working in AWS using lambda, I took the exported list of projects, wrote about 15 lines of python to clean the format so it matched my lambda and make a request to it. I threw in some python multiprocessing to fork 32 processes to call the endpoint reasonably quickly. 

This worked brilliantly. However the problem with the above was firstly the cost, and secondly lambda behind API-Gateway/ALB has a 30 second timeout, so it couldn't process large repositories fast enough. I knew going in that this was not going to be the most cost effective solution but assuming it came close to $100 I would have been willing to live with it. After processing 1 million repositories I checked and the cost was about $60 and since I didn't want a $700 AWS bill I decided to rethink my solution. Keep in mind that was mostly storage and CPU, or what was needed to collect this information. Assuming I processed or exported the data it was going to increase the cost considerably.

Since I was already in AWS the hip solution would be to dump the url's as messages into SQS and pull from it using EC2 instances or fargate for processing. Then scale out like crazy. However despite working in AWS in my day job I have always believed in [taco bell programming](http://widgetsandshit.com/teddziuba/2010/10/taco-bell-programming.html). Besides it was only 12 million repositories so I opted to implement a simpler (cheaper) solution.

Running this computation locally was out due to the abysmal state of the internet in Australia. However I do run [searchcode.com](https://searchcode.com/) fairly lean using dedicated servers from Hetzner. These boxes are quite powerful, i7 Quad Core 32 GB RAM machines often with 2 TB of disk space (usually unused). As such they usually have a lot of spare compute based on how I use them. The front-end varnish box for instance is doing the square root of zero most of the time. So why not run the processing there?

I didn't quite taco bell program the solution using bash and gnu tools. What I did was write a simple [Go program](https://github.com/boyter/scc-data/blob/master/process/main.go) to spin up 32 go-routines which read from a channel, spawned `git` and `scc` subprocesses before writing the JSON output into S3. I actually wrote a Python solution at first, but having to install the pip dependencies on my clean varnish box seemed like a bad idea and it keep breaking in odd ways which I didn't feel like debugging.

Running this on the box produced the following sort of metrics in htop, and the multiple git/scc processes running (scc is not visible in this screen capture) suggested that everything was working as expected, which I confirmed by looking at the results in S3.

![scc-data process load](/static/an-informal-survey/1.png#center)

## Presenting and Computing Results

Having recently read https://mattwarren.org/2017/10/12/Analysing-C-code-on-GitHub-with-BigQuery/ and https://psuter.net/2019/07/07/z-index I thought I would steal the format of those posts with regards to how I wanted to present the information. My twist on the previous however is to add [jQuery DataTables](https://www.datatables.net/) over the large tables of information. This allows you to sort and search/filter results. As such you can click the headers to sort and use the search box to filter. The search box indicates that this is enabled for you. I also added a jump link near these tables so you can skip over them if you like.

The size of the data I needed to process raised another question. How does one process 10 million JSON files taking up just over 1 TB of disk space in an S3 bucket? 

The first thought I had was AWS Athena. But since it's going to cost something like $2.50 USD **per query** for that dataset I quickly looked for an alternative. That said if you kept the data there and processed it infrequently this might still work out to be the cheapest solution.

I posted the question on the company slack because why should I solve issues alone.

One idea raised was to dump the data into a large SQL database. However this means processing the data into the database, then running queries over it multiple times. Plus the structure of the data meant having a few tables which means foreign keys and indexes to ensure some level of performance. This feels wasteful because we could just process the data as we read it from disk in a single pass. I was also worried about building a database this large. With just data it would be over 1 TB in size before adding indexes.

Seeing as I produced the JSON using spare compute, I thought why not process the results the same way? Of course there is one issue with this. Pulling 1 TB of data out of S3 is going to cost a lot. In the event the program crashes that is going to be annoying. To reduce costs I wanted to pull all the files down locally and save them for further processing. Handy tip, you really do not want to store lots of [little files on disk in a single directory](https://boyter.org/2014/02/storing-tracking-managing-billions-tiny-files-file-system-nightmare/). It sucks for runtime performance and file-systems don't like it.

My answer to this was another simple [go program](https://github.com/boyter/scc-data/blob/master/scc-tar/main.go) to pull the files down from S3 then store them in a tar file. I could then process that file over and over. The process itself is done though **very ugly** [go program](https://github.com/boyter/scc-data/blob/master/main.go) to process the tar file so I could re-run my questions without having to trawl S3 over and over. I didn't bother with go-routines for this code for two reasons. The first was that I didn't want to max out my server, so this limits it to a single core for the hard CPU work (another to read the tar file was mostly blocked on the processor). The second being I didn't want to ensure it was thread-safe.

With that done, what I needed was a collection of questions to answer. I used the slack brains trust again and crowd-sourced my work colleagues while I came up with some ideas of my own. The result of this mind meld is included below.

You can find all the code I used to process the JSON including that which pulled it down locally and the [ugly python script](https://github.com/boyter/scc-data/blob/master/convert_json.py) I used to mangle it into something useful for this post https://github.com/boyter/scc-data Please don't comment on it, I know the code is ugly and it is something I wrote as a throwaway as I am unlikely to ever look at it again. 

If you do want to review code I have written to be read by others have a look at the [source of scc](https://github.com/boyter/scc/).


## Cost

I spent about $60 in compute while trialling lambda. I have not looked at the S3 storage cost yet but it should be close to $25 based on the size of the data. However this is not including the transfer costs which I also have not observed. Please note I cleared the bucket when I was finished with it so this is not an ongoing cost for me.

However after time I chose not to use AWS in the end because of cost. So what's the real cost assuming I wanted to do it again?

Well to start all the software used is free as in freedom and open source. So nothing to worry about there.

In my case the cost would be free as I used "free compute" left over from searchcode. Not everyone has compute lying around however. So lets assume another person wishes to replicate this and as such needs to get a server.

It could be done for €73 using the cheapest new dedicated server from Hetzner https://www.hetzner.com/dedicated-rootserver However that cost includes a new server setup fee. If you are willing to wait and poke around on their auction house https://www.hetzner.com/sb you can find much cheaper servers with no setup fee at all. At time of writing I found the below machine which would be perfect for this project and is €25.21 a month with no setup fee.

![hetzner server](/static/an-informal-survey/hetzner.png#center)

Best part though? You can get the VAT removed if you are outside the EU. So give yourself an additional 10% discount on top if you are in this situation.

So were someone to do this from scratch using the same method I eventually went with it would cost under $100 USD to redo the same calculations, and more likely under $50 if you are a little patient or lucky. This also assumes you use the server for less than 2 months which is enough time to download and process. This also includes enough time for you to get a list of 10 million repositories to consider processing as well.

If I were to use a gzipped tar file in my analysis (which isn't that hard to do really) I could even do 10x the repositories on the same machine as the resulting file would still be small enough to fit on the same hard disk. That would take longer to download though which is going to increase the cost for each additional month and this might take multiple months to do. 

Going much larger then 100 million repositories however is going to require some level of sharding. Still it is safe to say that you could redo the entire process I did or larger one on the same hardware without much effort or code changes.

### Data Sources

From the three sources, github, bitbucket and gitlab how many projects came from each? Note that this is counted before excluding empty repositories hence the sum is over the number of repositories that actually form the counts below this point.

| source | count |
| ------ | ----- |
| github | 9,680,111 |
| bitbucket | 248,217 |
| gitlab | 56,722 |

Sorry to the GitHub/Bitbucket/GitLab teams if you read this. If this caused any issues for you (I doubt it) I will shout you a refreshing beverage of your choice should we ever meet.

### How many files in a repository?

On to the real questions. Lets start with a simple one. How many files are in an average repository? Do most projects have a few files in them, or many? By looping over the repositories and counting the number of files we can then drop them in buckets of 1, 2, 10, 12 or however many files it has and plot it out.

![scc-data files per project](/static/an-informal-survey/filesPerProject.png#center)

The X-axis in this case being buckets of the count of files, and Y-axis being the count of projects with that many files. This is limited to projects with less than 1000 files because the plot looks like empty with a thin smear on the left side if you include all the outliers.

As it turns out most repositories have less than 200 files in them. 

However what about plotting this by percentile, or more specifically by 95th percentile so its actually worth looking at? Turns out the vast majority 95% of projects have less than 1,000 files in them. While 90% of them have less than 300 files and 85% have less than 200.

![scc-data files per project 95th](/static/an-informal-survey/filesPerProjectPercentile95.png)

If you want to plot this yourself and do a better job than I here is a link to the raw data [filesPerProject.json](/static/an-informal-survey/filesPerProject.json).

### Whats the project breakdown per language?

This means for each project scanned if a Java file is identified increment the Java count by one and for the second file do nothing. This gives a quick view of what languages are most commonly used. Unsurprisingly the most common languages include markdown, .gitignore and plain text.

Markdown the most commonly used language in any project is included in just over 6 million projects which is about 2/3 of the entire project set. This makes sense since almost all projects include a README.md which is displayed in HTML for repository pages.

The full list is included below.

[skip table to next section](#how-many-files-in-a-repository-per-language)

<div class="table-1"></div>

| language | project count |
| -------- | ----- |
| Markdown | 6,041,849 |
| gitignore | 5,471,254 |
| Plain Text | 3,553,325 |
| JavaScript | 3,408,921 |
| HTML | 3,397,596 |
| CSS | 3,037,754 |
| License | 2,597,330 |
| XML | 2,218,846 |
| JSON | 1,903,569 |
| YAML | 1,860,523 |
| Python | 1,424,505 |
| Shell | 1,395,199 |
| Ruby | 1,386,599 |
| Java | 1,319,091 |
| C Header | 1,259,519 |
| Makefile | 1,215,586 |
| Rakefile | 1,006,022 |
| PHP | 992,617 |
| Properties File | 909,631 |
| SVG | 804,946 |
| C | 791,773 |
| C++ | 715,269 |
| Batch | 645,442 |
| Sass | 535,341 |
| Autoconf | 505,347 |
| Objective C | 503,932 |
| CoffeeScript | 435,133 |
| SQL | 413,739 |
| Perl | 390,775 |
| C# | 380,841 |
| ReStructuredText | 356,922 |
| MSBuild | 354,212 |
| LESS | 281,286 |
| CSV | 275,143 |
| C++ Header | 199,245 |
| CMake | 173,482 |
| Patch | 169,078 |
| Assembly | 165,587 |
| XML Schema | 148,511 |
| m4 | 147,204 |
| JavaServer Pages | 142,605 |
| Vim Script | 134,156 |
| Scala | 132,454 |
| Objective C++ | 127,797 |
| Gradle | 126,899 |
| Module-Definition | 120,181 |
| Bazel | 114,842 |
| R | 113,770 |
| ASP.NET | 111,431 |
| Go Template | 111,263 |
| Document Type Definition | 109,710 |
| Gherkin Specification | 107,187 |
| Smarty Template | 106,668 |
| Jade | 105,903 |
| Happy | 105,631 |
| Emacs Lisp | 105,620 |
| Prolog | 102,792 |
| Go | 99,093 |
| Lua | 98,232 |
| BASH | 95,931 |
| D | 94,400 |
| ActionScript | 93,066 |
| TeX | 84,841 |
| Powershell | 80,347 |
| AWK | 79,870 |
| Groovy | 75,796 |
| LEX | 75,335 |
| nuspec | 72,478 |
| sed | 70,454 |
| Puppet | 67,732 |
| Org | 67,703 |
| Clojure | 67,145 |
| XAML | 65,135 |
| TypeScript | 62,556 |
| Systemd | 58,197 |
| Haskell | 58,162 |
| XCode Config | 57,173 |
| Boo | 55,318 |
| LaTeX | 55,093 |
| Zsh | 55,044 |
| Stylus | 54,412 |
| Razor | 54,102 |
| Handlebars | 51,893 |
| Erlang | 49,475 |
| HEX | 46,442 |
| Protocol Buffers | 45,254 |
| Mustache | 44,633 |
| ASP | 43,114 |
| Extensible Stylesheet Language Transformations | 42,664 |
| Twig Template | 42,273 |
| Processing | 41,277 |
| Dockerfile | 39,664 |
| Swig | 37,539 |
| LD Script | 36,307 |
| FORTRAN Legacy | 35,889 |
| Scons | 35,373 |
| Scheme | 34,982 |
| Alex | 34,221 |
| TCL | 33,766 |
| Android Interface Definition Language | 33,000 |
| Ruby HTML | 32,645 |
| Device Tree | 31,918 |
| Expect | 30,249 |
| Cabal | 30,109 |
| Unreal Script | 29,113 |
| Pascal | 28,439 |
| GLSL | 28,417 |
| Intel HEX | 27,504 |
| Alloy | 27,142 |
| Freemarker Template | 26,456 |
| IDL | 26,079 |
| Visual Basic for Applications | 26,061 |
| Macromedia eXtensible Markup Language | 24,949 |
| F# | 24,373 |
| Cython | 23,858 |
| Jupyter | 23,577 |
| Forth | 22,108 |
| Visual Basic | 21,909 |
| Lisp | 21,242 |
| OCaml | 20,216 |
| Rust | 19,286 |
| Fish | 18,079 |
| Monkey C | 17,753 |
| Ada | 17,253 |
| SAS | 17,031 |
| Dart | 16,447 |
| TypeScript Typings | 16,263 |
| SystemVerilog | 15,541 |
| Thrift | 15,390 |
| C Shell | 14,904 |
| Fragment Shader File | 14,572 |
| Vertex Shader File | 14,312 |
| QML | 13,709 |
| ColdFusion | 13,441 |
| Elixir | 12,716 |
| Haxe | 12,404 |
| Jinja | 12,274 |
| JSX | 12,194 |
| Specman e | 12,071 |
| FORTRAN Modern | 11,460 |
| PKGBUILD | 11,398 |
| ignore | 11,287 |
| Mako | 10,846 |
| TOML | 10,444 |
| SKILL | 10,048 |
| AsciiDoc | 9,868 |
| Swift | 9,679 |
| BuildStream | 9,198 |
| ColdFusion CFScript | 8,614 |
| Stata | 8,296 |
| Creole | 8,030 |
| Basic | 7,751 |
| V | 7,560 |
| VHDL | 7,368 |
| Julia | 7,070 |
| ClojureScript | 7,018 |
| Closure Template | 6,269 |
| AutoHotKey | 5,938 |
| Wolfram | 5,764 |
| Docker ignore | 5,555 |
| Korn Shell | 5,541 |
| Arvo | 5,364 |
| Coq | 5,068 |
| SRecode Template | 5,019 |
| Game Maker Language | 4,557 |
| Nix | 4,216 |
| Vala | 4,110 |
| COBOL | 3,946 |
| Varnish Configuration | 3,882 |
| Kotlin | 3,683 |
| Bitbake | 3,645 |
| GDScript | 3,189 |
| Standard ML (SML) | 3,143 |
| Jenkins Buildfile | 2,822 |
| Xtend | 2,791 |
| ABAP | 2,381 |
| Modula3 | 2,376 |
| Nim | 2,273 |
| Verilog | 2,013 |
| Elm | 1,849 |
| Brainfuck | 1,794 |
| Ur/Web | 1,741 |
| Opalang | 1,367 |
| GN | 1,342 |
| TaskPaper | 1,330 |
| Ceylon | 1,265 |
| Crystal | 1,259 |
| Agda | 1,182 |
| Vue | 1,139 |
| LOLCODE | 1,101 |
| Hamlet | 1,071 |
| Robot Framework | 1,062 |
| MUMPS | 940 |
| Emacs Dev Env | 937 |
| Cargo Lock | 905 |
| Flow9 | 839 |
| Idris | 804 |
| Julius | 765 |
| Oz | 764 |
| Q# | 695 |
| Lucius | 627 |
| Meson | 617 |
| F* | 614 |
| ATS | 492 |
| PSL Assertion | 483 |
| Bitbucket Pipeline | 418 |
| PureScript | 370 |
| Report Definition Language | 313 |
| Isabelle | 296 |
| JAI | 286 |
| MQL4 | 271 |
| Ur/Web Project | 261 |
| Alchemist | 250 |
| Cassius | 213 |
| Softbridge Basic | 207 |
| MQL Header | 167 |
| JSONL | 146 |
| Lean | 104 |
| Spice Netlist | 100 |
| Madlang | 97 |
| Luna | 91 |
| Pony | 86 |
| MQL5 | 46 |
| Wren | 33 |
| Just | 30 |
| QCL | 27 |
| Zig | 21 |
| SPDX | 20 |
| Futhark | 16 |
| Dhall | 15 |
| FIDL | 14 |
| Bosque | 14 |
| Janet | 13 |
| Game Maker Project | 6 |
| Polly | 6 |
| Verilog Args File | 2

### How many files in a repository per language?

An extension of the above, but averaged over however many files are in each language per repository. So for projects that contain java, how many java files exist in that project, and on average for all projects how many files is that?

You can use this to see if a project is larger or smaller than usual for your language of choice.

[skip table to next section](#how-many-lines-of-code-are-in-a-typical-file-per-language)

<div class="table-2"></div>

| language | average file count |
| -------- | ------------------ |
| ABAP | 1.0008927583699165 |
| ASP | 1.6565139917314107 |
| ASP.NET | 346.88867258489296 |
| ATS | 7.888545610390882 |
| AWK | 5.098807478952136 |
| ActionScript | 15.682562363539644 |
| Ada | 7.265376817272021 |
| Agda | 1.2669381110755398 |
| Alchemist | 7.437307493090622 |
| Alex | 20.152479318023637 |
| Alloy | 1.0000000894069672 |
| Android Interface Definition Language | 3.1133707938643074 |
| Arvo | 9.872687772928423 |
| AsciiDoc | 14.645389421879814 |
| Assembly | 1049.6270518312476 |
| AutoHotKey | 1.5361384288472488 |
| Autoconf | 33.99728695464163 |
| BASH | 3.7384110335355545 |
| Basic | 5.103623499110781 |
| Batch | 3.943513588378872 |
| Bazel | 1.0013122734382187 |
| Bitbake | 1.0878349272366024 |
| Bitbucket Pipeline | 1 |
| Boo | 5.321822367969364 |
| Bosque | 1.28173828125 |
| Brainfuck | 1.3141119785974242 |
| BuildStream | 1.4704635441667189 |
| C | 15610.17972307699 |
| C Header | 14103.33936083782 |
| C Shell | 3.1231084093649315 |
| C# | 45.804460355773394 |
| C++ | 30.416980313492328 |
| C++ Header | 8.313450764990089 |
| CMake | 37.2566873554469 |
| COBOL | 3.129408853490878 |
| CSS | 5.332398714337156 |
| CSV | 8.370432089241898 |
| Cabal | 1.0078125149013983 |
| Cargo Lock | 1.0026407549221519 |
| Cassius | 4.657169356495984 |
| Ceylon | 7.397692655679642 |
| Clojure | 8.702303821528872 |
| ClojureScript | 5.384518778099244 |
| Closure Template | 1.0210028022356945 |
| CoffeeScript | 45.40906609668401 |
| ColdFusion | 13.611857060674573 |
| ColdFusion CFScript | 40.42554202020521 |
| Coq | 10.903652047164622 |
| Creole | 1.000122070313864 |
| Crystal | 3.8729367926098117 |
| Cython | 1.9811811237515262 |
| D | 529.2562750397005 |
| Dart | 1.5259554297822313 |
| Device Tree | 586.4119588123021 |
| Dhall | 5.072265625 |
| Docker ignore | 1.0058596283197403 |
| Dockerfile | 1.7570825852789156 |
| Document Type Definition | 2.2977520758534693 |
| Elixir | 8.916658446524252 |
| Elm | 1.6702759813968946 |
| Emacs Dev Env | 15.720268315288969 |
| Emacs Lisp | 11.378847912292201 |
| Erlang | 3.4764894379621607 |
| Expect | 2.8863991651091614 |
| Extensible Stylesheet Language Transformations | 1.2042068607534995 |
| F# | 1.2856606249320954 |
| F* | 32.784058919015 |
| FIDL | 1.8441162109375 |
| FORTRAN Legacy | 11.37801716560221 |
| FORTRAN Modern | 27.408192558594685 |
| Fish | 1.1282354207617833 |
| Flow9 | 5.218046229973186 |
| Forth | 10.64736177807574 |
| Fragment Shader File | 3.648087980622546 |
| Freemarker Template | 8.397226930409037 |
| Futhark | 4.671875 |
| GDScript | 3.6984173692608313 |
| GLSL | 1.6749061330076334 |
| GN | 1.0193083210608163 |
| Game Maker Language | 3.6370866431049604 |
| Game Maker Project | 1.625 |
| Gherkin Specification | 60.430588516231666 |
| Go | 115.23482489228113 |
| Go Template | 28.011342078505013 |
| Gradle | 5.628880473160033 |
| Groovy | 6.697367294187844 |
| HEX | 22.477003537989486 |
| HTML | 4.822243456786672 |
| Hamlet | 50.297887645777536 |
| Handlebars | 36.60120978679127 |
| Happy | 5.820573911044464 |
| Haskell | 8.730027121836951 |
| Haxe | 20.00590981880653 |
| IDL | 79.38510300176867 |
| Idris | 1.524684997890027 |
| Intel HEX | 113.25178379632708 |
| Isabelle | 1.8903018088753136 |
| JAI | 1.4865150753259275 |
| JSON | 6.507823973898348 |
| JSONL | 1.003931049286678 |
| JSX | 4.6359645801363465 |
| Jade | 5.353279289700571 |
| Janet | 1.0390625 |
| Java | 118.86142228014006 |
| JavaScript | 140.56079100796154 |
| JavaServer Pages | 2.390251418283771 |
| Jenkins Buildfile | 1.0000000000582077 |
| Jinja | 4.574843152310338 |
| Julia | 6.672268339671913 |
| Julius | 2.2510109380818903 |
| Jupyter | 13.480476117239338 |
| Just | 1.736882857978344 |
| Korn Shell | 1.5100887455636172 |
| Kotlin | 3.9004723322169363 |
| LD Script | 16.59996086864524 |
| LESS | 39.6484785300563 |
| LEX | 5.892075421476933 |
| LOLCODE | 1.0381496530137617 |
| LaTeX | 5.336103768010524 |
| Lean | 1.6653789470747329 |
| License | 5.593879701111845 |
| Lisp | 33.15947937896521 |
| Lua | 24.796117625764612 |
| Lucius | 6.5742989471450155 |
| Luna | 4.437807061133055 |
| MQL Header | 13.515527575704464 |
| MQL4 | 6.400151428436254 |
| MQL5 | 46.489316522221515 |
| MSBuild | 4.8321384193507875 |
| MUMPS | 8.187699062741014 |
| Macromedia eXtensible Markup Language | 2.1945287114300807 |
| Madlang | 3.7857666909751373 |
| Makefile | 1518.1769808494607 |
| Mako | 3.410234685769436 |
| Markdown | 45.687500000234245 |
| Meson | 32.45071679724949 |
| Modula3 | 1.1610784588847318 |
| Module-Definition | 4.9327688042002595 |
| Monkey C | 3.035163164383345 |
| Mustache | 19.052714578803542 |
| Nim | 1.202213335585401 |
| Nix | 2.7291879559930488 |
| OCaml | 3.7135029841909697 |
| Objective C | 4.9795510788040005 |
| Objective C++ | 2.2285232767506264 |
| Opalang | 1.9975597794742732 |
| Org | 5.258117805392903 |
| Oz | 22.250069644336204 |
| PHP | 199.17870638869982 |
| PKGBUILD | 7.50632295051949 |
| PSL Assertion | 3.0736406530442473 |
| Pascal | 90.55238627885495 |
| Patch | 25.331829692384225 |
| Perl | 27.46770444081142 |
| Plain Text | 1119.2375825397799 |
| Polly | 1 |
| Pony | 3.173291031071342 |
| Powershell | 6.629884642978543 |
| Processing | 9.139907354078636 |
| Prolog | 1.816763080890156 |
| Properties File | 2.1801967863634255 |
| Protocol Buffers | 2.0456253005879304 |
| Puppet | 43.424491631161054 |
| PureScript | 4.063801504037935 |
| Python | 22.473917606983292 |
| Q# | 5.712939431518483 |
| QCL | 7.590678825974464 |
| QML | 1.255201818986247 |
| R | 2.3781868952970115 |
| Rakefile | 14.856192677576413 |
| Razor | 62.79058974450959 |
| ReStructuredText | 11.63852408056825 |
| Report Definition Language | 23.065085061465403 |
| Robot Framework | 2.6260137148703535 |
| Ruby | 554.0134362337432 |
| Ruby HTML | 24.091116656979562 |
| Rust | 2.3002003813895207 |
| SAS | 1.0032075758254648 |
| SKILL | 1.9229039972029645 |
| SPDX | 2.457843780517578 |
| SQL | 2.293643752864969 |
| SRecode Template | 20.688193360975845 |
| SVG | 4.616972531365432 |
| Sass | 42.92418345584642 |
| Scala | 1.5957851387966393 |
| Scheme | 10.480490204644848 |
| Scons | 2.1977062552968114 |
| Shell | 41.88552208947577 |
| Smarty Template | 6.90615223295527 |
| Softbridge Basic | 22.218602385698304 |
| Specman e | 2.719783829645327 |
| Spice Netlist | 2.454830619852739 |
| Standard ML (SML) | 3.7598713626650295 |
| Stata | 2.832579915520368 |
| Stylus | 7.903926412469745 |
| Swift | 54.175594149331914 |
| Swig | 2.3953681161240747 |
| SystemVerilog | 7.120705494624247 |
| Systemd | 80.83254275520476 |
| TCL | 46.9378307136513 |
| TOML | 1.0316491217260413 |
| TaskPaper | 1.0005036006351133 |
| TeX | 8.690789447558961 |
| Thrift | 1.620168483240211 |
| Twig Template | 18.33051814392764 |
| TypeScript | 1.2610517452930048 |
| TypeScript Typings | 2.3638072576034137 |
| Unreal Script | 2.9971615019965148 |
| Ur/Web | 3.420488425604595 |
| Ur/Web Project | 1.8752869585517504 |
| V | 1.8780624768245784 |
| VHDL | 5.764059992075602 |
| Vala | 42.22072166146626 |
| Varnish Configuration | 1.9899734258599446 |
| Verilog | 1.443359777332832 |
| Verilog Args File | 25.5 |
| Vertex Shader File | 2.4700152399875077 |
| Vim Script | 3.2196359799822662 |
| Visual Basic | 119.8397831247842 |
| Visual Basic for Applications | 2.5806381264096503 |
| Vue | 249.0557418123258 |
| Wolfram | 1.462178856761796 |
| Wren | 227.4526259500999 |
| XAML | 2.6149608174399264 |
| XCode Config | 6.979387911493798 |
| XML | 146.10128153519918 |
| XML Schema | 6.832042266604565 |
| Xtend | 2.87054940757827 |
| YAML | 6.170148717655746 |
| Zig | 1.071681022644043 |
| Zsh | 2.6295064863912088 |
| gitignore | 6.878908416722053 |
| ignore | 1.0210649380633772 |
| m4 | 57.5969985568356 |
| nuspec | 3.245791111381787 |
| sed | 1.3985770380241234 |


### How many lines of code are in a typical file per language?

I suppose you could also look at this as what languages on average have the largest files? Using the average/mean for this pushes the results out to stupidly high numbers. This is because projects such as sqlite.c which is included in many projects is joined from many files into one, but nobody ever works on that single large file (I hope!).

So I calculated this using the median value. Even so there are still some definitions with stupidly high numbers such as Bosque and JavaScript. 

So I figured why not have both? I did one small change based on the suggestion of [Darrell](https://www.packtpub.com/au/big-data-and-business-intelligence/hands-deep-learning-go) (Kablamo's resident and most excellent data scientist) and modified the average value to ignore files over 5000 lines to remove the outliers.

[skip table to next section](#average-complexity-for-file-in-each-language)

<div class="table-3"></div>

| language | mean < 5000 | median |
| -------- | ----------- | ------ |
| ABAP | 139 | 36 |
| ASP | 513 | 170 |
| ASP.NET | 315 | 148 |
| ATS | 945 | 1,411 |
| AWK | 431 | 774 |
| ActionScript | 950 | 2,676 |
| Ada | 1,179 | 13 |
| Agda | 466 | 89 |
| Alchemist | 1,040 | 1,463 |
| Alex | 479 | 204 |
| Alloy | 72 | 66 |
| Android Interface Definition Language | 119 | 190 |
| Arvo | 257 | 1,508 |
| AsciiDoc | 519 | 1,724 |
| Assembly | 993 | 225 |
| AutoHotKey | 360 | 23 |
| Autoconf | 495 | 144 |
| BASH | 425 | 26 |
| Basic | 476 | 847 |
| Batch | 178 | 208 |
| Bazel | 226 | 20 |
| Bitbake | 436 | 10 |
| Bitbucket Pipeline | 19 | 13 |
| Boo | 898 | 924 |
| Bosque | 58 | 199,238 |
| Brainfuck | 141 | 177 |
| BuildStream | 1,955 | 2,384 |
| C | 1,052 | 5,774 |
| C Header | 869 | 126,460 |
| C Shell | 128 | 77 |
| C# | 1,215 | 1,138 |
| C++ | 1,166 | 232 |
| C++ Header | 838 | 125 |
| CMake | 750 | 15 |
| COBOL | 422 | 24 |
| CSS | 729 | 103 |
| CSV | 411 | 12 |
| Cabal | 116 | 13 |
| Cargo Lock | 814 | 686 |
| Cassius | 124 | 634 |
| Ceylon | 207 | 15 |
| Clojure | 521 | 19 |
| ClojureScript | 504 | 195 |
| Closure Template | 343 | 75 |
| CoffeeScript | 342 | 168 |
| ColdFusion | 686 | 5 |
| ColdFusion CFScript | 1,231 | 1,829 |
| Coq | 560 | 29,250 |
| Creole | 85 | 20 |
| Crystal | 973 | 119 |
| Cython | 853 | 1,738 |
| D | 397 | 10 |
| Dart | 583 | 500 |
| Device Tree | 739 | 44,002 |
| Dhall | 124 | 99 |
| Docker ignore | 10 | 2 |
| Dockerfile | 76 | 17 |
| Document Type Definition | 522 | 1,202 |
| Elixir | 402 | 192 |
| Elm | 438 | 121 |
| Emacs Dev Env | 646 | 755 |
| Emacs Lisp | 653 | 15 |
| Erlang | 930 | 203 |
| Expect | 419 | 195 |
| Extensible Stylesheet Language Transformations | 442 | 600 |
| F# | 384 | 64 |
| F* | 335 | 65 |
| FIDL | 655 | 1,502 |
| FORTRAN Legacy | 277 | 1,925 |
| FORTRAN Modern | 636 | 244 |
| Fish | 168 | 74 |
| Flow9 | 368 | 32 |
| Forth | 256 | 62 |
| Fragment Shader File | 309 | 11 |
| Freemarker Template | 522 | 20 |
| Futhark | 175 | 257 |
| GDScript | 401 | 1 |
| GLSL | 380 | 29 |
| GN | 950 | 8,866 |
| Game Maker Language | 710 | 516 |
| Game Maker Project | 1,290 | 374 |
| Gherkin Specification | 516 | 2,386 |
| Go | 780 | 558 |
| Go Template | 411 | 25,342 |
| Gradle | 228 | 22 |
| Groovy | 734 | 13 |
| HEX | 1,002 | 17,208 |
| HTML | 556 | 1,814 |
| Hamlet | 220 | 70 |
| Handlebars | 506 | 3,162 |
| Happy | 1,617 | 0 |
| Haskell | 656 | 17 |
| Haxe | 865 | 9,607 |
| IDL | 386 | 210 |
| Idris | 285 | 42 |
| Intel HEX | 1,256 | 106,650 |
| Isabelle | 792 | 1,736 |
| JAI | 268 | 41 |
| JSON | 289 | 39 |
| JSONL | 43 | 2 |
| JSX | 393 | 24 |
| Jade | 299 | 192 |
| Janet | 508 | 32 |
| Java | 1,165 | 697 |
| JavaScript | 894 | 73,979 |
| JavaServer Pages | 644 | 924 |
| Jenkins Buildfile | 79 | 6 |
| Jinja | 465 | 3,914 |
| Julia | 539 | 1,031 |
| Julius | 113 | 12 |
| Jupyter | 1,361 | 688 |
| Just | 62 | 72 |
| Korn Shell | 427 | 776 |
| Kotlin | 554 | 169 |
| LD Script | 521 | 439 |
| LESS | 1,086 | 17 |
| LEX | 1,014 | 214 |
| LOLCODE | 129 | 4 |
| LaTeX | 895 | 7,482 |
| Lean | 181 | 9 |
| License | 266 | 20 |
| Lisp | 746 | 1,201 |
| Lua | 820 | 559 |
| Lucius | 284 | 445 |
| Luna | 85 | 48 |
| MQL Header | 793 | 10,337 |
| MQL4 | 799 | 3,168 |
| MQL5 | 384 | 631 |
| MSBuild | 558 | 160 |
| MUMPS | 924 | 98,191 |
| Macromedia eXtensible Markup Language | 500 | 20 |
| Madlang | 368 | 340 |
| Makefile | 309 | 20 |
| Mako | 269 | 243 |
| Markdown | 206 | 10 |
| Meson | 546 | 205 |
| Modula3 | 162 | 17 |
| Module-Definition | 489 | 7 |
| Monkey C | 140 | 28 |
| Mustache | 298 | 8,083 |
| Nim | 352 | 3 |
| Nix | 240 | 78 |
| OCaml | 718 | 68 |
| Objective C | 1,111 | 17,103 |
| Objective C++ | 903 | 244 |
| Opalang | 151 | 29 |
| Org | 523 | 24 |
| Oz | 360 | 7,132 |
| PHP | 964 | 14,660 |
| PKGBUILD | 131 | 19 |
| PSL Assertion | 149 | 108 |
| Pascal | 1,044 | 497 |
| Patch | 676 | 12 |
| Perl | 762 | 11 |
| Plain Text | 352 | 841 |
| Polly | 12 | 26 |
| Pony | 338 | 42,488 |
| Powershell | 652 | 199 |
| Processing | 800 | 903 |
| Prolog | 282 | 6 |
| Properties File | 184 | 18 |
| Protocol Buffers | 576 | 8,080 |
| Puppet | 499 | 660 |
| PureScript | 598 | 363 |
| Python | 879 | 258 |
| Q# | 475 | 5,417 |
| QCL | 548 | 3 |
| QML | 815 | 6,067 |
| R | 566 | 20 |
| Rakefile | 122 | 7 |
| Razor | 713 | 1,842 |
| ReStructuredText | 735 | 5,049 |
| Report Definition Language | 1,389 | 34,337 |
| Robot Framework | 292 | 115 |
| Ruby | 739 | 4,942 |
| Ruby HTML | 326 | 192 |
| Rust | 1,007 | 4 |
| SAS | 233 | 65 |
| SKILL | 526 | 123 |
| SPDX | 1,242 | 379 |
| SQL | 466 | 143 |
| SRecode Template | 796 | 534 |
| SVG | 796 | 1,538 |
| Sass | 682 | 14,653 |
| Scala | 612 | 661 |
| Scheme | 566 | 6 |
| Scons | 545 | 6,042 |
| Shell | 304 | 4 |
| Smarty Template | 392 | 15 |
| Softbridge Basic | 2,067 | 3 |
| Specman e | 127 | 0 |
| Spice Netlist | 906 | 1,465 |
| Standard ML (SML) | 478 | 75 |
| Stata | 200 | 12 |
| Stylus | 505 | 214 |
| Swift | 683 | 663 |
| Swig | 1,031 | 4,540 |
| SystemVerilog | 563 | 830 |
| Systemd | 127 | 26 |
| TCL | 774 | 42,396 |
| TOML | 100 | 17 |
| TaskPaper | 37 | 7 |
| TeX | 804 | 905 |
| Thrift | 545 | 329 |
| Twig Template | 713 | 9,907 |
| TypeScript | 461 | 10 |
| TypeScript Typings | 1,465 | 236,866 |
| Unreal Script | 795 | 927 |
| Ur/Web | 429 | 848 |
| Ur/Web Project | 33 | 26 |
| V | 704 | 5,711 |
| VHDL | 952 | 1,452 |
| Vala | 603 | 2 |
| Varnish Configuration | 203 | 77 |
| Verilog | 198 | 2 |
| Verilog Args File | 456 | 481 |
| Vertex Shader File | 168 | 74 |
| Vim Script | 555 | 25 |
| Visual Basic | 738 | 1,050 |
| Visual Basic for Applications | 979 | 936 |
| Vue | 732 | 242 |
| Wolfram | 940 | 973 |
| Wren | 358 | 279,258 |
| XAML | 703 | 24 |
| XCode Config | 200 | 11 |
| XML | 605 | 1,033 |
| XML Schema | 1,008 | 248 |
| Xtend | 710 | 120 |
| YAML | 165 | 47,327 |
| Zig | 188 | 724 |
| Zsh | 300 | 9 |
| gitignore | 33 | 3 |
| ignore | 6 | 2 |
| m4 | 959 | 807 |
| nuspec | 187 | 193 |
| sed | 82 | 33 |

### Average complexity for file in each language?

What's the average complexity per file for each language?

The complexity estimate isn't really directly comparable between languages. Pulling from the README of `scc` itself

> The complexity estimate is really just a number that is only comparable to files in the same language. It should not be used to compare languages directly without weighting them. The reason for this is that its calculated by looking for branch and loop statements in the code and incrementing a counter for that file.

So comparing languages to each other is not the idea here, although it may be comparable between similar languages such as Java and C for example. Your mileage may vary.

This is more useful when you think about it applying to single files in the same language. So you could answer the question "On average is the file I am working with more or less complex than average?".

I should mention that I am always looking to improve this calculation and looking for submissions to [scc](https://github.com/boyter/scc/) to assist with this goal. Usually it is a case of just adding some keywords to the languages.json file so any programmer of any skill level should be able to assist with this. 

[skip table to next section](#what-are-the-most-common-filenames)

<div class="table-9"></div>

| language | complexity |
| -------- | ----------- |
| ABAP | 11.180740488380376 |
| ASP | 11.536947250366211 |
| ASP.NET | 2.149275320643484 |
| ATS | 0.7621728432717677 |
| AWK | 0 |
| ActionScript | 22.088579905848178 |
| Ada | 13.69141626294931 |
| Agda | 0.19536590785719454 |
| Alchemist | 0.3423442907696928 |
| Alex | 0 |
| Alloy | 6.9999997997656465 |
| Android Interface Definition Language | 0 |
| Arvo | 0 |
| AsciiDoc | 0 |
| Assembly | 1.5605608227976997 |
| AutoHotKey | 423.87785756399626 |
| Autoconf | 1.5524294972419739 |
| BASH | 7.500000094871363 |
| Basic | 1.0001350622574257 |
| Batch | 1.4136352496767306 |
| Bazel | 6.523681727119303 |
| Bitbake | 0.00391388021490391 |
| Bitbucket Pipeline | 0 |
| Boo | 65.67764583729533 |
| Bosque | 236.79837036132812 |
| Brainfuck | 27.5516445041791 |
| BuildStream | 0 |
| C | 220.17236548200242 |
| C Header | 0.027589923237434522 |
| C Shell | 1.4911166269191476 |
| C# | 1.0994400597744005 |
| C++ | 215.23628287682845 |
| C++ Header | 2.2893104921677154 |
| CMake | 0.887660006199008 |
| COBOL | 0.018726348891789816 |
| CSS | 6.317460331175305E-176 |
| CSV | 0 |
| Cabal | 3.6547924155738194 |
| Cargo Lock | 0 |
| Cassius | 0 |
| Ceylon | 21.664400369259404 |
| Clojure | 0.00009155273437716484 |
| ClojureScript | 0.5347588658332859 |
| Closure Template | 0.503426091716392 |
| CoffeeScript | 0.02021490140137264 |
| ColdFusion | 6.851776515250336 |
| ColdFusion CFScript | 22.287403080299764 |
| Coq | 3.3282556015266307 |
| Creole | 0 |
| Crystal | 1.6065794006138856 |
| Cython | 42.87412906489837 |
| D | 0 |
| Dart | 2.1264450684815657 |
| Device Tree | 0 |
| Dhall | 0 |
| Docker ignore | 0 |
| Dockerfile | 6.158891172385556 |
| Document Type Definition | 0 |
| Elixir | 0.5000612735793482 |
| Elm | 5.237952479502043 |
| Emacs Dev Env | 1.2701271416728307E-61 |
| Emacs Lisp | 0.19531250990197657 |
| Erlang | 0.08028322620528387 |
| Expect | 0.329944610851471 |
| Extensible Stylesheet Language Transformations | 0 |
| F# | 0.32300702900710193 |
| F* | 9.403954876643223E-38 |
| FIDL | 0.12695312593132269 |
| FORTRAN Legacy | 0.8337643985574195 |
| FORTRAN Modern | 7.5833590276411185 |
| Fish | 1.3386242155247368 |
| Flow9 | 34.5 |
| Forth | 2.4664166555765066 |
| Fragment Shader File | 0.0003388836600090293 |
| Freemarker Template | 10.511094652522283 |
| Futhark | 0.8057891242233386 |
| GDScript | 10.750000000022537 |
| GLSL | 0.6383056697891334 |
| GN | 22.400601854287807 |
| Game Maker Language | 4.709514207365569 |
| Game Maker Project | 0 |
| Gherkin Specification | 0.4085178437480328 |
| Go | 50.06279203974034 |
| Go Template | 2.3866690339840662E-153 |
| Gradle | 0 |
| Groovy | 3.2506868488244898 |
| HEX | 0 |
| HTML | 0 |
| Hamlet | 0.25053861103978114 |
| Handlebars | 1.6943764911351036E-21 |
| Happy | 0 |
| Haskell | 28.470107150053625 |
| Haxe | 66.52873523714804 |
| IDL | 7.450580598712868E-9 |
| Idris | 17.77642903881352 |
| Intel HEX | 0 |
| Isabelle | 0.0014658546850726184 |
| JAI | 7.749968137734008 |
| JSON | 0 |
| JSONL | 0 |
| JSX | 0.3910405338329044 |
| Jade | 0.6881713929215119 |
| Janet | 0 |
| Java | 297.22908150612085 |
| JavaScript | 1.861130583340945 |
| JavaServer Pages | 7.24235416213196 |
| Jenkins Buildfile | 0 |
| Jinja | 0.6118526458846931 |
| Julia | 5.779676990326951 |
| Julius | 3.7432448068125277 |
| Jupyter | 0 |
| Just | 1.625490248219907 |
| Korn Shell | 11.085027896435056 |
| Kotlin | 5.467347841779503 |
| LD Script | 6.538079182471746E-26 |
| LESS | 0 |
| LEX | 0 |
| LOLCODE | 5.980839657708373 |
| LaTeX | 0 |
| Lean | 0.0019872561135834133 |
| License | 0 |
| Lisp | 4.033602018074421 |
| Lua | 44.70686769972825 |
| Lucius | 0 |
| Luna | 0 |
| MQL Header | 82.8036524637758 |
| MQL4 | 2.9989408299408566 |
| MQL5 | 32.84198718928553 |
| MSBuild | 2.9802322387695312E-8 |
| MUMPS | 5.767955578948634E-17 |
| Macromedia eXtensible Markup Language | 0 |
| Madlang | 8.25 |
| Makefile | 3.9272747722381812E-90 |
| Mako | 0.007624773579836673 |
| Markdown | 0 |
| Meson | 0.3975182396400463 |
| Modula3 | 0.7517121883916386 |
| Module-Definition | 0.25000000023283153 |
| Monkey C | 9.838715311259486 |
| Mustache | 0.00004191328599945435 |
| Nim | 0.04812580073302998 |
| Nix | 25.500204694250044 |
| OCaml | 16.92218069843716 |
| Objective C | 65.08967337175548 |
| Objective C++ | 10.886891531550603 |
| Opalang | 1.3724696160763994E-8 |
| Org | 28.947825231747235 |
| Oz | 6.260657086070324 |
| PHP | 2.8314653639690874 |
| PKGBUILD | 0 |
| PSL Assertion | 0.5009768009185791 |
| Pascal | 4 |
| Patch | 0 |
| Perl | 48.16959255514553 |
| Plain Text | 0 |
| Polly | 0 |
| Pony | 4.91082763671875 |
| Powershell | 0.43151378893449877 |
| Processing | 9.691001653621564 |
| Prolog | 0.5029296875147224 |
| Properties File | 0 |
| Protocol Buffers | 0.07128906529847256 |
| Puppet | 0.16606500436341776 |
| PureScript | 1.3008141816356456 |
| Python | 11.510142201304832 |
| Q# | 5.222080192729404 |
| QCL | 13.195626304795667 |
| QML | 0.3208023407643109 |
| R | 0.40128818821921775 |
| Rakefile | 2.75786388297917 |
| Razor | 0.5298294073055322 |
| ReStructuredText | 0 |
| Report Definition Language | 0 |
| Robot Framework | 0 |
| Ruby | 7.8611656283491795 |
| Ruby HTML | 1.3175727506823756 |
| Rust | 8.62646485221385 |
| SAS | 0.5223999023437882 |
| SKILL | 0.4404907226562501 |
| SPDX | 0 |
| SQL | 0.00001537799835205078 |
| SRecode Template | 0.18119949102401853 |
| SVG | 1.7686873200833423E-74 |
| Sass | 7.002974651049148E-113 |
| Scala | 17.522343645163424 |
| Scheme | 0.00003147125255509322 |
| Scons | 25.56868253610655 |
| Shell | 6.409446969197895 |
| Smarty Template | 53.06143077491294 |
| Softbridge Basic | 7.5 |
| Specman e | 0.0639350358484781 |
| Spice Netlist | 1.3684555315672042E-48 |
| Standard ML (SML) | 24.686901116754818 |
| Stata | 1.5115316917094068 |
| Stylus | 0.3750006556512421 |
| Swift | 0.5793484510104517 |
| Swig | 0 |
| SystemVerilog | 0.250593163372906 |
| Systemd | 0 |
| TCL | 96.5072605676113 |
| TOML | 0.0048828125000002776 |
| TaskPaper | 0 |
| TeX | 54.0588040258797 |
| Thrift | 0 |
| Twig Template | 2.668124511961211 |
| TypeScript | 9.191392608918255 |
| TypeScript Typings | 6.1642456222327375 |
| Unreal Script | 2.7333421227943004 |
| Ur/Web | 16.51621568240534 |
| Ur/Web Project | 0 |
| V | 22.50230618938804 |
| VHDL | 18.05495198571289 |
| Vala | 147.2761703068509 |
| Varnish Configuration | 0 |
| Verilog | 5.582400367711671 |
| Verilog Args File | 0 |
| Vertex Shader File | 0.0010757446297590262 |
| Vim Script | 2.4234658314493798 |
| Visual Basic | 0.0004882812500167852 |
| Visual Basic for Applications | 4.761343429454877 |
| Vue | 0.7529517744621779 |
| Wolfram | 0.0059204399585724215 |
| Wren | 0.08593750013097715 |
| XAML | 6.984919309616089E-10 |
| XCode Config | 0 |
| XML | 0 |
| XML Schema | 0 |
| Xtend | 2.8245844719990547 |
| YAML | 0 |
| Zig | 1.0158334437942358 |
| Zsh | 1.81697392626756 |
| gitignore | 0 |
| ignore | 0 |
| m4 | 0 |
| nuspec | 0 |
| sed | 22.91158285739948 |

### Average comments for file in each language?

What's the average amount of comments used per file in each language? 

You could probably rephrase this to asking what developers write the most comments assuming you squint enough.

[skip table to next section](#what-are-the-most-common-filenames)

<div class="table-11"></div>

| language | complexity |
| -------- | ----------- |
| ABAP | 56.3020026683825 |
| ASP | 24.67145299911499 |
| ASP.NET | 9.140447860406259E-11 |
| ATS | 41.89465025163305 |
| AWK | 11.290069486393975 |
| ActionScript | 31.3568633027012 |
| Ada | 61.269572412982384 |
| Agda | 2.4337660860304755 |
| Alchemist | 2.232399710231226E-103 |
| Alex | 0 |
| Alloy | 0.000002207234501959681 |
| Android Interface Definition Language | 26.984662160277367 |
| Arvo | 0 |
| AsciiDoc | 0 |
| Assembly | 2.263919769706678E-72 |
| AutoHotKey | 15.833985920534857 |
| Autoconf | 0.47779749499136687 |
| BASH | 34.15625059662068 |
| Basic | 1.4219117348874069 |
| Batch | 1.0430908205926455 |
| Bazel | 71.21859817579139 |
| Bitbake | 0.002480246487177871 |
| Bitbucket Pipeline | 0.567799577547725 |
| Boo | 5.03128187009327 |
| Bosque | 0.125244140625 |
| Brainfuck | 0 |
| BuildStream | 12.84734197699206 |
| C | 256.2839210573451 |
| C Header | 184.88885430308878 |
| C Shell | 5.8409870392823375 |
| C# | 30.96563720101839 |
| C++ | 44.61584829131642 |
| C++ Header | 27.578790410119197 |
| CMake | 1.7564333047949374 |
| COBOL | 0.7503204345703562 |
| CSS | 4.998773531463529 |
| CSV | 0 |
| Cabal | 4.899812531420634 |
| Cargo Lock | 0.0703125 |
| Cassius | 0.07177734654413487 |
| Ceylon | 3.6406326349824667 |
| Clojure | 0.0987220821845421 |
| ClojureScript | 0.6025725119252456 |
| Closure Template | 17.078124673988057 |
| CoffeeScript | 1.6345682790069884 |
| ColdFusion | 33.745563628665096 |
| ColdFusion CFScript | 13.566947396771592 |
| Coq | 20.3222774725393 |
| Creole | 0 |
| Crystal | 6.0308081267588145 |
| Cython | 21.0593019957583 |
| D | 0 |
| Dart | 4.634361584097128 |
| Device Tree | 33.64898256434121 |
| Dhall | 1.0053101042303751 |
| Docker ignore | 8.003553375601768E-11 |
| Dockerfile | 4.526245545632278 |
| Document Type Definition | 0 |
| Elixir | 8.0581139370409 |
| Elm | 24.73191350743249 |
| Emacs Dev Env | 2.74822998046875 |
| Emacs Lisp | 12.168370702306452 |
| Erlang | 16.670030919109056 |
| Expect | 3.606161126133445 |
| Extensible Stylesheet Language Transformations | 0 |
| F# | 0.5029605040200058 |
| F* | 5.33528354690743E-27 |
| FIDL | 343.0418392068642 |
| FORTRAN Legacy | 8.121405267242158 |
| FORTRAN Modern | 171.32042583820953 |
| Fish | 7.979248739519377 |
| Flow9 | 0.5049991616979244 |
| Forth | 0.7578125 |
| Fragment Shader File | 0.2373057885016209 |
| Freemarker Template | 62.250244379050855 |
| Futhark | 0.014113984877253714 |
| GDScript | 31.14457228694065 |
| GLSL | 0.2182627061047912 |
| GN | 17.443267241931284 |
| Game Maker Language | 3.9815753922640824 |
| Game Maker Project | 0 |
| Gherkin Specification | 0.0032959059321794604 |
| Go | 6.464829990599041 |
| Go Template | 4.460169822267483E-251 |
| Gradle | 0.5374194774415457 |
| Groovy | 32.32068506016523 |
| HEX | 0 |
| HTML | 0.16671794164614084 |
| Hamlet | 4.203293477836184E-24 |
| Handlebars | 0.9389737429747177 |
| Happy | 0 |
| Haskell | 20.323476462551376 |
| Haxe | 9.023509566990532 |
| IDL | 1.01534495399968 |
| Idris | 0.36279318680267497 |
| Intel HEX | 0 |
| Isabelle | 4.389802167076498 |
| JAI | 2.220446049250313E-16 |
| JSON | 0 |
| JSONL | 0 |
| JSX | 0.9860839844113964 |
| Jade | 0.25000000000034117 |
| Janet | 9.719207406044006 |
| Java | 330.66188089718935 |
| JavaScript | 22.102491285372537 |
| JavaServer Pages | 4.31250095370342 |
| Jenkins Buildfile | 0 |
| Jinja | 2.5412145720173454E-50 |
| Julia | 12.542627036271085 |
| Julius | 0.24612165248208867 |
| Jupyter | 0 |
| Just | 0.3186038732601446 |
| Korn Shell | 40.89005232702741 |
| Kotlin | 0.3259347784770708 |
| LD Script | 3.7613336386434204 |
| LESS | 15.495439701029127 |
| LEX | 55.277186392539086 |
| LOLCODE | 13.578125958700468 |
| LaTeX | 3.316717967334341 |
| Lean | 21.194565176965895 |
| License | 0 |
| Lisp | 88.10676444837796 |
| Lua | 76.67247973843406 |
| Lucius | 0.3894241626790286 |
| Luna | 16.844066019174637 |
| MQL Header | 82.22436339969337 |
| MQL4 | 1.957314499740677 |
| MQL5 | 27.463183855085845 |
| MSBuild | 0.19561428198176206 |
| MUMPS | 5.960464477541773E-8 |
| Macromedia eXtensible Markup Language | 0 |
| Madlang | 6.75 |
| Makefile | 1.2287070602578574 |
| Mako | 1.3997604187154047E-8 |
| Markdown | 0 |
| Meson | 4.594536366188615 |
| Modula3 | 3.4375390004645627 |
| Module-Definition | 7.754887182446689 |
| Monkey C | 0.02734480644075532 |
| Mustache | 0.0000038370490074157715 |
| Nim | 0.8432132130061808 |
| Nix | 165.09375 |
| OCaml | 27.238212826702338 |
| Objective C | 32.250000004480256 |
| Objective C++ | 4.688333711547599 |
| Opalang | 3.2498599900436704 |
| Org | 2.4032862186444435 |
| Oz | 11.531631554476924 |
| PHP | 0.37573912739754056 |
| PKGBUILD | 0 |
| PSL Assertion | 4.470348358154297E-7 |
| Pascal | 274.7797153576955 |
| Patch | 0 |
| Perl | 42.73014043490598 |
| Plain Text | 0 |
| Polly | 0 |
| Pony | 0.2718505859375 |
| Powershell | 2.0956492198317282 |
| Processing | 11.358358417519032 |
| Prolog | 6.93889390390723E-17 |
| Properties File | 4.297774864451927 |
| Protocol Buffers | 5.013992889700926 |
| Puppet | 1.9962931947466012 |
| PureScript | 6.608705271035433 |
| Python | 15.208443286809963 |
| Q# | 0.4281108849922295 |
| QCL | 13.880147817629737 |
| QML | 16.17036877582475 |
| R | 5.355639399818855 |
| Rakefile | 0.4253943361101697 |
| Razor | 0.2500305203720927 |
| ReStructuredText | 0 |
| Report Definition Language | 1.8589575837924928E-119 |
| Robot Framework | 0 |
| Ruby | 8.696056880656087 |
| Ruby HTML | 0.031281024218515086 |
| Rust | 22.359375028118006 |
| SAS | 0.7712382248290134 |
| SKILL | 0.002197265625 |
| SPDX | 0 |
| SQL | 0.4963180149979617 |
| SRecode Template | 17.64534428715706 |
| SVG | 0.780306812508952 |
| Sass | 1.6041624981030795 |
| Scala | 2.7290137764062656 |
| Scheme | 18.68675828842983 |
| Scons | 9.985132321266597 |
| Shell | 19.757167057040007 |
| Smarty Template | 0.0009841919236350805 |
| Softbridge Basic | 4.76177694441164E-25 |
| Specman e | 0.1925095270881778 |
| Spice Netlist | 5.29710110812646 |
| Standard ML (SML) | 0.20708566564292288 |
| Stata | 0.04904100534194722 |
| Stylus | 4.534405773074049 |
| Swift | 1.8627019961192913E-9 |
| Swig | 11.786422730001505 |
| SystemVerilog | 0.00009708851624323821 |
| Systemd | 0 |
| TCL | 382.839838598133 |
| TOML | 0.37500173695180483 |
| TaskPaper | 0 |
| TeX | 8.266233975096164 |
| Thrift | 50.53134153016524 |
| Twig Template | 0 |
| TypeScript | 8.250029131770134 |
| TypeScript Typings | 37.89904005334354 |
| Unreal Script | 46.13322029508541 |
| Ur/Web | 0.04756343913582129 |
| Ur/Web Project | 6.776263578034403E-21 |
| V | 28.75797889154211 |
| VHDL | 37.47892257625405 |
| Vala | 74.26528331441615 |
| Varnish Configuration | 19.45791923156868 |
| Verilog | 4.165537942430622 |
| Verilog Args File | 0 |
| Vertex Shader File | 1.7979557178975683 |
| Vim Script | 0 |
| Visual Basic | 0.26300267116040704 |
| Visual Basic for Applications | 0.3985138943535276 |
| Vue | 5.039982162930666E-52 |
| Wolfram | 70.01674025323683 |
| Wren | 30694.003311276458 |
| XAML | 0.5000169009533838 |
| XCode Config | 13.653495818959595 |
| XML | 3.533205032457776 |
| XML Schema | 0 |
| Xtend | 19.279739396268607 |
| YAML | 1.1074293861154887 |
| Zig | 0.507775428428431 |
| Zsh | 6.769231127673729 |
| gitignore | 1.3347179947709417E-20 |
| ignore | 0.0356445312500015 |
| m4 | 5.4183238737327075 |
| nuspec | 3.640625 |
| sed | 6.423678000929861 |


### What are the most common filenames?

What filenames are most common across all code-bases ignoring extension and case?

Had you asked me before I started this I would have said, README, main, index, license. Thankfully the results reflect my thoughts pretty well. Although there are a lot of interesting ones in there. I have no idea why so many projects contain a file called `15` or `s15`. 

The makefile being the most common surprised me a little, but then I remembered it is used in many new JavaScript projects. Another interesting thing to note is that it appears jQuery is still king and reports of its death are greatly exaggerated, with it appearing as #4 on the list.

<div class="table-12"></div>

| file-name | count |
| -------- | ----- |
| makefile | 59,141,098 |
| index | 33,962,093 |
| readme | 22,964,539 |
| jquery | 20,015,171 |
| main | 12,308,009 |
| package | 10,975,828 |
| license | 10,441,647 |
| \__init__ | 10,193,245 |
| strings | 8,414,494 |
| android | 7,915,225 |
| config | 7,391,812 |
| default | 5,563,255 |
| build | 5,510,598 |
| setup | 5,291,751 |
| test | 5,282,106 |
| irq | 4,914,052 |
| 15 | 4,295,032 |
| country | 4,274,451 |
| pom | 4,054,543 |
| io | 3,642,747 |
| system | 3,629,821 |
| common | 3,629,698 |
| gpio | 3,622,587 |
| core | 3,571,098 |
| module | 3,549,789 |
| init | 3,378,919 |
| dma | 3,301,536 |
| bootstrap | 3,162,859 |
| application | 3,000,210 |
| time | 2,928,715 |
| cmakelists | 2,907,539 |
| plugin | 2,881,206 |
| base | 2,805,340 |
| s15 | 2,733,747 |
| androidmanifest | 2,727,041 |
| cache | 2,695,345 |
| debug | 2,687,902 |
| file | 2,629,406 |
| app | 2,588,208 |
| version | 2,580,288 |
| assemblyinfo | 2,485,708 |
| exception | 2,471,403 |
| project | 2,432,361 |
| util | 2,412,138 |
| user | 2,343,408 |
| clock | 2,283,091 |
| timex | 2,280,225 |
| pci | 2,231,228 |
| style | 2,226,920 |
| styles | 2,212,127 |

Note that due to memory constraints I made this process slightly lossy. Every 100 projects checked I would check the map and if an identified filename had < 10 counts it was dropped from the list. It could come back for the next run and if there was > 10 at this point it would remain. It shouldn't happen that often but it is possible the counts may be out by some amount if some common name appeared sparsely in the first batch of repositories before becoming common. In short they are not absolute numbers but should be close enough.

I could have used a trie structure to "compress" the space and gotten absolute numbers for this, but I didn't feel like writing one and just abused the map slightly to save enough memory and achieve my goal. I am however curious enough to try this out at a later date to see how a trie would perform.

### How many repositories appear to be missing a license?

This is an interesting one. Which repositories have an explicit license file somewhere? Note that the lack of a license file here does not mean that the project has none, as it might exist within the README or be indicated through SPDX comment tags in-line. it just means that `scc` could not find an explicit license file using its own criteria which at time of writing means a file ignoring case named "license", "licence", "copying", "copying3", "unlicense", "unlicence", "license-mit", "licence-mit" or "copyright".

Sadly it appears that the vast majority of repositories are missing a license. I would argue that all software should have a license for a variety of reasons but here is [someone else's take](https://www.infoworld.com/article/2839560/sticking-a-license-on-everything.html) on that.

| has license | count |
| ----------- | ----- |
| no | 6,502,753 |
| yes | 2,597,330 |

![scc-data license count](/static/an-informal-survey/hasLicense.png#center)

### How many projects use multiple .gitignore files?

Some may not know this but it is possible to have multiple .gitignore files in a git project. Given that fact how many projects use multiple .gitignore files? While we are looking how many have none?

What I did find that was interesting was one project that has 25,794 .gitignore files in its repository. The next highest was 2,547. I have no idea what is going on there. I had a brief look at it and it looks like they are used to allow checking in of the directories but I cannot confirm this.

Bringing this back to something sensible here is a plot of the data up to 20 .gitignore files and close to 99% of the total result.

![scc-data process load](/static/an-informal-survey/gitignorePerProject.png#center)

Something you would expect would be that the majority of projects would have either 0 or 1 .gitignore files. This is confirmed by the results with a massive drop-off of 10x for projects with 2 .gitignores. What was surprising to me was how many projects have more than a single .gitignore file. The long tail is especially long in this case.

I was also curious as to why some projects had thousands of .gitignore files. One of the main offenders appears to be forks of https://github.com/PhantomX/slackbuilds which all have ~2,547 .gitignore files. However the other repositories with 1000+ ignore files are listed below.

 - 25,794 files https://github.com/knot-git/rrs_publication_ocr
 - 1,523 files https://github.com/brunosimon/hetic 
 - 1,267 files https://github.com/par4all/par4all 
 - 1,186 files https://github.com/ramonluis1987/code_school 
 - 1,133 files https://github.com/signed/license-maven-plugin 

[skip table to next section](#which-language-developers-have-the-biggest-potty-mouth)

<div class="table-7"></div>

| .gitignore count | project count |
| -----------------| ------------- |
| 0 | 3,628,829 |
| 1 | 4,576,435 |
| 2 | 387,748 |
| 3 | 136,641 |
| 4 | 79,808 |
| 5 | 48,336 |
| 6 | 33,686 |
| 7 | 33,408 |
| 8 | 22,571 |
| 9 | 16,453 |
| 10 | 11,198 |
| 11 | 10,070 |
| 12 | 8,194 |
| 13 | 7,701 |
| 14 | 5,040 |
| 15 | 4,320 |
| 16 | 5,905 |
| 17 | 4,156 |
| 18 | 4,542 |
| 19 | 3,828 |
| 20 | 2,706 |
| 21 | 2,449 |
| 22 | 1,975 |
| 23 | 2,255 |
| 24 | 2,060 |
| 25 | 1,768 |
| 26 | 2,886 |
| 27 | 2,648 |
| 28 | 2,690 |
| 29 | 1,949 |
| 30 | 1,677 |
| 31 | 3,348 |
| 32 | 1,176 |
| 33 | 794 |
| 34 | 1,153 |
| 35 | 845 |
| 36 | 488 |
| 37 | 627 |
| 38 | 533 |
| 39 | 502 |
| 40 | 398 |
| 41 | 370 |
| 42 | 355 |
| 43 | 1,002 |
| 44 | 265 |
| 45 | 262 |
| 46 | 295 |
| 47 | 178 |
| 48 | 384 |
| 49 | 270 |
| 50 | 189 |
| 51 | 435 |
| 52 | 202 |
| 53 | 196 |
| 54 | 325 |
| 55 | 253 |
| 56 | 320 |
| 57 | 126 |
| 58 | 329 |
| 59 | 286 |
| 60 | 292 |
| 61 | 152 |
| 62 | 237 |
| 63 | 163 |
| 64 | 149 |
| 65 | 187 |
| 66 | 164 |
| 67 | 92 |
| 68 | 80 |
| 69 | 138 |
| 70 | 102 |
| 71 | 68 |
| 72 | 62 |
| 73 | 178 |
| 74 | 294 |
| 75 | 89 |
| 76 | 118 |
| 77 | 110 |
| 78 | 319 |
| 79 | 843 |
| 80 | 290 |
| 81 | 162 |
| 82 | 127 |
| 83 | 147 |
| 84 | 170 |
| 85 | 275 |
| 86 | 1,290 |
| 87 | 614 |
| 88 | 4,014 |
| 89 | 2,275 |
| 90 | 775 |
| 91 | 3,630 |
| 92 | 362 |
| 93 | 147 |
| 94 | 110 |
| 95 | 71 |
| 96 | 75 |
| 97 | 62 |
| 98 | 228 |
| 99 | 71 |
| 100 | 174 |
| 101 | 545 |
| 102 | 304 |
| 103 | 212 |
| 104 | 284 |
| 105 | 516 |
| 106 | 236 |
| 107 | 39 |
| 108 | 69 |
| 109 | 131 |
| 110 | 82 |
| 111 | 102 |
| 112 | 465 |
| 113 | 621 |
| 114 | 47 |
| 115 | 59 |
| 116 | 43 |
| 117 | 40 |
| 118 | 43 |
| 119 | 443 |
| 120 | 72 |
| 121 | 42 |
| 122 | 33 |
| 123 | 392 |
| 124 | 66 |
| 125 | 46 |
| 126 | 381 |
| 127 | 19 |
| 128 | 99 |
| 129 | 906 |
| 130 | 52 |
| 131 | 19 |
| 132 | 11 |
| 133 | 99 |
| 134 | 10 |
| 135 | 15 |
| 136 | 6 |
| 137 | 22 |
| 138 | 44 |
| 139 | 33 |
| 140 | 24 |
| 141 | 33 |
| 142 | 39 |
| 143 | 48 |
| 144 | 80 |
| 145 | 20 |
| 146 | 28 |
| 147 | 19 |
| 148 | 17 |
| 149 | 11 |
| 150 | 20 |
| 151 | 57 |
| 152 | 35 |
| 153 | 24 |
| 154 | 31 |
| 155 | 35 |
| 156 | 55 |
| 157 | 89 |
| 158 | 57 |
| 159 | 88 |
| 160 | 18 |
| 161 | 47 |
| 162 | 56 |
| 163 | 36 |
| 164 | 63 |
| 165 | 99 |
| 166 | 44 |
| 167 | 64 |
| 168 | 86 |
| 169 | 70 |
| 170 | 111 |
| 171 | 106 |
| 172 | 25 |
| 173 | 39 |
| 174 | 14 |
| 175 | 25 |
| 176 | 53 |
| 177 | 20 |
| 178 | 56 |
| 179 | 11 |
| 180 | 7 |
| 181 | 40 |
| 182 | 32 |
| 183 | 17 |
| 184 | 68 |
| 185 | 38 |
| 186 | 16 |
| 187 | 3 |
| 188 | 4 |
| 189 | 2 |
| 190 | 12 |
| 191 | 18 |
| 192 | 37 |
| 193 | 9 |
| 194 | 10 |
| 195 | 11 |
| 196 | 18 |
| 197 | 45 |
| 198 | 27 |
| 199 | 11 |
| 200 | 39 |
| 201 | 23 |
| 202 | 37 |
| 203 | 22 |
| 204 | 21 |
| 205 | 7 |
| 206 | 40 |
| 207 | 7 |
| 208 | 8 |
| 209 | 16 |
| 210 | 29 |
| 211 | 20 |
| 212 | 21 |
| 213 | 7 |
| 214 | 4 |
| 215 | 12 |
| 217 | 21 |
| 218 | 13 |
| 220 | 12 |
| 221 | 2 |
| 222 | 15 |
| 223 | 4 |
| 224 | 12 |
| 225 | 9 |
| 226 | 1 |
| 227 | 8 |
| 228 | 3 |
| 229 | 6 |
| 230 | 8 |
| 231 | 31 |
| 232 | 26 |
| 233 | 6 |
| 234 | 17 |
| 235 | 6 |
| 236 | 23 |
| 237 | 1 |
| 238 | 11 |
| 239 | 2 |
| 240 | 10 |
| 241 | 7 |
| 242 | 11 |
| 243 | 1 |
| 244 | 14 |
| 245 | 21 |
| 246 | 3 |
| 247 | 12 |
| 248 | 1 |
| 249 | 6 |
| 250 | 10 |
| 251 | 5 |
| 252 | 18 |
| 253 | 7 |
| 254 | 17 |
| 255 | 4 |
| 256 | 16 |
| 257 | 8 |
| 258 | 24 |
| 259 | 17 |
| 260 | 4 |
| 261 | 1 |
| 262 | 3 |
| 263 | 12 |
| 264 | 3 |
| 265 | 8 |
| 267 | 2 |
| 268 | 1 |
| 269 | 3 |
| 271 | 4 |
| 272 | 1 |
| 273 | 1 |
| 274 | 1 |
| 275 | 3 |
| 276 | 6 |
| 279 | 5 |
| 280 | 1 |
| 281 | 1 |
| 284 | 4 |
| 285 | 1 |
| 286 | 1 |
| 288 | 2 |
| 289 | 1 |
| 290 | 5 |
| 291 | 4 |
| 293 | 7 |
| 294 | 4 |
| 295 | 1 |
| 296 | 1 |
| 297 | 1 |
| 299 | 70 |
| 300 | 2 |
| 301 | 4 |
| 302 | 1 |
| 303 | 7 |
| 305 | 1 |
| 306 | 2 |
| 307 | 2 |
| 309 | 1 |
| 310 | 7 |
| 311 | 1 |
| 313 | 14 |
| 316 | 1 |
| 320 | 1 |
| 321 | 6 |
| 322 | 2 |
| 323 | 3 |
| 324 | 4 |
| 327 | 4 |
| 328 | 2 |
| 329 | 1 |
| 330 | 13 |
| 331 | 5 |
| 332 | 11 |
| 333 | 3 |
| 334 | 1 |
| 335 | 1 |
| 336 | 11 |
| 337 | 1 |
| 338 | 20 |
| 339 | 11 |
| 340 | 2 |
| 341 | 6 |
| 342 | 10 |
| 343 | 37 |
| 344 | 25 |
| 345 | 9 |
| 346 | 32 |
| 347 | 4 |
| 348 | 9 |
| 349 | 7 |
| 350 | 12 |
| 351 | 2 |
| 352 | 5 |
| 354 | 7 |
| 358 | 32 |
| 359 | 7 |
| 360 | 6 |
| 361 | 1 |
| 362 | 21 |
| 363 | 14 |
| 364 | 51 |
| 365 | 17 |
| 367 | 18 |
| 368 | 9 |
| 370 | 7 |
| 371 | 6 |
| 372 | 15 |
| 373 | 1 |
| 374 | 38 |
| 375 | 113 |
| 376 | 57 |
| 377 | 37 |
| 378 | 23 |
| 379 | 87 |
| 380 | 65 |
| 382 | 1 |
| 386 | 2 |
| 388 | 1 |
| 391 | 5 |
| 392 | 1 |
| 394 | 1 |
| 397 | 3 |
| 401 | 1 |
| 403 | 1 |
| 408 | 1 |
| 409 | 2 |
| 410 | 5 |
| 411 | 1 |
| 413 | 4 |
| 415 | 1 |
| 418 | 1 |
| 420 | 1 |
| 427 | 3 |
| 428 | 2 |
| 430 | 2 |
| 433 | 314 |
| 437 | 1 |
| 450 | 2 |
| 453 | 1 |
| 468 | 1 |
| 469 | 1 |
| 483 | 5 |
| 484 | 1 |
| 486 | 1 |
| 488 | 2 |
| 489 | 9 |
| 490 | 4 |
| 492 | 2 |
| 493 | 106 |
| 494 | 3 |
| 495 | 1 |
| 496 | 2 |
| 498 | 1 |
| 512 | 1 |
| 539 | 1 |
| 553 | 1 |
| 560 | 2 |
| 570 | 2 |
| 600 | 1 |
| 602 | 3 |
| 643 | 1 |
| 646 | 2 |
| 657 | 1 |
| 663 | 1 |
| 670 | 1 |
| 672 | 2 |
| 729 | 5 |
| 732 | 1 |
| 739 | 1 |
| 744 | 1 |
| 759 | 1 |
| 778 | 1 |
| 819 | 1 |
| 859 | 1 |
| 956 | 1 |
| 959 | 2 |
| 964 | 2 |
| 965 | 1 |
| 973 | 1 |
| 1,133 | 1 |
| 1,186 | 1 |
| 1,267 | 2 |
| 1,523 | 1 |
| 2,535 | 1 |
| 2,536 | 1 |
| 2,537 | 2 |
| 2,539 | 1 |
| 2,540 | 1 |
| 2,541 | 5 |
| 2,542 | 1 |
| 2,545 | 1 |
| 2,547 | 1 |
| 25,794 | 1 |

### Which language developers have the biggest potty mouth?

Working this out is not an exact science. It falls into the NLP class of problems really. Picking up cursing/swearing or offensive terms using filenames from a defined list is never going to be effective. If you do a simple string contains test you pick up all sorts or normal files such as `assemble.sh` and such. So to produce the following I pulled a list of curse words, then checked if any files in each project start with one of those values followed by a period. This would mean a file named `gangbang.java` would be picked up while `assemble.sh` would not. However this is going to miss all sorts of cases such as `pu55syg4rgle.java` and other such crude names.

The list I used contained some leet speak such as `b00bs` and `b1tch` to try and catch some of the most interesting cases. The full list is [here](/static/an-informal-survey/curse.txt).

While not accurate at all as mentioned it is incredibly fun to see what this produces. So lets start with a list of which languages have the most curse words. However we should probably weight this against how much code exists as well. So here are the top ones.

| language | filename curse count | percent of files |
| -------- | -------------------- | ---------------- |
| C Header | 7,660 | 0.00126394567906% |
| Java | 7,023 | 0.00258792635479% |
| C | 6,897 | 0.00120706524533% |
| PHP | 5,713 | 0.00283428484703% |
| JavaScript | 4,306 | 0.00140692338568% |
| HTML | 3,560 | 0.00177646776919% |
| Ruby | 3,121 | 0.00223136542655% |
| JSON | 1,598 | 0.00293688627715% |
| C++ | 1,543 | 0.00135977378652% |
| Dart | 1,533 | 0.19129310646% |
| Rust | 1,504 | 0.038465935524% |
| Go Template | 1,500 | 0.0792233157387% |
| SVG | 1,234 | 0.00771043360379% |
| XML | 1,212 | 0.000875741051608% |
| Python | 1,092 | 0.00119138129893% |
| JavaServer Pages | 1,037 | 0.0215440542669% |

Interesting! My first thought was "those naughty C developers!" but as it turns out while they have a high count they write so much code it probably isn't that big a deal. However pretty clearly Dart developers have an axe to grind! If you know someone coding in Dart you may want to go offer them a hug.

I also want to know what are the most commonly used curse words. Lets see how dirty a mind we have collectively. A few of the top ones I could see being legitimate names (if you squint), but the majority would certainly produce few comments in a PR and a raised eyebrow.

| word | count |
| ---- | ----- |
| ass | 11,358 |
| knob | 10,368 |
| balls | 8,001 |
| xxx | 7,205
| sex | 5,021 |
| nob | 3,385 |
| pawn | 2,919 |
| hell | 2,819 |
| crap | 1,112 |
| anal | 950 |
| snatch | 885 |
| fuck | 572 |
| poop | 510 |
| cox | 476 |
| shit | 383 |
| lust | 367 |
| butt | 265 |
| bum | 151 |
| bugger | 132 |
| pron | 121 |
| cum | 118 |
| cok | 112 |
| damn | 105 |

Note that some of the more offensive words in the list did have matching filenames which I find rather shocking considering what they were. Thankfully they were not very common and didn't make my list above which was limited to those which had counts over 100. I am hoping that those files only exist for testing allow/deny lists and such.

### Longest files by lines per language

As you would probably expect Plain Text, SQL, XML, JSON and CSV take the top positions of this one, seeing as they usually contain meta-data, database dumps and the like.

Limited to 40 because at some point there is only a hello world example or such available and the result is not very interesting. It is not surprising to see that someone has checked in `sqlite3.c` somewhere but I would be a little worried about that 3,064,594 line Python file and that 1,997,637 line TypeScript monster.

**NB** Some of the links below MAY not translate 100% due to throwing away some information when I created the files. Most should work, but a few you may need to mangle the URL to resolve.

[skip table to next section](#whats-the-largest-file-for-each-language)

<div class="table-4"></div>

| language | filename | lines |
| -------- | -------- | ----- |
| Plain Text | <a href="https://github.com/igorshing/flow/blob/master/data/1366100696temp.txt">1366100696temp.txt</a> | 347,671,811 |
| PHP | <a href="https://github.com/aadityajs/keylinkz/blob/master/file/log/phpfox_error_log_04_04_12_3d4b11f6ee2a89fd5ace87c910cee04b.php">phpfox_error_log_04_04_12_3d4b11f6ee2a89fd5ace87c910cee04b.php</a> | 121,930,973 |
| HTML | <a href="https://github.com/vanbug/codes/blob/master/yo.html">yo.html</a> | 54,596,752 |
| LEX | <a href="https://github.com/phoenixiizero/snes9x-3d/blob/master/gtk/l">l</a> | 39,743,785 |
| XML | <a href="https://bitbucket.com/mellson/dblp-statistics/src/master/SBDM Exercise 2/dblp.xml">dblp.xml</a> | 39,445,222 |
| Autoconf | <a href="https://github.com/rubusch/usi-compilers/blob/master/082__compiler/testsrepo/04while/21-t2.in">21-t2.in</a> | 33,526,784 |
| CSV | <a href="https://github.com/allisonbmccoy/smart-summarization/blob/master/knowledgebases/ontology.csv">ontology.csv</a> | 31,946,031 |
| Prolog | <a href="https://github.com/jrvalcourt/salary-prediction/blob/master/temp/top500_full.p">top500_full.p</a> | 22,428,770 |
| JavaScript | <a href="https://bitbucket.com/eternum/rails-facebook-webgl/src/master/app/assets/three/mirus.js">mirus.js</a> | 22,023,354 |
| JSON | <a href="https://github.com/nicknisi/safeomaha-data/blob/master/formatted_data/douglasCountyVoterRegistration.json">douglasCountyVoterRegistration.json</a> | 21,104,668 |
| Game Maker Language | <a href="https://github.com/shajain/i590project/blob/master/lgEvans/lg.gml">lg.gml</a> | 13,302,632 |
| C Header | <a href="https://github.com/alinefr/netcat-cpi-kernel-module/blob/master/tracks/trk6data.h">trk6data.h</a> | 13,025,371 |
| Objective C++ | <a href="https://bitbucket.com/vanghdi/yelp/src/master/data/yelp/review-1.mm">review-1.mm</a> | 12,788,052 |
| SQL | <a href="https://github.com/rigidus/bzzr/blob/master/newdump.sql">newdump.sql</a> | 11,595,909 |
| Patch | <a href="https://github.com/lnava/cs411g1/blob/master/clook_iosched-team01.patch">clook_iosched-team01.patch</a> | 10,982,879 |
| YAML | <a href="https://github.com/cclausen/communtu/blob/master/db/data.yml">data.yml</a> | 10,764,489 |
| SVG | <a href="https://github.com/ruby-gnome2/ruby-gnome2/blob/master/rsvg2/test/tmp/large-file.svg">large-file.svg</a> | 10,485,763 |
| Sass | <a href="https://bitbucket.com/abegarcia/lifyember/src/master/node_modules/node-sass/libsass/sass-spec/spec/benchmarks/large_empty.scss">large_empty.scss</a> | 10,000,000 |
| Assembly | <a href="https://github.com/bogwonch/thesis/blob/master/MIPS/1-INST-JMP/J.s">J.s</a> | 8,388,608 |
| LaTeX | <a href="https://github.com/zbhappy/c/blob/master/file/tex">tex</a> | 8,316,556 |
| C++ Header | <a href="https://github.com/algotrust/qmlib/blob/master/include/qmlib/math/random/impl/primpoly_impl.hh">primpoly_impl.hh</a> | 8,129,599 |
| Lisp | <a href="https://github.com/ayoub-89/nltk_data/blob/master/corpora/lin_thesaurus/simN.lsp">simN.lsp</a> | 7,233,972 |
| Perl | <a href="https://github.com/logicmoo/jellyfish/blob/master/temp/aimlCore3.pl">aimlCore3.pl</a> | 6,539,759 |
| SAS | <a href="https://github.com/herry13/fdt/blob/master/benchmark/cloud/cloud.old2/output.sas">output.sas</a> | 5,874,153 |
| C | <a href="https://github.com/anumq/protein/blob/master/Imp6/sourceFiles/CathDomainDescriptionFile.v3.5.c">CathDomainDescriptionFile.v3.5.c</a> | 5,440,052 |
| Lua | <a href="https://github.com/nan0meter/helix/blob/master/Content/Scenes/giant/giant.lua">giant.lua</a> | 5,055,019 |
| R | <a href="https://bitbucket.com/richardberendsen/rdwps/src/master/R/disambisearches.R">disambisearches.R</a> | 4,985,492 |
| MUMPS | <a href="https://bitbucket.com/dzhang50/gpm/src/master/spec2006/450.soplex/ref.mps">ref.mps</a> | 4,709,289 |
| HEX | <a href="https://bitbucket.com/bcforres/18545/src/master/roms/bin/combine.hex">combine.hex</a> | 4,194,304 |
| Python | <a href="https://github.com/anirudhvenkats/clowdflows/blob/master/workflows/segmine/data/mappings.py">mappings.py</a> | 3,064,594 |
| Scheme | <a href="https://github.com/cosmoharrigan/test-datasets/blob/master/pln/tuffy/smokes/tests/03-08-14/atomspace.scm">atomspace.scm</a> | 3,027,366 |
| C++ | <a href="https://github.com/arandur/prog3b/blob/master/src/Int.cpp">Int.cpp</a> | 2,900,609 |
| Properties File | <a href="https://github.com/gunner14/old_rr_code/blob/master/java_workplace/sns-xiaonei/xiaonei-guide/trunk/src/main/resources/nuomi_active_user_ids.properties">nuomi_active_user_ids.properties</a> | 2,747,671 |
| Alex | <a href="https://bitbucket.com/brendenl92/coldest-steeliest/src/master/Cold Steel/Release/media/Models/Dalek.X">Dalek.X</a> | 2,459,209 |
| TCL | <a href="https://github.com/pdsteele/socialnetworksproject/blob/master/datasets/TCL">TCL</a> | 2,362,970 |
| Ruby | <a href="https://github.com/bonifacefr/oddborg-1/blob/master/ext/fiparse/test/data/smj_12_2004.rb">smj_12_2004.rb</a> | 2,329,560 |
| Wolfram | <a href="https://github.com/liei/shoebox/blob/master/hmm.nb">hmm.nb</a> | 2,177,422 |
| Brainfuck | <a href="https://github.com/bletchley13/signature-based-malware-detection/blob/master/FFBloomFilter/BF">BF</a> | 2,097,158 |
| TypeScript | <a href="https://github.com/selbst/handy-chem/blob/master/HandwritingV010/testingcharacters/0/all_6.ts">all_6.ts</a> | 1,997,637 |
| Module-Definition | <a href="https://github.com/fukayatsu/kani_acid/blob/master/lib/mecab-naist-jdic/matrix.def">matrix.def</a> | 1,948,817 |
| LESS | <a href="https://github.com/er2/euler/blob/master/less">less</a> | 1,930,356 |
| Objective C | <a href="https://github.com/chinnabommu/computervisionwip/blob/master/imports/fast/fast-matlab-src-2.1/faster.m">faster.m</a> | 1,913,966 |
| Org | <a href="https://github.com/dyfeng/linuxscripts/blob/master/sogou_dict_import/default.org">default.org</a> | 1,875,096 |
| Jupyter | <a href="https://github.com/dunovank/pynb/blob/master/RADD/RADD/Proactive/ProHDDM/X/ReHDDM - AllGo sxFits-Copy0.ipynb">ReHDDM - AllGo sxFits-Copy0.ipynb</a> | 1,780,197 |
| Specman e | <a href="https://github.com/yecol/yecolhubio/blob/master/twitter/twitter.e">twitter.e</a> | 1,768,135 |
| F* | <a href="https://gitlab.com/projet_alphasat/projet_alphasat/blob/master/Pan_troglodytes_monomers.fst">Pan_troglodytes_monomers.fst</a> | 1,739,878 |
| Systemd | <a href="https://github.com/cocoxu/shakespeare/blob/master/models/Translators/video_corpus_baseline/data/video_clean_lower_tokenized.target">video_clean_lower_tokenized.target</a> | 1,685,570 |
| V | <a href="https://github.com/thepedestrian/fpga-simple-maze-game-using-vga-output/blob/master/SRC/FPGA-Verilog-Code/ImageMazeChannelValueROM.v">ImageMazeChannelValueROM.v</a> | 1,440,068 |
| Markdown | <a href="https://github.com/subjectraw/subjectraw/blob/master/subject/data/eng/eukaryota.md">eukaryota.md</a> | 1,432,161 |
| TeX | <a href="https://github.com/rogerbraun/wadoku-scripts/blob/master/japanischtest.tex">japanischtest.tex</a> | 1,337,456 |
| Forth | <a href="https://github.com/mar-one/machine-trans-shared-task/blob/master/data/corpus/europarl.tok.fr">europarl.tok.fr</a> | 1,288,074 |
| Shell | <a href="https://github.com/jcs/openbsd-commitid/blob/master/out/add_commitids_to_src.sh">add_commitids_to_src.sh</a> | 1,274,873 |
| SKILL | <a href="https://github.com/jacktsai/tos/blob/master/9.13/Assembly-CSharp/hijacked.il">hijacked.il</a> | 1,187,701 |
| CSS | <a href="https://github.com/pligo/icantgo/blob/master/web/css/7f116c3.css">7f116c3.css</a> | 1,170,216 |
| C# | <a href="https://github.com/sdanil/test/blob/master/Form1.cs">Form1.cs</a> | 1,140,480 |
| gitignore | <a href="https://github.com/arikanu/harpy/blob/master/harpy/.gitignore">.gitignore</a> | 1,055,167 |
| Boo | <a href="https://github.com/ph111p/bwinf_2014_runde2/blob/master/Aufgabe_1/3.out.tex">3.out.tex</a> | 1,032,145 |
| Java | <a href="https://github.com/castlely/compiler_tiger_lab1/blob/master/test/Monster.java">Monster.java</a> | 1,000,019 |
| ActionScript | <a href="https://github.com/tarzzz/ds/blob/master/linked-list/as">as</a> | 1,000,000 |
| MSBuild | <a href="https://github.com/chrisleewashere/swirl-osx/blob/master/corpus/train.props">train.props</a> | 989,860 |
| D | <a href="https://github.com/khalefa/dataset/blob/master/D">D</a> | 883,308 |
| Coq | <a href="https://bitbucket.com/mpettersson/reinsverifier/src/master/REINS/CompiledDFAs.v">CompiledDFAs.v</a> | 873,354 |
| Clojure | <a href="https://github.com/candera/cs-atom/blob/master/raw-data.clj">raw-data.clj</a> | 694,202 |
| Swig | <a href="https://bitbucket.com/roman_shafeyev/navsystem/src/master/workspace/NavCenter/3DEditor/3DEditor.i">3DEditor.i</a> | 645,117 |
| Happy | <a href="https://github.com/symbolicpower/perceptrondict/blob/master/y">y</a> | 624,673 |
| GLSL | <a href="https://github.com/aminems/opendwarfs/blob/master/test/n-body-methods/gem/capsid.vert">capsid.vert</a> | 593,618 |
| Verilog | <a href="https://github.com/sammsiontir/eecs470_superernie/blob/master/synth_report_0420/pipeline.vg">pipeline.vg</a> | 578,418 |
| Standard ML (SML) | <a href="https://github.com/amtriathlon/goldencheetah/blob/master/test/rides/Ambit3-HRVbutNoHR.sml">Ambit3-HRVbutNoHR.sml</a> | 576,071 |
| SystemVerilog | <a href="https://gitlab.com/andresavilas/ece337project/blob/master/mapped/bitcoinminer.v">bitcoinminer.v</a> | 561,974 |
| Visual Basic | <a href="https://bitbucket.com/mscalella/pronto/src/master/BussinessLogic/linqStoreProcs.designer.vb">linqStoreProcs.designer.vb</a> | 561,067 |
| Go | <a href="https://github.com/reusee/go-qt/blob/master/smoke_info/info.go">info.go</a> | 559,236 |
| Expect | <a href="https://github.com/jalwes/30_year_average/blob/master/Argonne_Hourly_Data/Argonne_hourly_dewpoint.exp">Argonne_hourly_dewpoint.exp</a> | 552,269 |
| Erlang | <a href="https://github.com/siteview/erlide/blob/master/com.siteview.kernel.core/modules/nnsdh/sdh_analogue_data.erl">sdh_analogue_data.erl</a> | 473,924 |
| Makefile | <a href="https://github.com/kssanath/ece497/blob/master/qt-everywhere-opensource-src-4.6.2/src/3rdparty/webkit/WebCore/Makefile">Makefile</a> | 462,433 |
| QML | <a href="https://github.com/kasra-hosseini/obspydmt/blob/master/obspyDMT/gcmt_catalog/COMBO/2005.qml">2005.qml</a> | 459,113 |
| SPDX | <a href="https://github.com/triplecheck/misc/blob/master/misc_spdx/linux-coreos.spdx">linux-coreos.spdx</a> | 444,743 |
| VHDL | <a href="https://github.com/mbarga/archive-cpu/blob/master/mapped/cpuTest.vhd">cpuTest.vhd</a> | 442,043 |
| ASP.NET | <a href="https://github.com/claq2/lcbodrinkfinder/blob/master/TestLcbo/AllProducts.aspx">AllProducts.aspx</a> | 438,423 |
| XML Schema | <a href="https://github.com/dvhill/impengineering/blob/master/xsd/rsx/7.4.1/856/AdvanceShipNotices.xsd">AdvanceShipNotices.xsd</a> | 436,055 |
| Elixir | <a href="https://github.com/lulance/courseranlangp/blob/master/h1-p/gene.train.with.rare.ex">gene.train.with.rare.ex</a> | 399,995 |
| Macromedia eXtensible Markup Language | <a href="https://bitbucket.com/jackcviers/flex-3-flex-4-performance-test/src/master/Flex4PerformanceTest/src/StaticFlex4PerformanceTest20000.mxml">StaticFlex4PerformanceTest20000.mxml</a> | 399,821 |
| Ada | <a href="https://github.com/raghup17/bmm/blob/master/hls/bmm_top/solution1/.autopilot/db/bmm_top.adb">bmm_top.adb</a> | 390,275 |
| TypeScript Typings | <a href="https://github.com/vansimke/dojotypedescriptiongenerator/blob/master/DojoTypeDescriptor/Scripts/typings/dojox.d.ts">dojox.d.ts</a> | 384,171 |
| Pascal | <a href="https://github.com/grahamegrieve/fhirserver/blob/master/library/r4/FHIR.R4.Resources.pas">FHIR.R4.Resources.pas</a> | 363,291 |
| COBOL | <a href="https://github.com/nesl/scpi-scripts/blob/master/kei/cpy">cpy</a> | 358,745 |
| Basic | <a href="https://github.com/jcorrius/go-oo-mingw32-soc/blob/master/test/macro/vba_streams/excel-vba-streams-#1.bas">excel-vba-streams-#1.bas</a> | 333,707 |
| Visual Basic for Applications | <a href="https://github.com/franc90/furnacedriver/blob/master/FurnaceDriver/FurnaceDriver_rpy/Default/Dispatcher.cls">Dispatcher.cls</a> | 332,266 |
| Puppet | <a href="https://github.com/malkia/multi_msvcrt/blob/master/main_110.pp">main_110.pp</a> | 314,217 |
| FORTRAN Legacy | <a href="https://github.com/yinyanghu/dmar/blob/master/Code/SVM/resultsX/f">f</a> | 313,599 |
| OCaml | <a href="https://github.com/merelyapseudonym/afp/blob/master/thys/Flyspeck-Tame/Archives/Pent.ML">Pent.ML</a> | 312,749 |
| FORTRAN Modern | <a href="https://github.com/johannesgerer/jburkardt-f/blob/master/slatec/slatec.f90">slatec.f90</a> | 298,677 |
| CoffeeScript | <a href="https://github.com/cherifya/letterpresser/blob/master/js/dictionary.coffee">dictionary.coffee</a> | 271,378 |
| Nix | <a href="https://github.com/edolstra/nixpkgs/blob/master/pkgs/development/haskell-modules/hackage-packages.nix">hackage-packages.nix</a> | 259,940 |
| Intel HEX | <a href="https://github.com/giorgio130/linux-2635-kobo-multitouch/blob/master/firmware/imx/epdc_ED060SCE.fw.ihex">epdc_ED060SCE.fw.ihex</a> | 253,836 |
| Scala | <a href="https://github.com/nette22/threeflowjs/blob/master/examples/renders/models_camaro.sc">models_camaro.sc</a> | 253,559 |
| Julia | <a href="https://github.com/carljv/bayescomp/blob/master/src/julia/*IJulia 0*.jl">*IJulia 0*.jl</a> | 221,058 |
| SRecode Template | <a href="https://github.com/malex984/mysettings/blob/master/.epsilon/espell.srt">espell.srt</a> | 216,243 |
| sed | <a href="https://github.com/sncosmo/sncosmohubio/blob/master/data/models/pierel/CSP-2004fe.SED">CSP-2004fe.SED</a> | 214,290 |
| ReStructuredText | <a href="https://github.com/ledmonster/japanese-law/blob/master/doc/S40/S40HO033.rst">S40HO033.rst</a> | 211,403 |
| Bosque | <a href="https://github.com/jalbertbowden/world-data/blob/master/world-dem-5-arcmin/world_dem_5arcmin_geo.bsq/world_dem_5arcmin_geo.bsq">world_dem_5arcmin_geo.bsq</a> | 199,238 |
| Emacs Lisp | <a href="https://github.com/cmungall/uberon/blob/master/util/ubermacros.el">ubermacros.el</a> | 195,861 |
| F# | <a href="https://bitbucket.com/akohlmey/lammps/src/master/examples/USER/misc/momb/Ag_O1X5.5_O2X0.55.eam.fs">Ag_O1X5.5_O2X0.55.eam.fs</a> | 180,008 |
| GDScript | <a href="https://github.com/hugo53/vinearts/blob/master/giaoduc.net.vn/Xa-hoi/Chum-anh-Phat-hien-chan-dong-tai-chua-Dam-Bac-Ninh/72906.gd">72906.gd</a> | 178,628 |
| Gherkin Specification | <a href="https://github.com/czlyc/2c-web-research/blob/master/data/raw/8/feature">feature</a> | 175,229 |
| Haskell | <a href="https://github.com/jjinkou2/agentcom/blob/master/Excel.hs">Excel.hs</a> | 173,039 |
| Dart | <a href="https://github.com/mattliberty/wordgen/blob/master/source/surnames_list.dart">surnames_list.dart</a> | 153,144 |
| Bazel | <a href="https://github.com/nonas/debian-clang/blob/master/tests/build_timeout/buildlogs/run1/matplotlib_1.3.1-1_amd64-20140427-1441.build">matplotlib_1.3.1-1_amd64-20140427-1441.build</a> | 149,234 |
| Haxe | <a href="https://github.com/ivideo/easytake/blob/master/FFmpeg-iOS-Encoder-master/hahah/ffmpeg-iOS/yasm-1.1.0/results/elf-x86id.hx">elf-x86id.hx</a> | 145,800 |
| IDL | <a href="https://github.com/brownplt/strobe/blob/master/data/all-idls.idl">all-idls.idl</a> | 129,435 |
| LD Script | <a href="https://github.com/t-crest/ospat/blob/master/misc/ldscripts/patmos/prep/kernel_partitions.lds">kernel_partitions.lds</a> | 127,187 |
| Monkey C | <a href="https://github.com/jlesniewski/pycrysfml/blob/master/hklgen/LFO/M3/LFO_BT1-point.mc">LFO_BT1-point.mc</a> | 120,881 |
| Modula3 | <a href="https://github.com/damsl/k3-mosaic/blob/master/tests/m3/tpch22.m3">tpch22.m3</a> | 120,185 |
| Batch | <a href="https://github.com/malific/script-shop/blob/master/EZhunter.cmd">EZhunter.cmd</a> | 119,341 |
| Rust | <a href="https://github.com/jankeromnes/gecko-dev/blob/master/third_party/rust/encoding_rs/src/data.rs">data.rs</a> | 114,408 |
| Ur/Web | <a href="https://github.com/deepuiitk/indian-parallel-corpora/blob/master/ur-en/tok/dict.ur-en.ur">dict.ur-en.ur</a> | 113,911 |
| Unreal Script | <a href="https://github.com/polyatail/scher_et_al_2013/blob/master/Figure_3/Panel_C/orfs.derep_id97.uc">orfs.derep_id97.uc</a> | 110,737 |
| Groovy | <a href="https://bitbucket.com/nfredricks/vim-files/src/master/tags/groovy">groovy</a> | 100,297 |
| Smarty Template | <a href="https://github.com/rodneyrehm/php-template-engines/blob/master/test/tests/smarty3/templates/assign.100000.tpl">assign.100000.tpl</a> | 100,002 |
| Bitbake | <a href="https://github.com/kellyhou/clientserver/blob/master/Examples/sys_net/io/bb">bb</a> | 100,000 |
| BASH | <a href="https://github.com/palmer-dabbelt/tek/blob/master/test/tek/palmer-master-thesis.bash">palmer-master-thesis.bash</a> | 96,911 |
| PSL Assertion | <a href="https://github.com/khayer/blat_parser/blob/master/lib/blat_parser/test_uno.psl">test_uno.psl</a> | 96,253 |
| ASP | <a href="https://github.com/grote/oclingo/blob/master/tests/gbie/instances/sat_gbie_01.asp">sat_gbie_01.asp</a> | 95,144 |
| Protocol Buffers | <a href="https://github.com/burrows-labs/sqllogictest/blob/master/proto/select1.proto">select1.proto</a> | 89,796 |
| Report Definition Language | <a href="https://github.com/vinhdoan/angroupdemo/blob/master/Reports/ACG.rdl">ACG.rdl</a> | 84,666 |
| Powershell | <a href="https://github.com/dwj7738/my-powershell-repository/blob/master/Modules/WPK/GeneratedControls/PresentationFramework.ps1">PresentationFramework.ps1</a> | 83,861 |
| Jinja | <a href="https://github.com/truebluedata/chakravyuhaiiitd/blob/master/jinja2">jinja2</a> | 76,040 |
| AWK | <a href="https://github.com/apurtell/llvm-test-suite/blob/master/MultiSource/Benchmarks/MallocBench/perl/INPUT/words-large.awk">words-large.awk</a> | 69,964 |
| LOLCODE | <a href="https://github.com/eldog/fface/blob/master/src/fsort/lol">lol</a> | 67,520 |
| Wren | <a href="https://github.com/munificent/wren/blob/master/test/limit/reuse_constants.wren">reuse_constants.wren</a> | 65,550 |
| JSX | <a href="https://github.com/imclab/drawgrid/blob/master/scripts/AEscript.jsx">AEscript.jsx</a> | 65,108 |
| Rakefile | <a href="https://github.com/vajapravin/pokemon_rails/blob/master/lib/tasks/seed.rake">seed.rake</a> | 63,000 |
| Stata | <a href="https://github.com/funzelknut/migrationsvariable/blob/master/Stata/DoFile/Logic/combinations/.31113.do">.31113.do</a> | 60,343 |
| Vim Script | <a href="https://github.com/yuratomo/cpp-api-ddk/blob/master/autoload/cppapi/ddk.vim">ddk.vim</a> | 60,282 |
| Swift | <a href="https://github.com/alexeyxo/protobuf-swift/blob/master/plugin/Tests/pbTests/Google.Protobuf.UnittestEnormousDescriptor.proto.swift">Google.Protobuf.UnittestEnormousDescriptor.proto.swift</a> | 60,236 |
| Korn Shell | <a href="https://github.com/ggouaillardet/ompi-www/blob/master/community/lists/devel/attachments/20090617/3f35b1fc/attachment-0002.ksh">attachment-0002.ksh</a> | 58,298 |
| AsciiDoc | <a href="https://github.com/cafe008/spring-framework/blob/master/src/asciidoc/index.adoc">index.adoc</a> | 52,627 |
| Freemarker Template | <a href="https://github.com/livesense/orglivesensesamplesimpleportal/blob/master/src/main/resources/apps/simpleportal/htmlmail/designed.eml.ftl">designed.eml.ftl</a> | 52,160 |
| Cython | <a href="https://github.com/szeng2013/eecs392/blob/master/391_Project/CALC/CALC.pex.netlist.CALC.pxi">CALC.pex.netlist.CALC.pxi</a> | 50,283 |
| m4 | <a href="https://github.com/tjgiese/atizer/blob/master/python/atizer/m4/m4/ax.m4">ax.m4</a> | 47,828 |
| Extensible Stylesheet Language Transformations | <a href="https://github.com/cmps290t/cmps290t/blob/master/templates/green_ccd.xslt">green_ccd.xslt</a> | 37,247 |
| License | <a href="https://github.com/xnox/android/blob/master/copyright">copyright</a> | 37,205 |
| JavaServer Pages | <a href="https://github.com/julien1990/wtp-sourceediting/blob/master/tests/org.eclipse.jst.jsp.ui.tests.performance/data/1MB.jsp">1MB.jsp</a> | 36,007 |
| Document Type Definition | <a href="https://github.com/braunoeder/yesodcms/blob/master/dita/dtd/bookmap/dtd/bookmap.dtd">bookmap.dtd</a> | 32,815 |
| Fish | <a href="https://gitlab.com/mitzip/fish-config/blob/master/functions/Godsay.fish">Godsay.fish</a> | 31,112 |
| ClojureScript | <a href="https://github.com/timvisher/painful-clojurescript-compilation/blob/master/src/painful_clojurescript_compilation/core.cljs">core.cljs</a> | 31,013 |
| Robot Framework | <a href="https://github.com/tomasz-kucharski/robocode/blob/master/doc/src_OpenGL/robot">robot</a> | 30,460 |
| Processing | <a href="https://github.com/bombilee/nxv11/blob/master/LidarSpoofing/data.pde">data.pde</a> | 30,390 |
| Ruby HTML | <a href="https://github.com/vijedi/javascript_framework_test/blob/master/app/views/home/big_table.rhtml">big_table.rhtml</a> | 29,306 |
| ColdFusion | <a href="https://github.com/llimllib/personal_code/blob/master/web/bnia/data/spreadsheet2009Q1.cfm">spreadsheet2009Q1.cfm</a> | 27,974 |
| CMake | <a href="https://github.com/luisibanez/vista-debian-med-package/blob/master/CMake/ListOfVistARoutines.cmake">ListOfVistARoutines.cmake</a> | 27,550 |
| ATS | <a href="https://github.com/ashalkhakov/ats-postiats/blob/master/npm-utils/contrib/libats-/hwxi/Andes/TEST/test06.dats">test06.dats</a> | 24,350 |
| Nim | <a href="https://github.com/renox/nimrod/blob/master/lib/windows/windows.nim">windows.nim</a> | 23,949 |
| Vue | <a href="https://github.com/deepfire/tech/blob/master/Ogre.vue">Ogre.vue</a> | 22,916 |
| Razor | <a href="https://github.com/agglerithm/readathonentry/blob/master/src/ReadAThonEntry/Views/Home/validationerror.cshtml">validationerror.cshtml</a> | 22,832 |
| Spice Netlist | <a href="https://github.com/danielsig727/dcs_labs/blob/master/DCS_FinalProject/input6.ckt">input6.ckt</a> | 22,454 |
| Isabelle | <a href="https://github.com/jshs/scyther-proof/blob/master/experiments/effect_of_reuse_and_minimization/output_FN/WooLam_cert_auto.thy">WooLam_cert_auto.thy</a> | 22,312 |
| XAML | <a href="https://github.com/tomba/dwarrowdelf/blob/master/Tests/WPFMapControlTest/SymbolDrawings.xaml">SymbolDrawings.xaml</a> | 20,764 |
| Opalang | <a href="https://github.com/kevtron/uniformsphere/blob/master/sample_data/p4000_g+5.0_m0.0_t00_st_z+0.00_a+0.00_c+0.00_n+0.00_o+0.00_r+0.00_s+0.00.opa">p4000_g+5.0_m0.0_t00_st_z+0.00_a+0.00_c+0.00_n+0.00_o+0.00_r+0.00_s+0.00.opa</a> | 20,168 |
| TOML | <a href="https://github.com/karupanerura/toml-parser/blob/master/xt/toml/too_large.toml">too_large.toml</a> | 20,000 |
| Madlang | <a href="https://github.com/cesarotti/dark-photons/blob/master/madgraph/madgraph_binaries/vendor/StdHEP/example/evgg.mad">evgg.mad</a> | 19,416 |
| Stylus | <a href="https://bitbucket.com/johanneskoch/interaktionskoll-client/src/master/node_modules/stylus-brunch/node_modules/stylus/testing/test.styl">test.styl</a> | 19,127 |
| Go Template | <a href="https://github.com/jtimothyking/perl6-bench/blob/master/data/html-template.tmpl">html-template.tmpl</a> | 19,016 |
| AutoHotKey | <a href="https://github.com/tinku99/ahk-hello-gl-ch2/blob/master/glext.ahk">glext.ahk</a> | 18,036 |
| ColdFusion CFScript | <a href="https://github.com/mfgglobalsolutions/collectmed/blob/master/collectmed1.0/CustomTags/com/common/db/IntakeHCPCIO.cfc">IntakeHCPCIO.cfc</a> | 17,606 |
| Zsh | <a href="https://github.com/voronenko/dotfiles/blob/master/completions/_oc.zsh">_oc.zsh</a> | 17,307 |
| Twig Template | <a href="https://github.com/terokaisti/elfinderbundle/blob/master/src/AlphaLemon/ElFinderBundle/Resources/views/ElFinder/show.html.twig">show.html.twig</a> | 16,320 |
| ABAP | <a href="https://github.com/yetaai/sap/blob/master/ZIM2Reports/ZRIM01F01.abap">ZRIM01F01.abap</a> | 16,029 |
| Elm | <a href="https://github.com/frankyn/csgraphics-fall13/blob/master/project1/data/viewpoint/57chevy.elm">57chevy.elm</a> | 14,968 |
| Kotlin | <a href="https://github.com/jetbrains/kotlin/blob/master/libraries/stdlib/common/src/generated/_Arrays.kt">_Arrays.kt</a> | 14,396 |
| Varnish Configuration | <a href="https://github.com/aureq/securityvcl/blob/master/vcl/breach/40_generic_attacks.vcl">40_generic_attacks.vcl</a> | 13,367 |
| Mustache | <a href="https://github.com/financial-times/o-table/blob/master/demos/src/huge.mustache">huge.mustache</a> | 13,313 |
| Alloy | <a href="https://github.com/epintos/fajita/blob/master/fajita/result/unroll_2/tacoOutput/output.als">output.als</a> | 12,168 |
| Device Tree | <a href="https://gitlab.com/mac/android_kernel_htc_flounder/blob/master/arch/arm64/boot/dts/tegra132-flounder-emc.dtsi">tegra132-flounder-emc.dtsi</a> | 11,893 |
| MQL4 | <a href="https://github.com/lvcster/java-examples/blob/master/PhD Appsolute System.mq4">PhD Appsolute System.mq4</a> | 11,280 |
| Jade | <a href="https://github.com/ernesto-licea/basictheme/blob/master/views/basic/images/icons/fugue.jade">fugue.jade</a> | 10,711 |
| Q# | <a href="https://github.com/afibanez/eneboo-modules/blob/master/direccion/analisis/scripts/in_navegador.qs">in_navegador.qs</a> | 10,025 |
| JSONL | <a href="https://github.com/aneeshmg/python/blob/master/NLP-TextEntailment/data/train.jsonl">train.jsonl</a> | 10,000 |
| Flow9 | <a href="https://github.com/hitchiker42/my-code/blob/master/cs758/asst-11/graph2.flow">graph2.flow</a> | 9,902 |
| Vala | <a href="https://github.com/stronnag/mwptools/blob/master/mwp/mwp.vala">mwp.vala</a> | 8,765 |
| Handlebars | <a href="https://github.com/jonschlinkert/liquid-to-handlebars/blob/master/test/expected/shopify-narrative/assets/theme.scss.hbs">theme.scss.hbs</a> | 8,259 |
| Crystal | <a href="https://github.com/000861/openfoam-21x/blob/master/tutorials/combustion/PDRFoam/flamePropagationWithObstacles/0/CR">CR</a> | 8,084 |
| C Shell | <a href="https://gitlab.com/bgcx262/zswi2010-svn-to-git/blob/master/trunk/data/plna.csh">plna.csh</a> | 8,000 |
| Hamlet | <a href="https://github.com/aliced3645/os/blob/master/weenix/user/hamlet">hamlet</a> | 7,882 |
| BuildStream | <a href="https://github.com/mrzork/proyectomaestria/blob/master/Final/Cites/Biometrics/biometrics.bst">biometrics.bst</a> | 7,746 |
| Mako | <a href="https://github.com/daniel2101/verificaciones/blob/master/report/verificaciones.mako">verificaciones.mako</a> | 7,306 |
| Agda | <a href="https://github.com/zsparks/pi-dual/blob/master/Univalence/Obsolete/Pifextra.agda">Pifextra.agda</a> | 6,483 |
| Thrift | <a href="https://github.com/cinchapi/concourse/blob/master/interface/concourse.thrift">concourse.thrift</a> | 6,471 |
| Fragment Shader File | <a href="https://github.com/jcamposr/wis20/blob/master/Simplex/ms812_bseqoslabel_l.fsh">ms812_bseqoslabel_l.fsh</a> | 6,269 |
| Cargo Lock | <a href="https://github.com/mozilla/servo/blob/master/Cargo.lock">Cargo.lock</a> | 6,202 |
| Xtend | <a href="https://github.com/arnobl/kompren/blob/master/kompren-examples/examples.umlFootprinting/src/main/java/umlfootprinter/UMLSlicerAspect.xtend">UMLSlicerAspect.xtend</a> | 5,936 |
| Arvo | <a href="https://github.com/scaleunlimited/cascadingavro/blob/master/scheme/src/test/resources/cascading/avro/test-extra-large.avsc">test-extra-large.avsc</a> | 5,378 |
| Scons | <a href="https://github.com/dxx-rebirth/dxx-rebirth/blob/master/SConstruct">SConstruct</a> | 5,272 |
| Closure Template | <a href="https://github.com/facebook/buck/blob/master/docs/files-and-dirs/buckconfig.soy">buckconfig.soy</a> | 5,189 |
| GN | <a href="https://github.com/v8/v8/blob/master/BUILD.gn">BUILD.gn</a> | 4,653 |
| Softbridge Basic | <a href="https://gitlab.com/bgcx261/znos-git/blob/master/build/usr/octopus/port/live/owptext.sbl">owptext.sbl</a> | 4,646 |
| PKGBUILD | <a href="https://bitbucket.com/axil42/aur-mirror/src/master/stepmania-extras/PKGBUILD">PKGBUILD</a> | 4,636 |
| Oz | <a href="https://github.com/aglie/mozart2/blob/master/lib/compiler/StaticAnalysis.oz">StaticAnalysis.oz</a> | 4,500 |
| Lucius | <a href="https://github.com/dsmatter/timetracker/blob/master/templates/bootstrap.lucius">bootstrap.lucius</a> | 3,992 |
| Ceylon | <a href="https://github.com/ceylon/ceylonast/blob/master/source/ceylon/ast/redhat/RedHatTransformer.ceylon">RedHatTransformer.ceylon</a> | 3,907 |
| Creole | <a href="https://github.com/bsmr-mariadb/mariadb-manager-monitor/blob/master/MariaDB_Manager_Monitors.creole">MariaDB_Manager_Monitors.creole</a> | 3,855 |
| Luna | <a href="https://github.com/luna/luna/blob/master/stdlib/Std/src/Base.luna">Base.luna</a> | 3,731 |
| Gradle | <a href="https://github.com/jasig/cas/blob/master/gradle/dependencies.gradle">dependencies.gradle</a> | 3,612 |
| MQL Header | <a href="https://github.com/ro31337/romanpushkin-dailygrid/blob/master/IncGUI.mqh">IncGUI.mqh</a> | 3,544 |
| Cabal | <a href="https://github.com/keqh-remote/cabals-mirror/blob/master/smartword.cabal">smartword.cabal</a> | 3,452 |
| Emacs Dev Env | <a href="https://github.com/a3090103838/emacs-24-mac/blob/master/info/ede">ede</a> | 3,400 |
| Meson | <a href="https://github.com/keruspe/systemd/blob/master/meson.build">meson.build</a> | 3,264 |
| nuspec | <a href="https://github.com/giggio/npm-nuget/blob/master/Npm.js.nuspec">Npm.js.nuspec</a> | 2,823 |
| Game Maker Project | <a href="https://github.com/kgs0142/ludum-dare/blob/master/LD40/LudumDare.yyp">LudumDare.yyp</a> | 2,679 |
| Julius | <a href="https://github.com/ahushh/monaba/blob/master/monaba/templates/default-layout.julius">default-layout.julius</a> | 2,454 |
| Idris | <a href="https://github.com/francks/ringidris/blob/master/Provers/ring_reduce.idr">ring_reduce.idr</a> | 2,434 |
| Alchemist | <a href="https://github.com/nim-hrkn/ecalj/blob/master/TestInstall/crn/out.lmf-dos.crn">out.lmf-dos.crn</a> | 2,388 |
| MQL5 | <a href="https://github.com/dennislwm/mlea/blob/master/MQL5/Experts/Commercial/DTS1-Build_814.1_B-test~.mq5">DTS1-Build_814.1_B-test~.mq5</a> | 2,210 |
| Android Interface Definition Language | <a href="https://github.com/android/platform_frameworks_base/blob/master/telephony/java/com/android/internal/telephony/ITelephony.aidl">ITelephony.aidl</a> | 2,005 |
| Vertex Shader File | <a href="https://github.com/bubbasacs/finalproj/blob/master/src/materialsystem/stdshaders/sdk_macros.vsh">sdk_macros.vsh</a> | 1,922 |
| Lean | <a href="https://github.com/leanprover/lean/blob/master/library/init/meta/interactive.lean">interactive.lean</a> | 1,664 |
| Jenkins Buildfile | <a href="https://github.com/elektrainitiative/libelektra/blob/master/scripts/jenkins/Jenkinsfile">Jenkinsfile</a> | 1,559 |
| FIDL | <a href="https://github.com/otcshare/automotive-message-broker/blob/master/docs/amb.in.fidl">amb.in.fidl</a> | 1,502 |
| Pony | <a href="https://bitbucket.com/thecomet/ponycraft-prototype/src/master/scripts/maps/demomap/scenery.pony">scenery.pony</a> | 1,497 |
| PureScript | <a href="https://github.com/maxnordlund/purescript/blob/master/prelude/prelude.purs">prelude.purs</a> | 1,225 |
| TaskPaper | <a href="https://github.com/jdzak/ncs_navigator_core/blob/master/audit/task-3275.taskpaper">task-3275.taskpaper</a> | 1,196 |
| Dockerfile | <a href="https://github.com/iplantcollaborativeopensource/discoveryenvironmentbackend/blob/master/docker/backwards-compat/Dockerfile">Dockerfile</a> | 1,187 |
| Janet | <a href="https://github.com/bianary/moosehead/blob/master/player/Janet">Janet</a> | 1,158 |
| Futhark | <a href="https://github.com/hiperfit/futhark/blob/master/futlib/math.fut">math.fut</a> | 990 |
| Zig | <a href="https://github.com/aurametrix/aurametrixhubio/blob/master/Games/Tetris-zig/src/main.zig">main.zig</a> | 903 |
| XCode Config | <a href="https://github.com/krzyzanowskim/cryptoswift/blob/master/config/Project-Shared.xcconfig">Project-Shared.xcconfig</a> | 522 |
| JAI | <a href="https://github.com/andy-hay/lightzone/blob/master/lightcrafts/resources/com/lightcrafts/mediax/jai/LCregistryFile.jai">LCregistryFile.jai</a> | 489 |
| QCL | <a href="https://bitbucket.com/gltronred/quipper-cabal/src/master/Programs/QCLParser/bwt.qcl">bwt.qcl</a> | 447 |
| Ur/Web Project | <a href="https://github.com/huluwa/bazqux-urweb/blob/master/reader.urp">reader.urp</a> | 346 |
| Cassius | <a href="https://github.com/nubis/haskellers/blob/master/cassius/default-layout.cassius">default-layout.cassius</a> | 313 |
| Docker ignore | <a href="https://github.com/ahbeng/nusmods/blob/master/.dockerignore">.dockerignore</a> | 311 |
| Dhall | <a href="https://github.com/github/linguist/blob/master/samples/Dhall/largeExpressionA.dhall">largeExpressionA.dhall</a> | 254 |
| ignore | <a href="https://github.com/9renpoto/dotfiles/blob/master/.ignore">.ignore</a> | 192 |
| Bitbucket Pipeline | <a href="https://github.com/ghb24/neci_stable/blob/master/bitbucket-pipelines.yml">bitbucket-pipelines.yml</a> | 181 |
| Just | <a href="https://github.com/sporto/kic/blob/master/api/graphql/Justfile">Justfile</a> | 95 |
| Verilog Args File | <a href="https://github.com/doswellf/combinator-uvm/blob/master/uvm_ref/1.2/uvm_ref_flow_1.2/designs/socv/rtl/rtl_lpw/opencores/or1200.irunargs">or1200.irunargs</a> | 60 |
| Polly | <a href="https://github.com/hsoft/aurdiff/blob/master/aur/polly">polly</a> | 26 |

### Whats the most complex file in each language?

Once again these values are not directly comparable to each other, but it is interesting to see what is considered the most complex in each language.

Some of these files are absolute monsters. For example consider the most complex C++ file I found [COLLADASaxFWLColladaParserAutoGen15PrivateValidation.cpp](https://github.com/KhronosGroup/OpenCOLLADA/blob/master/COLLADASaxFrameworkLoader/src/generated15/COLLADASaxFWLColladaParserAutoGen15PrivateValidation.cpp) which is 28.3 MB of compiler hell (and thankfully appears to be generated).

**NB** Some of the links below MAY not translate 100% due to throwing away some information when I created the files. Most should work, but a few you may need to mangle the URL to resolve.

[skip table to next section](#whats-the-most-complex-file-weighted-against-lines)

<div class="table-5"></div>

| language | filename | complexity |
| -------- | -------- | ---------- |
| C++ | <a href="https://bitbucket.com/zhangjingguo/opencollada/src/master/COLLADASaxFrameworkLoader/src/generated15/COLLADASaxFWLColladaParserAutoGen15PrivateValidation.cpp">COLLADASaxFWLColladaParserAutoGen15PrivateValidation.cpp</a> | 682,001 |
| JavaScript | <a href="https://github.com/gmarty/smsstarec/blob/master/compiled/blocks.js">blocks.js</a> | 582,070 |
| C Header | <a href="https://github.com/bathtub/c---project-with-ruby-tests/blob/master/src/bigmofoheader.h">bigmofoheader.h</a> | 465,589 |
| C | <a href="https://github.com/alexrhein/typechef-linuxanalysis/blob/master/fmFormula.c">fmFormula.c</a> | 445,545 |
| Objective C | <a href="https://github.com/chinnabommu/computervisionwip/blob/master/imports/fast/fast-matlab-src-2.1/faster.m">faster.m</a> | 409,792 |
| SQL | <a href="https://github.com/iklementiev/smscenter/blob/master/dump20120515.sql">dump20120515.sql</a> | 181,146 |
| ASP.NET | <a href="https://github.com/openindiana/oi-userland/blob/master/components/developer/gcc-7/test/results-i386.master">results-i386.master</a> | 164,528 |
| Java | <a href="https://github.com/cinchapi/concourse/blob/master/concourse-driver-java/src/main/java/com/cinchapi/concourse/thrift/ConcourseService.java">ConcourseService.java</a> | 139,020 |
| TCL | <a href="https://github.com/mheinrichs/68030tk/blob/master/Logic/68030_TK.tcl">68030_TK.tcl</a> | 136,578 |
| C++ Header | <a href="https://github.com/abdollah110/tauhlt/blob/master/CalibCalorimetry/EcalTPGTools/test/TPG_hardcoded.hh">TPG_hardcoded.hh</a> | 129,465 |
| TypeScript Typings | <a href="https://github.com/teppeis/closure-librarydts/blob/master/all.d.ts">all.d.ts</a> | 127,785 |
| SVG | <a href="https://github.com/tarekauel/planspiel/blob/master/Class Diagram2.svg">Class Diagram2.svg</a> | 105,353 |
| Lua | <a href="https://github.com/ldeniau/mad/blob/master/src/blackBoxTest/luaFile1000kLines.lua">luaFile1000kLines.lua</a> | 102,960 |
| PHP | <a href="https://github.com/bit3archive/php-benchmark/blob/master/fopen.php">fopen.php</a> | 100,000 |
| Org | <a href="https://github.com/mquinson/smpi-modeling/blob/master/collectives/log/2015-02-25_idfreeze-2.org">2015-02-25_idfreeze-2.org</a> | 63,326 |
| Ruby | <a href="https://github.com/calebbarr/tent-of-meeting/blob/master/resources/all_search_helpers.rb">all_search_helpers.rb</a> | 60,375 |
| Scheme | <a href="https://github.com/shilrobot/shilscript_plus_plus/blob/master/scripts/test.ss">test.ss</a> | 50,000 |
| Stata | <a href="https://github.com/funzelknut/migrationsvariable/blob/master/Stata/DoFile/Logic/combinations/.31113.do">.31113.do</a> | 48,600 |
| Elixir | <a href="https://github.com/teamcohen/querendipity/blob/master/data/pmid.sgd.crawl.ex">pmid.sgd.crawl.ex</a> | 46,479 |
| Brainfuck | <a href="https://bitbucket.com/pholey/poll/src/master/Brainfuck/Poll.bf">Poll.bf</a> | 41,399 |
| Perl | <a href="https://github.com/aanavas/srproject/blob/master/t2p/r1d7.pl">r1d7.pl</a> | 41,128 |
| Go | <a href="https://github.com/tsileo/blobstash/blob/master/vendor/github.com/blevesearch/segment/segment_words_prod.go">segment_words_prod.go</a> | 34,715 |
| Python | <a href="https://github.com/wks/lrparsing3/blob/master/doc/examples/lrparsing-sqlite.py">lrparsing-sqlite.py</a> | 34,700 |
| Module-Definition | <a href="https://github.com/mihania/slovo/blob/master/src/WindowsPhone/Slovo.UI/Data/wordnet3_0.def">wordnet3_0.def</a> | 32,008 |
| Clojure | <a href="https://github.com/candera/cs-atom/blob/master/raw-data.clj">raw-data.clj</a> | 29,950 |
| C# | <a href="https://github.com/accord-net/framework/blob/master/Sources/Accord.Math/Matrix/Matrix.Product.Generated.cs">Matrix.Product.Generated.cs</a> | 29,675 |
| D | <a href="https://github.com/cybershadow/seatd/blob/master/src/seatd/parser.d">parser.d</a> | 27,249 |
| FORTRAN Modern | <a href="https://github.com/rfitzp/tomuhawc/blob/master/chease_src/euitm_routines_407c.f90">euitm_routines_407c.f90</a> | 27,161 |
| Puppet | <a href="https://github.com/joliebig/crefactor-sqliteevaluation/blob/master/sqlite3.c.pp">sqlite3.c.pp</a> | 25,753 |
| SystemVerilog | <a href="https://github.com/rajdeep87/verilog-c/blob/master/safe/hwmcc15/6s131/6s131.sv">6s131.sv</a> | 24,300 |
| Autoconf | <a href="https://github.com/lronaldo/cpctelera/blob/master/cpctelera/tools/sdcc-3.6.8-r9946/src/device/lib/pic16/libio/Makefile.in">Makefile.in</a> | 23,183 |
| Specman e | <a href="https://github.com/alopez/en600468/blob/master/aligner/data/hansards.e">hansards.e</a> | 20,893 |
| Smarty Template | <a href="https://github.com/jouvin/pan/blob/master/panc/tests/Performance/tests/include/test-include-09.tpl">test-include-09.tpl</a> | 20,000 |
| TypeScript | <a href="UNKNOWN/blob/master/lib/compiler/parsers/JSONiqParser.ts">JSONiqParser.ts</a> | 18,162 |
| V | <a href="https://github.com/amartya00/verilog/blob/master/gplgpu/gplgpu/hdl/sim_lib/altera_mf.v">altera_mf.v</a> | 13,584 |
| F* | <a href="https://github.com/mmjb/kittel-koat/blob/master/koat-evaluation/examples/T2/slayer-3.fst">slayer-3.fst</a> | 13,428 |
| TeX | <a href="https://bitbucket.com/kommusoft/publications/src/master/urbandictionaryprintable/definitions.tex">definitions.tex</a> | 13,342 |
| Swift | <a href="https://github.com/alexeyxo/protobuf-swift/blob/master/plugin/Tests/pbTests/Google.Protobuf.UnittestEnormousDescriptor.proto.swift">Google.Protobuf.UnittestEnormousDescriptor.proto.swift</a> | 13,017 |
| Assembly | <a href="https://bitbucket.com/bminor/binutils-gdb/src/master/gas/testsuite/gas/tic54x/all-opcodes.s">all-opcodes.s</a> | 12,800 |
| Bazel | <a href="https://github.com/nonas/debian-clang/blob/master/tests/build_timeout/buildlogs/run1/firebird2.5_2.5.2.26540.ds4-10_amd64-20140427-2159.build">firebird2.5_2.5.2.26540.ds4-10_amd64-20140427-2159.build</a> | 12,149 |
| FORTRAN Legacy | <a href="https://github.com/bakhtatou/ecalj/blob/master/lm7K/lm67src/lm67.F">lm67.F</a> | 11,837 |
| R | <a href="https://github.com/nicebread/wrs/blob/master/pkg/R/Rallfun-v36.R">Rallfun-v36.R</a> | 11,287 |
| ActionScript | <a href="https://github.com/3bu1/crossbridge/blob/master/avmplus/test/acceptance/as3/Definitions/FunctionAccessors/AccessorSpray.as">AccessorSpray.as</a> | 10,804 |
| Haskell | <a href="https://bitbucket.com/plooney/lamdicom/src/master/Tags.hs">Tags.hs</a> | 10,444 |
| Prolog | <a href="https://github.com/nhuntwalker/mystuff/blob/master/books/books_save.p">books_save.p</a> | 10,243 |
| Dart | <a href="https://github.com/financecoding/dartlr/blob/master/tests/out/DartParser.dart">DartParser.dart</a> | 9,606 |
| VHDL | <a href="https://github.com/arunkumarcea/ahir/blob/master/vhdl/unisims/unisim_VITAL.vhd">unisim_VITAL.vhd</a> | 9,590 |
| Batch | <a href="https://github.com/alepharchives/arabica/blob/master/tests/XSLT/testsuite/test.bat">test.bat</a> | 9,424 |
| Boo | <a href="https://github.com/nddrylliog/ooc-legacy/blob/master/legacy-doc/compman.tex">compman.tex</a> | 9,280 |
| Coq | <a href="https://github.com/juliankemmerer/drexel-ecec575/blob/master/Encounter/NangateOpenCellLibrary/Front_End/Verilog/NangateOpenCellLibrary.v">NangateOpenCellLibrary.v</a> | 8,988 |
| Shell | <a href="https://github.com/existme/myvimconfig/blob/master/zsh/completion/i3_completion.sh">i3_completion.sh</a> | 8,669 |
| Kotlin | <a href="https://github.com/jetbrains/kotlin/blob/master/compiler/tests-spec/testData/diagnostics/notLinked/dfa/pos/1.kt">1.kt</a> | 7,388 |
| JSX | <a href="https://github.com/shibukawa/typescript-parserjsx/blob/master/src/typescript-parser.jsx">typescript-parser.jsx</a> | 7,123 |
| Makefile | <a href="https://github.com/kharmajbird/jayos/blob/master/jlfs/Makefile">Makefile</a> | 6,642 |
| Emacs Lisp | <a href="https://github.com/bbohrer/emacs/blob/master/correct/bible.el">bible.el</a> | 6,345 |
| Objective C++ | <a href="https://github.com/metamath/setmm/blob/master/set.mm">set.mm</a> | 6,285 |
| OCaml | <a href="https://github.com/nrnrnr/qc--/blob/master/gen/sparcrec.ml">sparcrec.ml</a> | 6,285 |
| Expect | <a href="https://bitbucket.com/chameleonos/android_external_valgrind/src/master/main/none/tests/s390x/condloadstore.stdout.exp">condloadstore.stdout.exp</a> | 6,144 |
| SAS | <a href="https://github.com/gracieha/fixit/blob/master/import_REDCap.sas">import_REDCap.sas</a> | 5,783 |
| Julia | <a href="https://github.com/open-data/ckan-datatools/blob/master/data/pilot-2013-05-14.jl">pilot-2013-05-14.jl</a> | 5,599 |
| Cython | <a href="https://github.com/facebook/fbthrift/blob/master/thrift/compiler/test/fixtures/mcpp2-compare/gen-py3/module/types.pyx">types.pyx</a> | 5,278 |
| Modula3 | <a href="https://github.com/damsl/k3-mosaic/blob/master/tests/m3/tpch22.m3">tpch22.m3</a> | 5,182 |
| Haxe | <a href="https://github.com/saumya/nuggeta/blob/master/src-nuggeta/com/nuggeta/temp/k/l/m/T1231.hx">T1231.hx</a> | 5,110 |
| Visual Basic for Applications | <a href="https://gitlab.com/celnet/celnet/blob/master/Bt-UAT/src/classes/Coverage.cls">Coverage.cls</a> | 5,029 |
| Lisp | <a href="https://github.com/ayoub-89/nltk_data/blob/master/corpora/lin_thesaurus/simN.lsp">simN.lsp</a> | 4,994 |
| Scala | <a href="https://github.com/byvoid/sugarcpp/blob/master/src/SugarCpp.Test/Performance/SpeedTest1MB.sc">SpeedTest1MB.sc</a> | 4,908 |
| Groovy | <a href="https://github.com/nagysz/zkui/blob/master/grails-app/taglib/org/grails/plugins/zkui/ZulTagLib.groovy">ZulTagLib.groovy</a> | 4,714 |
| Powershell | <a href="https://github.com/carneycalcutt/windowspowershell/blob/master/Modules/WPK/GeneratedControls/PresentationFramework.ps1">PresentationFramework.ps1</a> | 4,108 |
| Ada | <a href="https://github.com/grahamstark/tax_benefit_model_components/blob/master/src/uk/raw/bhps/bhps-print_full_version.adb">bhps-print_full_version.adb</a> | 3,961 |
| JavaServer Pages | <a href="https://github.com/builtlean/builtlean/blob/master/styles/sink_jq.jsp">sink_jq.jsp</a> | 3,850 |
| GN | <a href="https://github.com/reallyenglish/freebsd-ports/blob/master/www/chromium/files/patch-third_party__ffmpeg__ffmpeg_generated.gni">patch-third_party__ffmpeg__ffmpeg_generated.gni</a> | 3,742 |
| Basic | <a href="https://github.com/hpux735/spectrum-analyzer/blob/master/Spectrum Analyzer/MSA_version116_4q.bas">MSA_version116_4q.bas</a> | 3,502 |
| Pascal | <a href="https://github.com/katrid/python4delphi/blob/master/PythonVCL/Components/Sources/VCL/D4/Python_StdCtrls.pas">Python_StdCtrls.pas</a> | 3,399 |
| Standard ML (SML) | <a href="https://github.com/dsheets/hol/blob/master/examples/l3-machine-code/arm/model/arm.sml">arm.sml</a> | 3,375 |
| Erlang | <a href="https://github.com/h4xxel/prog2/blob/master/inl2/lipsum.hrl">lipsum.hrl</a> | 3,228 |
| ASP | <a href="https://github.com/cinsoft/my-library/blob/master/mylib.asp">mylib.asp</a> | 3,149 |
| CSS | <a href="https://github.com/kaosat-dev/polymer-threejs/blob/master/elements/three-viewer/vendor/three-viewer.css">three-viewer.css</a> | 3,071 |
| Unreal Script | <a href="https://github.com/svn2github/deus-ex-plus/blob/master/Classes/ScriptedPawn.uc">ScriptedPawn.uc</a> | 2,909 |
| CoffeeScript | <a href="https://github.com/uhyo/jinrou/blob/master/server/rpc/game/game.coffee">game.coffee</a> | 2,772 |
| AutoHotKey | <a href="https://github.com/billman87/fishlog/blob/master/fishlog5.93.ahk">fishlog5.93.ahk</a> | 2,764 |
| MQL4 | <a href="https://github.com/lvcster/java-examples/blob/master/PhD Appsolute System.mq4">PhD Appsolute System.mq4</a> | 2,738 |
| Processing | <a href="https://github.com/apcs-k/per10-zilbersher-reschke-lasercore/blob/master/Final/Final.pde">Final.pde</a> | 2,635 |
| Isabelle | <a href="https://github.com/shxycwxj/topos/blob/master/thy_lib/isabelle_afp/Collections/gen_algo/StdInst.thy">StdInst.thy</a> | 2,401 |
| Razor | <a href="https://github.com/nathantownsend/protodev/blob/master/DEQMYCOAL.web/Views/ePermitCompleteness/Checklist.cshtml">Checklist.cshtml</a> | 2,341 |
| Sass | <a href="https://github.com/aleph3d/wafer/blob/master/SOURCE/THIRDPARTY/Stackicons/scss/_multi-color-css-stackicons-social.scss">_multi-color-css-stackicons-social.scss</a> | 2,325 |
| Vala | <a href="https://github.com/gnome/vala/blob/master/codegen/valaccodebasemodule.vala">valaccodebasemodule.vala</a> | 2,100 |
| MSBuild | <a href="https://github.com/blackoutjack/jamweaver/blob/master/src/native/all.props">all.props</a> | 2,008 |
| Rust | <a href="https://github.com/blei/rust-dumb-gtk/blob/master/src/ffi.rs">ffi.rs</a> | 1,928 |
| QML | <a href="https://github.com/rakesh91/qt_mobile_apps/blob/master/DOTS/SmartDots-build-simulator/qml/Dots/Dots.qml">Dots.qml</a> | 1,875 |
| F# | <a href="https://github.com/jack-pappas/fsharp/blob/master/tests/fsharp/core/libtest/test.fsx">test.fsx</a> | 1,826 |
| Vim Script | <a href="https://github.com/anoxic/dotfiles/blob/master/vim/autoload/netrw.vim">netrw.vim</a> | 1,790 |
| Korn Shell | <a href="https://github.com/ggouaillardet/ompi-www/blob/master/community/lists/devel/attachments/20090108/42143e6f/attachment.ksh">attachment.ksh</a> | 1,773 |
| Vue | <a href="https://github.com/sebabelmar/sebabelmarhubio/blob/master/img/Seba Belmar - Software Engineer_files/vue">vue</a> | 1,738 |
| sed | <a href="https://bitbucket.com/foss4mv/scarletdme/src/master/GPL.BP/SED">SED</a> | 1,699 |
| GLSL | <a href="https://github.com/asirjoosingh/renorm_gamess/blob/master/comp">comp</a> | 1,699 |
| Nix | <a href="https://bitbucket.com/ifenglin/ideploy/src/master/Splunk_TA_nix/samples/auth.nix">auth.nix</a> | 1,615 |
| Mustache | <a href="https://github.com/browniefed/parsetemplateexample/blob/master/public/template.mustache">template.mustache</a> | 1,561 |
| Bitbake | <a href="https://github.com/rvclayton/bibtex/blob/master/my-2010.bb">my-2010.bb</a> | 1,549 |
| Ur/Web | <a href="https://github.com/deepuiitk/indian-parallel-corpora/blob/master/ur-en/votes.ur">votes.ur</a> | 1,515 |
| BASH | <a href="https://github.com/andrei-mart/postgres-xc/blob/master/contrib/pgxc_ctl/pgxc_ctl.bash">pgxc_ctl.bash</a> | 1,426 |
| MQL Header | <a href="https://github.com/ujfjhz/costbalancer/blob/master/Include/hanoverfunctions.mqh">hanoverfunctions.mqh</a> | 1,393 |
| Visual Basic | <a href="https://github.com/mateso/lgmd2ward/blob/master/lgmd2ward/LGMDdataDataSet.Designer.vb">LGMDdataDataSet.Designer.vb</a> | 1,369 |
| Q# | <a href="https://github.com/mariomop/abanq-ar/blob/master/modulos/facturacion/facturacion/scripts/flfacturac.qs">flfacturac.qs</a> | 1,359 |
| C Shell | <a href="https://bitbucket.com/dargueso/unsw-ccrc-wrf/src/master/WRFV3/tools/regtest_hwrf.csh">regtest_hwrf.csh</a> | 1,214 |
| MQL5 | <a href="https://github.com/dennislwm/mlea/blob/master/MQL5/Experts/Commercial/DTS1-Build_814.1_B-test~.mq5">DTS1-Build_814.1_B-test~.mq5</a> | 1,186 |
| Xtend | <a href="https://github.com/knisterpeter/djeypeg/blob/master/parser/src/main/java/de/matrixweb/djeypeg/internal/Parser.xtend">Parser.xtend</a> | 1,116 |
| Nim | <a href="https://github.com/comex/imaon/blob/master/disas.nim">disas.nim</a> | 1,098 |
| CMake | <a href="https://github.com/adc90/sandbox/blob/master/cmake/tutorial1/src/cmake/modules/MacroOutOfSourceBuild.cmake">MacroOutOfSourceBuild.cmake</a> | 1,069 |
| Protocol Buffers | <a href="https://github.com/areascout/vice-gles2/blob/master/configure.proto">configure.proto</a> | 997 |
| SKILL | <a href="https://github.com/kth-prosper/main/blob/master/verification/out/from_hol2/switch.il">switch.il</a> | 997 |
| COBOL | <a href="https://github.com/rflejeune/geekcode/blob/master/geekcode.cob">geekcode.cob</a> | 989 |
| Game Maker Language | <a href="https://github.com/bkiselka/hale/blob/master/doc/plugins/eu.esdihumboldt.hale.doc.user.examples.meridian2/hydro/hydroEx_River.gml">hydroEx_River.gml</a> | 982 |
| Gherkin Specification | <a href="https://github.com/catrobat/catroweb-symfony/blob/master/tests/behat/features/api/upload_remixed_program_again_complex.feature">upload_remixed_program_again_complex.feature</a> | 959 |
| Alloy | <a href="https://github.com/iyouboushi/mirc-battlearena/blob/master/battlearena/battleformulas.als">battleformulas.als</a> | 948 |
| Bosque | <a href="https://github.com/umeshsahoo/website_umesh/blob/master/servlet/oracle/product/10.2.0/server/RDBMS/ADMIN/recover.bsq">recover.bsq</a> | 924 |
| ColdFusion | <a href="https://bitbucket.com/busches/lucee/src/master/lucee-cfml/lucee-admin/jquery.js.cfm">jquery.js.cfm</a> | 920 |
| Stylus | <a href="https://gitlab.com/artisin/dreamdocs/blob/master/client/styles/imports/buttron.styl">buttron.styl</a> | 866 |
| ColdFusion CFScript | <a href="https://github.com/blueriver/muracms/blob/master/core/mura/client/api/json/v1/apiUtility.cfc">apiUtility.cfc</a> | 855 |
| Verilog | <a href="https://bitbucket.com/gdevic/a-z80/src/master/cpu/control/exec_matrix.vh">exec_matrix.vh</a> | 793 |
| Freemarker Template | <a href="https://github.com/jonesde/moqui/blob/master/runtime/template/screen-macro/DefaultScreenMacros.html.ftl">DefaultScreenMacros.html.ftl</a> | 771 |
| Crystal | <a href="https://github.com/mverzilli/crystal/blob/master/src/compiler/crystal/syntax/lexer.cr">lexer.cr</a> | 753 |
| Forth | <a href="https://github.com/mdko/cs478/blob/master/project-toolkits/toolkitc/bin/e4">e4</a> | 690 |
| Monkey C | <a href="https://github.com/nasahackto/mesh/blob/master/mesh-1.4/mesh/perl/mc">mc</a> | 672 |
| Rakefile | <a href="https://github.com/structuralartistry/wavelineup3/blob/master/lib/tasks/import.rake">import.rake</a> | 652 |
| Zsh | <a href="https://github.com/kino/mydotfile/blob/master/homefile/zshrc">zshrc</a> | 649 |
| Ruby HTML | <a href="https://github.com/mallikarjunrao/hmm_application/blob/master/cravecupcakes/trunk/app/views/ownify/sales/ext_report.rhtml">ext_report.rhtml</a> | 633 |
| Handlebars | <a href="https://github.com/sakai-mirror/roster2/blob/master/src/webapp/templates/templates.handlebars">templates.handlebars</a> | 557 |
| SRecode Template | <a href="https://github.com/samdutton/chromesearch/blob/master/tracks/Al3SEbeK61s.srt">Al3SEbeK61s.srt</a> | 535 |
| Scons | <a href="https://gitlab.com/ineris/amc/blob/master/share/SConstruct">SConstruct</a> | 522 |
| Agda | <a href="https://github.com/hott/hott-agda/blob/master/core/lib/cubical/Square.agda">Square.agda</a> | 491 |
| Ceylon | <a href="https://github.com/unratito/ceylonlanguage/blob/master/test/metamodel/runtime.ceylon">runtime.ceylon</a> | 467 |
| Julius | <a href="https://github.com/ahushh/monaba/blob/master/monaba/templates/default-layout.julius">default-layout.julius</a> | 436 |
| Wolfram | <a href="https://github.com/peeterjoot/mathematica/blob/master/phy487/qmSolidsPs8dContourPlot.nb">qmSolidsPs8dContourPlot.nb</a> | 417 |
| Cabal | <a href="https://github.com/simonmar/parconc-examples/blob/master/parconc-examples.cabal">parconc-examples.cabal</a> | 406 |
| Fragment Shader File | <a href="https://github.com/benraziel/flappybird-shader/blob/master/flappybird.fsh">flappybird.fsh</a> | 349 |
| ATS | <a href="https://github.com/alex-ren/atstools/blob/master/src/ats_staexp2_util1.dats">ats_staexp2_util1.dats</a> | 311 |
| Jinja | <a href="https://github.com/f500/ansible-php_cli/blob/master/templates/php.ini.j2">php.ini.j2</a> | 307 |
| Opalang | <a href="https://github.com/alexkit/opalang/blob/master/lib/stdlib/core/unicode.opa">unicode.opa</a> | 306 |
| Twig Template | <a href="https://github.com/babyonline/opencart/blob/master/upload/admin/view/template/catalog/product_form.twig">product_form.twig</a> | 296 |
| ClojureScript | <a href="https://github.com/clojure/clojurescript/blob/master/src/main/cljs/cljs/core.cljs">core.cljs</a> | 271 |
| Hamlet | <a href="https://github.com/pgarst/javabio/blob/master/Data/Text/hamlet">hamlet</a> | 270 |
| Oz | <a href="https://github.com/doublec/mozart/blob/master/share/lib/compiler/StaticAnalysis.oz">StaticAnalysis.oz</a> | 267 |
| Elm | <a href="https://github.com/cthree/dotfiles/blob/master/config/atom/packages/elmjutsu/elm/Indexer.elm">Indexer.elm</a> | 267 |
| Meson | <a href="https://github.com/ixit/mesa-3d/blob/master/meson.build">meson.build</a> | 248 |
| ABAP | <a href="https://github.com/yetaai/sap/blob/master/ZBCS01Reports/ZRFFORI99.abap">ZRFFORI99.abap</a> | 244 |
| Dockerfile | <a href="https://github.com/osgeo/gdal/blob/master/gdal/docker/alpine-normal/Dockerfile">Dockerfile</a> | 243 |
| Wren | <a href="https://github.com/munificent/wren/blob/master/src/module/repl.wren">repl.wren</a> | 242 |
| Fish | <a href="https://github.com/adampash/dotfiles/blob/master/config/fish/functions/fisher.fish">fisher.fish</a> | 217 |
| Emacs Dev Env | <a href="https://github.com/timvisher/emacsbak/blob/master/info/ede">ede</a> | 211 |
| GDScript | <a href="https://github.com/grandmasterhack/1gam/blob/master/2017/June/Godot Picking Sticks/addons/vnen.tiled_importer/tiled_map.gd">tiled_map.gd</a> | 195 |
| IDL | <a href="https://github.com/bkaradzic/bgfx/blob/master/scripts/bgfx.idl">bgfx.idl</a> | 187 |
| Jade | <a href="https://github.com/jaeh/printit/blob/master/views/printit/pages/docs.jade">docs.jade</a> | 181 |
| PureScript | <a href="https://github.com/purescript-contrib/purescript-lists/blob/master/test/Test/Data/List.purs">List.purs</a> | 180 |
| XAML | <a href="https://github.com/btsmarco/theagpeya/blob/master/CopticAgpeya/English/Midnight.xaml">Midnight.xaml</a> | 179 |
| Flow9 | <a href="https://github.com/samst0r/samst0rhubio/blob/master/node_modules/graphql-compose/mjs/TypeMapper.js.flow">TypeMapper.js.flow</a> | 173 |
| Idris | <a href="https://github.com/cheepnis/idris-dev/blob/master/libs/base/Language/Reflection/Utils.idr">Utils.idr</a> | 166 |
| PSL Assertion | <a href="https://github.com/endyson/sept-13-2011/blob/master/sim/psl/pre_dec.psl">pre_dec.psl</a> | 162 |
| Lean | <a href="https://github.com/avigad/libraries/blob/master/library/standard/kernel.lean">kernel.lean</a> | 161 |
| MUMPS | <a href="https://github.com/openzelda/content-package/blob/master/scripts/link.mps">link.mps</a> | 161 |
| Vertex Shader File | <a href="https://github.com/sharaugn/mc-reloaded-shaders/blob/master/base.vsh">base.vsh</a> | 152 |
| Go Template | <a href="https://github.com/aicp/frameworks_native/blob/master/vulkan/libvulkan/code-generator.tmpl">code-generator.tmpl</a> | 148 |
| Mako | <a href="https://github.com/epithumia/spline-pokedex/blob/master/splinext/pokedex/templates/pokedex/search/pokemon.mako">pokemon.mako</a> | 137 |
| Closure Template | <a href="https://github.com/gan/ganhubio/blob/master/blockly-games/genetics/template.soy">template.soy</a> | 121 |
| Zig | <a href="https://github.com/aurametrix/aurametrixhubio/blob/master/Games/Tetris-zig/src/main.zig">main.zig</a> | 115 |
| TOML | <a href="https://github.com/monome/teletype/blob/master/docs/ops/telex_o.toml">telex_o.toml</a> | 100 |
| Softbridge Basic | <a href="https://github.com/hardbol/spitbol/blob/master/gas/asm.sbl">asm.sbl</a> | 98 |
| QCL | <a href="https://bitbucket.com/gltronred/quipper-cabal/src/master/Programs/QCLParser/bwt.qcl">bwt.qcl</a> | 96 |
| Futhark | <a href="https://github.com/hiperfit/futhark/blob/master/futlib/math.fut">math.fut</a> | 86 |
| Pony | <a href="https://github.com/rockneurotiko/rockneurotikohubio/blob/master/RandomThings/Pony/json/jstypes.pony">jstypes.pony</a> | 70 |
| LOLCODE | <a href="https://bitbucket.com/animeshsinha/linguist-github/src/master/samples/LOLCODE/LOLTracer.lol">LOLTracer.lol</a> | 61 |
| Alchemist | <a href="https://github.com/boyter/scc/blob/master/alchemist.crn">alchemist.crn</a> | 55 |
| Madlang | <a href="https://github.com/aldomx/stepmania/blob/master/Docs/Copying.MAD">Copying.MAD</a> | 44 |
| LD Script | <a href="https://github.com/rockbox/rockbox/blob/master/apps/plugins/plugin.lds">plugin.lds</a> | 39 |
| Device Tree | <a href="https://github.com/wanghao-xznu/vte/blob/master/testcases/third_party_suite/dt/dt.d/dts">dts</a> | 22 |
| FIDL | <a href="https://github.com/bmwcarit/joynr/blob/master/basemodel/src/main/franca/joynr/GlobalCapabilitiesDirectory.fidl">GlobalCapabilitiesDirectory.fidl</a> | 19 |
| JAI | <a href="https://github.com/mihxil/mmbase/blob/master/documentation/releases/legal/LICENSE.jai">LICENSE.jai</a> | 18 |
| Just | <a href="https://github.com/opetushallitus/eperusteet/blob/master/Justfile">Justfile</a> | 7 |
| Android Interface Definition Language | <a href="https://github.com/zhaojl/demo/blob/master/android_base/aidl">aidl</a> | 3 |
| Ur/Web Project | <a href="https://bitbucket.com/rkeatin3/code-samples/src/master/C++(ROS)/ME530646/ur5/jointSpace.urp">jointSpace.urp</a> | 2 |
| Spice Netlist | <a href="https://github.com/acuoci/edcsmoke/blob/master/run/kineticMechanisms/thermodynamics/GRI30.CKT">GRI30.CKT</a> | 2 |

### Whats the most complex file weighted against lines?

This sounds good in practice, but in reality... anything minified or with no newlines skews the results making this one effectively pointless. As such I have not included this calculation. I have however created an issue inside `scc` to support detection of minified code so it can be removed from the calculation results https://github.com/boyter/scc/issues/91

It's probably possible to infer this using just the data at hand, but id like to make it a more robust check that anyone using `scc` can benefit from.

### Whats the most commented file in each language?

Whats the most commented file in each language? I have no idea what sort of information you can get out of this that might be useful but it is interesting to have a look.

**NB** Some of the links below MAY not translate 100% due to throwing away some information when I created the files. Most should work, but a few you may need to mangle the URL to resolve.

[skip table to next section](#how-many-pure-projects)

<div class="table-6"></div>

| language | filename | comment lines |
| -------- | -------- | ------------- |
| Prolog | <a href="https://github.com/ncvc/sentiment/blob/master/ts-with-score-multiplier.p">ts-with-score-multiplier.p</a> | 5,603,870 |
| C | <a href="https://github.com/aclements/thesis/blob/master/thesis/data/testgen.c">testgen.c</a> | 1,705,508 |
| Python | <a href="https://github.com/maxsong123/bearded-wight/blob/master/Untitled0.py">Untitled0.py</a> | 1,663,466 |
| JavaScript | <a href="https://github.com/meotimdihia/pythontest/blob/master/100MB.js">100MB.js</a> | 1,165,656 |
| SVG | <a href="https://bitbucket.com/nicholma/cs4021-advanced-architecture-misc/src/master/Interleaves/alt/p4-s3_I369600.svg">p4-s3_I369600.svg</a> | 1,107,955 |
| SQL | <a href="https://github.com/freeside/freeside/blob/master/FS-Test/share/test.sql">test.sql</a> | 858,993 |
| C Header | <a href="https://github.com/findingdestity/vuforiaaugmentedreality/blob/master/VuforiaAugmentedReality/VuforiaAugmentedReality/Models/head.h">head.h</a> | 686,587 |
| C++ | <a href="https://github.com/major-lab/mccore/blob/master/lib/ResidueTopology.cc">ResidueTopology.cc</a> | 663,024 |
| Autoconf | <a href="https://github.com/revivo/py_snippets/blob/master/data/square_detector_local.in">square_detector_local.in</a> | 625,464 |
| TypeScript | <a href="https://github.com/forivall/typescript/blob/master/tests/cases/fourslash/reallyLargeFile.ts">reallyLargeFile.ts</a> | 583,708 |
| LEX | <a href="https://github.com/furushchev/jsk_demos/blob/master/jsk_2015_06_hrp_drc/drc_task_common/euslisp/vehicle/polaris-xp900.l">polaris-xp900.l</a> | 457,288 |
| XML | <a href="https://github.com/cyberdrcarr/knowledgemanagementframework/blob/master/QA/CommonDataLayer/Test1-CDL-soapui-project.xml">Test1-CDL-soapui-project.xml</a> | 411,321 |
| HTML | <a href="https://github.com/alvarobp/donde_estudio/blob/master/test/fixtures/todos_centros.html">todos_centros.html</a> | 366,776 |
| Pascal | <a href="https://github.com/grahamegrieve/fhirserver/blob/master/library/r4/FHIR.R4.Resources.pas">FHIR.R4.Resources.pas</a> | 363,289 |
| SystemVerilog | <a href="https://github.com/t-crest/bluetree/blob/master/mkToplevelBT64.v">mkToplevelBT64.v</a> | 338,042 |
| PHP | <a href="https://github.com/easyvyc/easywebmanager/blob/master/site/classes/lib/cool-php-captcha/resources/words/lt.php">lt.php</a> | 295,054 |
| TypeScript Typings | <a href="https://github.com/vansimke/dojotypedescriptiongenerator/blob/master/DojoTypeDescriptor/Scripts/typings/dojox.d.ts">dojox.d.ts</a> | 291,002 |
| Verilog | <a href="https://github.com/krios262/551project/blob/master/synthVdotP/CVP14_synth.vg">CVP14_synth.vg</a> | 264,649 |
| Lua | <a href="https://github.com/kinshi/tarkin_scripts/blob/master/scripts/object/mobile/objects.lua">objects.lua</a> | 205,006 |
| V | <a href="https://github.com/bart-group/bart/blob/master/tests/ARTIETests/testfiles/TestDataset01-functional.v">TestDataset01-functional.v</a> | 201,973 |
| Java | <a href="https://github.com/christianharrington/mdd/blob/master/IfcXML/src/org/tech/iai/ifc/xml/ifc/_2x3/final_/FinalPackage.java">FinalPackage.java</a> | 198,035 |
| C++ Header | <a href="https://github.com/pykoder/redemption/blob/master/tests/includes/fixtures/test_cliprdr_channel_xfreerdp_full_authorisation.hpp">test_cliprdr_channel_xfreerdp_full_authorisation.hpp</a> | 196,958 |
| Shell | <a href="https://github.com/jcs/openbsd-commitid/blob/master/out/add_commitids_to_src.sh">add_commitids_to_src.sh</a> | 179,223 |
| C# | <a href="https://github.com/arventwei/wcell/blob/master/Core/WCell.Constants/Items/ItemId.cs">ItemId.cs</a> | 171,944 |
| FORTRAN Modern | <a href="https://github.com/johannesgerer/jburkardt-f/blob/master/slatec/slatec.f90">slatec.f90</a> | 169,817 |
| Assembly | <a href="https://github.com/dpeddi/ws-28xx-hacking/blob/master/HeavyWeather.asm">HeavyWeather.asm</a> | 169,645 |
| Module-Definition | <a href="https://github.com/msherman13/e6321_top_half/blob/master/apr/top_level.final.def">top_level.final.def</a> | 139,150 |
| FORTRAN Legacy | <a href="https://github.com/wch/r-source/blob/master/src/modules/lapack/dlapack.f">dlapack.f</a> | 110,640 |
| VHDL | <a href="https://github.com/mbarga/archive-cpu/blob/master/mapped/cpuTest.vhd">cpuTest.vhd</a> | 107,882 |
| Groovy | <a href="https://bitbucket.com/nfredricks/vim-files/src/master/tags/groovy">groovy</a> | 98,985 |
| IDL | <a href="https://github.com/brownplt/strobe/blob/master/data/all-idls.idl">all-idls.idl</a> | 91,771 |
| Wolfram | <a href="https://github.com/gillespie/torsions/blob/master/LinKnots/K2KL.nb">K2KL.nb</a> | 90,224 |
| Go | <a href="https://github.com/tsileo/blobstash/blob/master/vendor/gopkg.in/src-d/enry.v1/data/frequencies.go">frequencies.go</a> | 89,661 |
| Scheme | <a href="https://github.com/mojmir-svoboda/dbgtoolkit/blob/master/3rd/s7/s7test.scm">s7test.scm</a> | 88,907 |
| D | <a href="https://github.com/khanhbui/jpf/blob/master/old_projects/jpf-symbc/.hg/store/data/lib/coral.jar.d">coral.jar.d</a> | 80,674 |
| Coq | <a href="https://github.com/amigabill/minimig-de1/blob/master/lib/altera/cycloneiv_hssi_atoms.v">cycloneiv_hssi_atoms.v</a> | 74,936 |
| Specman e | <a href="https://github.com/shihyu/mytool/blob/master/mybin/se/macros/sysobjs.e">sysobjs.e</a> | 65,146 |
| Puppet | <a href="https://github.com/joliebig/crefactor-sqliteevaluation/blob/master/sqlite3.c.pp">sqlite3.c.pp</a> | 63,656 |
| Wren | <a href="https://github.com/munificent/wren/blob/master/test/limit/many_globals.wren">many_globals.wren</a> | 61,388 |
| Boo | <a href="https://github.com/dt888/starlink/blob/master/applications/kappa/sun95.tex">sun95.tex</a> | 57,018 |
| Ruby | <a href="https://github.com/pburba/demo_app/blob/master/app/models/bigfile.rb">bigfile.rb</a> | 50,000 |
| Objective C | <a href="https://github.com/jmcarp/fmri-pipe/blob/master/joblog/job_sub011.m">job_sub011.m</a> | 44,788 |
| CSS | <a href="https://github.com/26mansi/raxa-jss/blob/master/src/resources/css/screener.css">screener.css</a> | 43,785 |
| Swig | <a href="https://bitbucket.com/abellnets/hrossparser/src/master/xml_files/CIDE.I">CIDE.I</a> | 37,235 |
| Fish | <a href="https://gitlab.com/mitzip/fish-config/blob/master/functions/Godsay.fish">Godsay.fish</a> | 31,103 |
| Sass | <a href="https://github.com/wangbiaouestc/clpeak/blob/master/OpsBench/ma-thesis/bench/results/instruction/sm30_kernels.sass">sm30_kernels.sass</a> | 30,306 |
| CoffeeScript | <a href="https://github.com/trshugu/nt/blob/master/tmp.coffee">tmp.coffee</a> | 29,088 |
| Erlang | <a href="https://github.com/nussiminen/git_nussia/blob/master/nci_ft_ricm_dul_SUITE.erl">nci_ft_ricm_dul_SUITE.erl</a> | 28,306 |
| Lisp | <a href="https://github.com/mbcode/kmb/blob/master/km_2-5-33.lisp">km_2-5-33.lisp</a> | 27,579 |
| YAML | <a href="https://github.com/mariouriarte/anbeed/blob/master/data/fixtures/ciudades.yml">ciudades.yml</a> | 27,168 |
| R | <a href="https://github.com/cran/phylosim/blob/master/R/PhyloSimSource.R">PhyloSimSource.R</a> | 26,023 |
| Scala | <a href="https://github.com/ajanker/typechef/blob/master/CTypeChecker/src/test/scala/de/fosd/typechef/typesystem/generated/GeneratedRedeclTests.scala">GeneratedRedeclTests.scala</a> | 24,647 |
| Emacs Lisp | <a href="https://gitlab.com/com-informatimago/emacs/blob/master/pjb-java.el">pjb-java.el</a> | 24,375 |
| Haskell | <a href="https://github.com/weltensegler/zeno/blob/master/src/Calculus/Dipole80.hs">Dipole80.hs</a> | 24,245 |
| ATS | <a href="https://github.com/ashalkhakov/ats-postiats/blob/master/npm-utils/contrib/libats-/hwxi/Andes/TEST/test06.dats">test06.dats</a> | 24,179 |
| m4 | <a href="https://github.com/tjgiese/atizer/blob/master/python/atizer/m4/m4/ax.m4">ax.m4</a> | 22,675 |
| ActionScript | <a href="https://github.com/oyhc/simonynx/blob/master/hero/src/cmodule/ALCSWF/__2E_str95.as">__2E_str95.as</a> | 21,173 |
| Objective C++ | <a href="https://github.com/antonbikineev/clang/blob/master/test/Analysis/edges-new.mm">edges-new.mm</a> | 20,789 |
| Visual Basic | <a href="https://github.com/celinak/directobservationtoolsforwindows/blob/master/MapWindow/Classes/Projections/clsProjections.vb">clsProjections.vb</a> | 20,641 |
| TCL | <a href="https://github.com/mheinrichs/68030tk/blob/master/Logic/68030_TK.tcl">68030_TK.tcl</a> | 20,616 |
| Nix | <a href="https://github.com/matthewbauer/matthewbauerhubio/blob/master/nix">nix</a> | 19,605 |
| Perl | <a href="https://github.com/imazen/repositext/blob/master/vendor/lf_aligner_3.12/scripts/LF_aligner_3.12_with_modules.pl">LF_aligner_3.12_with_modules.pl</a> | 18,013 |
| Ada | <a href="https://github.com/landgraf/matreshka/blob/master/source/amf/uml/amf-internals-tables-uml_metamodel-objects.adb">amf-internals-tables-uml_metamodel-objects.adb</a> | 14,535 |
| Batch | <a href="https://github.com/asfuyao/asfuyao/blob/master/Windows/微软产品激活/Microsoft Activation Script _0.6_中英双语/MAS_0.6_en.cmd">MAS_0.6_en.cmd</a> | 14,402 |
| OCaml | <a href="https://github.com/camlspotter/ocamlpro-ocaml-branch/blob/master/examples/00036_mandelbrot/code_new.ml">code_new.ml</a> | 13,648 |
| LaTeX | <a href="https://github.com/gnuplot/tex_demos/blob/master/demo/output/context/pm3dcolors.tex">pm3dcolors.tex</a> | 13,092 |
| Properties File | <a href="https://github.com/open-wide/owsi-nuxeo-translations-explorer/blob/master/l10n/5.6/messages_ar_SA.properties">messages_ar_SA.properties</a> | 13,074 |
| MSBuild | <a href="https://github.com/muraad/bc-csharp/blob/master/crypto/ncrypto.csproj">ncrypto.csproj</a> | 11,302 |
| ASP.NET | <a href="https://github.com/fran9rodriguez/socios/blob/master/SocIoS Front End/SociosFrontEnd/DesktopModules/EasyDNNGallery/GallerySettings.ascx">GallerySettings.ascx</a> | 10,969 |
| Powershell | <a href="https://github.com/uti7/foo/blob/master/mail_imap.ps1">mail_imap.ps1</a> | 10,798 |
| Standard ML (SML) | <a href="https://github.com/petersewell/netsem/blob/master/unmaintained/Net/TCP/Spec2/TCP1_hostLTSScript.sml">TCP1_hostLTSScript.sml</a> | 10,790 |
| Dart | <a href="https://github.com/nkratzke/nkratzkehubio/blob/master/assets/ss2018/webtech/games/team-4g/packages/$sdk/lib/html/dart2js/html_dart2js.dart">html_dart2js.dart</a> | 10,547 |
| AutoHotKey | <a href="https://github.com/dajunx/cplusplus/blob/master/ahk_script/others/studio.ahk">studio.ahk</a> | 10,391 |
| Expect | <a href="https://github.com/vicamo/b2g_mozilla-central/blob/master/cmd/macfe/projects/client/Navigator.exp">Navigator.exp</a> | 10,063 |
| Julia | <a href="https://github.com/stevengj/petscjl/blob/master/src/generated/PETScRealSingle.jl">PETScRealSingle.jl</a> | 9,417 |
| Makefile | <a href="https://github.com/objeck/objeck-lang/blob/master/misc/wxWidgets/stc/lexer/Makefile">Makefile</a> | 9,204 |
| Forth | <a href="https://github.com/mar-one/machine-trans-shared-task/blob/master/data/corpus/europarl.lowercased.fr">europarl.lowercased.fr</a> | 9,107 |
| ColdFusion | <a href="https://github.com/phillipsenn/matrix/blob/master/js/js.cfm">js.cfm</a> | 8,786 |
| TeX | <a href="https://github.com/delcmo/dissertation/blob/master/hyperref.sty">hyperref.sty</a> | 8,591 |
| Opalang | <a href="https://github.com/ajbetteridge/opalang/blob/master/lib/stdlib/core/i18n/i18n_language.opa">i18n_language.opa</a> | 7,860 |
| LESS | <a href="https://github.com/2jdesign/magento2/blob/master/lib/web/css/docs/source/_variables.less">_variables.less</a> | 7,394 |
| Swift | <a href="https://github.com/smart-on-fhir/swift-fhir/blob/master/Sources/Models/CodeSystems.swift">CodeSystems.swift</a> | 6,847 |
| Bazel | <a href="https://github.com/nonas/debian-clang/blob/master/tests/build_timeout/buildlogs/run1/gcc-mingw-w64_12_amd64-20140427-2100.build">gcc-mingw-w64_12_amd64-20140427-2100.build</a> | 6,429 |
| Kotlin | <a href="https://github.com/jetbrains/kotlin/blob/master/libraries/stdlib/common/src/generated/_Arrays.kt">_Arrays.kt</a> | 5,887 |
| SAS | <a href="https://github.com/ikonovalov/sas-bug/blob/master/sas-checkstyle/src/main/resources/202_002_Stream_DQ_DRVT.sas">202_002_Stream_DQ_DRVT.sas</a> | 5,597 |
| Haxe | <a href="https://github.com/waneck/hx-javastd/blob/master/src/com/sun/rowset/CachedRowSetImpl.hx">CachedRowSetImpl.hx</a> | 5,438 |
| Rust | <a href="https://github.com/autonome/gecko-dev/blob/master/third_party/rust/lalrpop-snap/src/parser/lrgrammar.rs">lrgrammar.rs</a> | 5,150 |
| Monkey C | <a href="https://github.com/nasahackto/mesh/blob/master/mesh-1.4/mesh/perl/mc">mc</a> | 5,044 |
| Cython | <a href="https://github.com/strawlab/python-pcl/blob/master/pcl/pcl_common_172.pxd">pcl_common_172.pxd</a> | 5,030 |
| Nim | <a href="https://github.com/comex/imaon/blob/master/disas.nim">disas.nim</a> | 4,547 |
| Game Maker Language | <a href="https://github.com/ecriss/gmspineapi/blob/master/gm_spineapi.gml">gm_spineapi.gml</a> | 4,345 |
| ABAP | <a href="https://github.com/yetaai/sap/blob/master/ZDFIReports/ZACO19U_SHOP_NEW_1.abap">ZACO19U_SHOP_NEW_1.abap</a> | 4,244 |
| XAML | <a href="https://github.com/pittruff/mystik/blob/master/MyStik/RaumPlan/Raumplan.xaml">Raumplan.xaml</a> | 4,193 |
| Razor | <a href="https://github.com/jonezy/epilogger/blob/master/Epilogger.Web/Views/Home/Privacy.cshtml">Privacy.cshtml</a> | 4,092 |
| Varnish Configuration | <a href="https://github.com/aureq/securityvcl/blob/master/vcl/breach/46_slr_et_rfi_attacks.vcl">46_slr_et_rfi_attacks.vcl</a> | 3,924 |
| Basic | <a href="https://github.com/hpux735/spectrum-analyzer/blob/master/Spectrum Analyzer/MSA_version116_4q.bas">MSA_version116_4q.bas</a> | 3,892 |
| Isabelle | <a href="https://github.com/tangentstorm/tangentlabs/blob/master/isar/Pick.thy">Pick.thy</a> | 3,690 |
| Protocol Buffers | <a href="https://github.com/nospamdan/frameworks_base/blob/master/proto/src/metrics_constants.proto">metrics_constants.proto</a> | 3,682 |
| BASH | <a href="https://github.com/c0moshack/c0moshack/blob/master/dotfiles/bashrc">bashrc</a> | 3,606 |
| Clojure | <a href="https://github.com/dedeibel/dilist/blob/master/experiment/all-playlists-output.clj">all-playlists-output.clj</a> | 3,440 |
| F# | <a href="https://github.com/chrisa23/fmat/blob/master/src/Fmat.Numerics/GenericMatrixDoc.fs">GenericMatrixDoc.fs</a> | 3,383 |
| Thrift | <a href="https://github.com/evernote/evernote-thrift/blob/master/src/NoteStore.thrift">NoteStore.thrift</a> | 3,377 |
| COBOL | <a href="https://github.com/therocket/mixanalytics/blob/master/lib/db2include/cobol_mf/db2ApiDf.cbl">db2ApiDf.cbl</a> | 3,319 |
| JavaServer Pages | <a href="https://github.com/builtlean/builtlean/blob/master/styles/sink_jq.jsp">sink_jq.jsp</a> | 3,204 |
| Modula3 | <a href="https://github.com/jeffmendoza/86duino/blob/master/build/linux/work/DJGPP/info/gdb.i3">gdb.i3</a> | 3,124 |
| Visual Basic for Applications | <a href="https://bitbucket.com/ckuyehar/openahlta/src/master/Source/Data Manager/Back End/HL7xmlBuilder.cls">HL7xmlBuilder.cls</a> | 2,987 |
| Oz | <a href="https://github.com/raphinou/oz-compiler/blob/master/perfs/timing.oz">timing.oz</a> | 2,946 |
| Closure Template | <a href="https://github.com/facebook/buck/blob/master/docs/files-and-dirs/buckconfig.soy">buckconfig.soy</a> | 2,915 |
| Agda | <a href="https://github.com/zsparks/pi-dual/blob/master/Univalence/Obsolete/Pifextra.agda">Pifextra.agda</a> | 2,892 |
| Stata | <a href="https://github.com/ibli/ibli_borena_hh_survey/blob/master/R2_2cleaningprocess.do">R2_2cleaningprocess.do</a> | 2,660 |
| ColdFusion CFScript | <a href="https://github.com/mfgglobalsolutions/collectmed/blob/master/collectmed1.0/CustomTags/com/common/Intake.cfc">Intake.cfc</a> | 2,578 |
| Luna | <a href="https://github.com/luna/luna/blob/master/stdlib/Std/src/Base.luna">Base.luna</a> | 2,542 |
| Unreal Script | <a href="https://github.com/arcaneflux/hostile-worlds/blob/master/src/Hostile Worlds/Development/Src/Engine/Classes/UIRoot.uc">UIRoot.uc</a> | 2,449 |
| CMake | <a href="https://github.com/ipfire/ipfire-2x/blob/master/config/rootfiles/oldcore/66/filelists/cmake">cmake</a> | 2,425 |
| Org | <a href="https://github.com/willijar/lens/blob/master/doc/lens-wsn.org">lens-wsn.org</a> | 2,417 |
| Flow9 | <a href="https://github.com/hitode909/higashi-dance-network/blob/master/flow-typed/jquery/index.js.flow">index.js.flow</a> | 2,361 |
| MQL Header | <a href="https://github.com/ro31337/romanpushkin-dailygrid/blob/master/IncGUI.mqh">IncGUI.mqh</a> | 2,352 |
| JSX | <a href="https://github.com/lezuse/photoshop-scripts/blob/master/default/ContactSheetII.jsx">ContactSheetII.jsx</a> | 2,243 |
| MQL4 | <a href="https://github.com/lvcster/java-examples/blob/master/PhD Appsolute System.mq4">PhD Appsolute System.mq4</a> | 2,061 |
| Ruby HTML | <a href="https://github.com/srvanderplas/dissertation/blob/master/Presentations/Final Oral/FinalOral-Old.Rhtml">FinalOral-Old.Rhtml</a> | 2,061 |
| GDScript | <a href="https://github.com/laurentbartholdi/fr/blob/master/gap/group.gd">group.gd</a> | 2,023 |
| Processing | <a href="https://github.com/alexisgrinbold/pjs-2d-game-engine/blob/master/mario/testcode.pde">testcode.pde</a> | 2,014 |
| PSL Assertion | <a href="https://github.com/seomoz/url-py/blob/master/url/psl/2016-08-16.psl">2016-08-16.psl</a> | 2,011 |
| ASP | <a href="https://github.com/zhou-hui/zblog/blob/master/Release/zb_system/FUNCTION/c_system_plugin.asp">c_system_plugin.asp</a> | 1,878 |
| AWK | <a href="https://github.com/azizyemloul/plover-france-dict/blob/master/Atelier/dic-generator.awk">dic-generator.awk</a> | 1,732 |
| Jinja | <a href="https://github.com/dfederlein/ansible-aiua/blob/master/playbooks/roles/php5/templates/etc/php5/fpm/php.ini.j2">php.ini.j2</a> | 1,668 |
| Zsh | <a href="https://github.com/saizai/dotfiles/blob/master/.zshrc">.zshrc</a> | 1,588 |
| Q# | <a href="https://github.com/siagal/eneboo-modules/blob/master/direccion/analisis/scripts/in_navegador.qs">in_navegador.qs</a> | 1,568 |
| sed | <a href="https://github.com/alexjordan/patmos-benchmarks/blob/master/MiBench/office/ghostscript/src/Makefile.sed">Makefile.sed</a> | 1,554 |
| Stylus | <a href="https://github.com/leiferikb/bitpop-private/blob/master/bitpop_specific/extensions/bittorrent_surf/app/popup/popup.styl">popup.styl</a> | 1,550 |
| Bitbake | <a href="https://github.com/sasha-tvo/beam-splitting/blob/master/Lib/Doxyfile.bb">Doxyfile.bb</a> | 1,533 |
| Rakefile | <a href="https://github.com/ccls/odms/blob/master/lib/tasks/samples.rake">samples.rake</a> | 1,509 |
| Gherkin Specification | <a href="https://bitbucket.com/hagashennaidu/warewolf-esb/src/master/Dev/Dev2.Activities.Specs/Composition/WorkflowExecution.feature">WorkflowExecution.feature</a> | 1,421 |
| Crystal | <a href="https://github.com/manastech/crystal/blob/master/src/string.cr">string.cr</a> | 1,412 |
| Android Interface Definition Language | <a href="https://github.com/android/platform_frameworks_base/blob/master/telephony/java/com/android/internal/telephony/ITelephony.aidl">ITelephony.aidl</a> | 1,410 |
| Xtend | <a href="https://github.com/skunkiferous/meta/blob/master/api/src/main/java/com/blockwithme/meta/Properties.xtend">Properties.xtend</a> | 1,363 |
| SKILL | <a href="https://github.com/saycv/saycv_cadence_skillmngt/blob/master/_src/destub/src/DT_destub.il">DT_destub.il</a> | 1,181 |
| Madlang | <a href="https://github.com/madrocker/kernels/blob/master/15March/.config.mad">.config.mad</a> | 1,137 |
| Spice Netlist | <a href="https://github.com/gauxonz/prp_2013/blob/master/Lib/APEXLINEAR.ckt">APEXLINEAR.ckt</a> | 1,114 |
| QML | <a href="https://bitbucket.com/patrickfi/combinedindirectgestures/src/master/qml/MainFULL.qml">MainFULL.qml</a> | 1,078 |
| GLSL | <a href="https://github.com/carlos-felipe88/spacecraft/blob/master/data/shader/subPlanetNoise.frag">subPlanetNoise.frag</a> | 1,051 |
| Ur/Web | <a href="https://github.com/vagoff/the-matrix/blob/master/initial.ur">initial.ur</a> | 1,018 |
| Alloy | <a href="https://github.com/davletd/productlinetesting/blob/master/ProductLineTesting/AlloyModels/TransactionFeatureFinal.als">TransactionFeatureFinal.als</a> | 1,012 |
| Vala | <a href="https://github.com/kamilprusko/puzzle/blob/master/src/puzzle-piece.vala">puzzle-piece.vala</a> | 968 |
| Smarty Template | <a href="https://github.com/poganini/ee/blob/master/themes/Ensau.tpl">Ensau.tpl</a> | 965 |
| Mako | <a href="https://github.com/andrewsallans/centerforopenscienceorg/blob/master/www/jobs.mako">jobs.mako</a> | 950 |
| TOML | <a href="https://github.com/chuqingq/codeeveryday/blob/master/golang/20170420_go_reverseproxy_traefik/traefik.toml">traefik.toml</a> | 938 |
| gitignore | <a href="https://github.com/michalsc/aros/blob/master/.gitignore">.gitignore</a> | 880 |
| Elixir | <a href="https://github.com/alexbaranosky/elixir/blob/master/lib/elixir/macros.ex">macros.ex</a> | 832 |
| GN | <a href="https://github.com/autonome/gecko-dev/blob/master/media/webrtc/trunk/build/config/android/rules.gni">rules.gni</a> | 827 |
| Korn Shell | <a href="https://github.com/filipinotech/illumos-gate/blob/master/usr/src/lib/brand/lx/zone/lx_distro_install.ksh">lx_distro_install.ksh</a> | 807 |
| LD Script | <a href="https://bitbucket.com/imoseyon/d2vzw-alpha/src/master/arch/arm/kernel/vmlinux.lds">vmlinux.lds</a> | 727 |
| Scons | <a href="https://github.com/ajdavis/mongo/blob/master/SConstruct">SConstruct</a> | 716 |
| Handlebars | <a href="https://github.com/palanieswaran/cs247projectwebsite/blob/master/views/Consent-Form.handlebars">Consent-Form.handlebars</a> | 714 |
| Device Tree | <a href="https://github.com/coreboot/coreboot/blob/master/src/mainboard/cavium/cn8100_sff_evb/ddr4-common.dtsi">ddr4-common.dtsi</a> | 695 |
| FIDL | <a href="https://github.com/otcshare/automotive-message-broker/blob/master/docs/amb.in.fidl">amb.in.fidl</a> | 686 |
| Julius | <a href="https://github.com/sseefried/play-space-online/blob/master/julius/glMatrix.julius">glMatrix.julius</a> | 686 |
| C Shell | <a href="https://github.com/pmaksim1/bin/blob/master/setup_grid.csh">setup_grid.csh</a> | 645 |
| Lean | <a href="https://bitbucket.com/leanprover/lean/src/master/library/data/list/perm.lean">perm.lean</a> | 642 |
| Idris | <a href="https://github.com/eckart/idris-tutorial/blob/master/Overview.idr">Overview.idr</a> | 637 |
| PureScript | <a href="https://github.com/purescript/purescript-arrays/blob/master/src/Data/Array.purs">Array.purs</a> | 631 |
| Freemarker Template | <a href="https://github.com/oscar810429/painiu-project/blob/master/web/WEB-INF/templates/search/result_softwares.ftl">result_softwares.ftl</a> | 573 |
| ClojureScript | <a href="https://github.com/dwaynekj/lt-cljs-tutorial/blob/master/lt-cljs-tutorial.cljs">lt-cljs-tutorial.cljs</a> | 518 |
| Fragment Shader File | <a href="https://github.com/imclab/aluminum/blob/master/osx/examples/nautilus/resources/bulb.fsh">bulb.fsh</a> | 464 |
| Elm | <a href="https://github.com/catherinemoresco/catherinemorescohubio/blob/master/static/fourier-elm/elm-stuff/packages/evancz/elm-html/4.0.2/src/Html/Attributes.elm">Attributes.elm</a> | 434 |
| Jade | <a href="https://github.com/benbaker/zooid/blob/master/zooid_web/views/control/index.jade">index.jade</a> | 432 |
| Vue | <a href="https://github.com/iview/iview/blob/master/examples/routers/form.vue">form.vue</a> | 418 |
| Gradle | <a href="https://github.com/wso2/emm-agent-android/blob/master/client/client/build.gradle">build.gradle</a> | 416 |
| Lucius | <a href="https://github.com/dsmatter/timetracker/blob/master/templates/bootstrap.lucius">bootstrap.lucius</a> | 404 |
| Go Template | <a href="https://github.com/disposaboy/margo/blob/master/vendor/github.com/ugorji/go/codec/fast-path.go.tmpl">fast-path.go.tmpl</a> | 400 |
| Meson | <a href="https://github.com/tieto/pidgin/blob/master/meson.build">meson.build</a> | 306 |
| F* | <a href="https://github.com/fstarlang/fstar/blob/master/examples/low-level/crypto/Crypto.Symmetric.Poly1305.Bignum.Lemmas.Part1.fst">Crypto.Symmetric.Poly1305.Bignum.Lemmas.Part1.fst</a> | 289 |
| Ceylon | <a href="https://github.com/ceylon/ceylon-ide-intellij/blob/master/source/org/eclipse/ceylon/ide/intellij/psi/IdeaCeylonParser.ceylon">IdeaCeylonParser.ceylon</a> | 286 |
| MQL5 | <a href="https://github.com/dennislwm/mlea/blob/master/MQL5/Experts/Zigzag/ZigzagPattern_oldest.mq5">ZigzagPattern_oldest.mq5</a> | 282 |
| XCode Config | <a href="https://github.com/krzyzanowskim/cryptoswift/blob/master/config/Project-Shared.xcconfig">Project-Shared.xcconfig</a> | 265 |
| Futhark | <a href="https://github.com/hiperfit/futhark/blob/master/tests/blackscholes.fut">blackscholes.fut</a> | 257 |
| Pony | <a href="https://bitbucket.com/thecomet/ponycraft-prototype/src/master/scripts/maps/demomap/scenery.pony">scenery.pony</a> | 252 |
| Vertex Shader File | <a href="https://bitbucket.com/arif_uap/flipmymob-social-madness-iphone/src/master/FlipMobiPhone/cocos3d/GLSL/CC3TexturableRigidBones.vsh">CC3TexturableRigidBones.vsh</a> | 205 |
| Softbridge Basic | <a href="https://github.com/snowballstem/snowball/blob/master/algorithms/greek.sbl">greek.sbl</a> | 192 |
| Cabal | <a href="https://github.com/tener/deeplearning-thesis/blob/master/deeplearning.cabal">deeplearning.cabal</a> | 180 |
| nuspec | <a href="https://github.com/prashantvc/xamarinauth/blob/master/nuget/Xamarin.Auth.XamarinForms.nuspec">Xamarin.Auth.XamarinForms.nuspec</a> | 156 |
| Dockerfile | <a href="https://github.com/snewhouse/ngs/blob/master/containerized/pipeline/Dockerfile">Dockerfile</a> | 152 |
| Mustache | <a href="https://bitbucket.com/moodle/moodle/src/master/admin/tool/analytics/templates/models_list.mustache">models_list.mustache</a> | 141 |
| LOLCODE | <a href="https://bitbucket.com/animeshsinha/linguist-github/src/master/samples/LOLCODE/LOLTracer.lol">LOLTracer.lol</a> | 139 |
| BuildStream | <a href="https://github.com/npadmana/npadmana_config/blob/master/texmf/bibtex/bst/mybst/astrobib.bst">astrobib.bst</a> | 120 |
| Janet | <a href="https://github.com/bianary/moosehead/blob/master/player/Janet">Janet</a> | 101 |
| Cassius | <a href="https://github.com/trevorssmith1392/yuplanner/blob/master/templates/xweek.cassius">xweek.cassius</a> | 94 |
| Docker ignore | <a href="https://github.com/ahbeng/nusmods/blob/master/.dockerignore">.dockerignore</a> | 92 |
| Hamlet | <a href="https://github.com/raphaelj/getwebborg/blob/master/templates/upload.hamlet">upload.hamlet</a> | 90 |
| QCL | <a href="https://github.com/unixpickle/quantumprogramming/blob/master/mylib2/mod.qcl">mod.qcl</a> | 88 |
| Dhall | <a href="https://github.com/trskop/command-wrapper/blob/master/dhall/Exec/completion/nix.bash.dhall">nix.bash.dhall</a> | 86 |
| ignore | <a href="https://github.com/wy8023b/ka7ku/blob/master/.ignore">.ignore</a> | 60 |
| Just | <a href="https://github.com/sporto/kic/blob/master/api/graphql/Justfile">Justfile</a> | 46 |
| SRecode Template | <a href="https://bitbucket.com/andrewjchen/dotfiles/src/master/emacs.d/plugins/cedet/srecode/templates/srecode-test.srt">srecode-test.srt</a> | 35 |
| Bitbucket Pipeline | <a href="https://bitbucket.com/c2tarun/integrated-genome-browser/src/master/bitbucket-pipelines.yml">bitbucket-pipelines.yml</a> | 30 |
| Ur/Web Project | <a href="https://github.com/huluwa/bazqux-urweb/blob/master/reader.urp">reader.urp</a> | 22 |
| Alchemist | <a href="https://github.com/bakhtatou/ecalj/blob/master/lm7K/fp/test/ctrl.crn">ctrl.crn</a> | 16 |
| Zig | <a href="https://github.com/aurametrix/aurametrixhubio/blob/master/Games/Tetris-zig/src/main.zig">main.zig</a> | 12 |
| MUMPS | <a href="https://github.com/milindparikh/mps/blob/master/rel/files/mps">mps</a> | 11 |
| Bosque | <a href="https://github.com/boyter/scc/blob/master/bosque.bsq">bosque.bsq</a> | 8 |
| Report Definition Language | <a href="https://github.com/jayaddison/autochef/blob/master/idx/rdl/example.rdl">example.rdl</a> | 4 |
| Emacs Dev Env | <a href="https://bitbucket.com/hoangtu/my-emacs-prelude/src/master/cedet/lisp/cedet/Project.ede">Project.ede</a> | 3 |
| Cargo Lock | <a href="https://bitbucket.com/dorianpula/rookeries/src/master/Cargo.lock">Cargo.lock</a> | 2 |
| JAI | <a href="https://github.com/thekla/thekla_atlas/blob/master/src/thekla/thekla_atlas.jai">thekla_atlas.jai</a> | 1 |

### How many "pure" projects

Assuming you define pure to mean a project that has 1 language in it. Of course that would not be very interesting by itself, so lets see what the spread is. As it turns out most projects have fewer than 25 languages in them with most in the less than 10 bracket.

The peak in the below graph is for 4 languages.

Of course pure projects might only have one programming language, but have lots of supporting other formats such as markdown, json, yml, css, .gitignore which are picked up by scc. It's probably reasonable to assume that any project with less than 5 languages is "pure" (for some level of purity) and as it turns out is just over half the total data set. Of course your definition of purity might be different to mine so feel free to adjust to whatever number you like.

What suprises me is an odd bump around 34-35 languages. I have no reasonable explanation as to why this might be the case and it probably warrents some investigation.

![scc-data pure projects](/static/an-informal-survey/languagesPerProject.png#center)

The full list of results is included below.

[skip table to next section](#projects-with-typescript-but-not-javascript)

<div class="table-8"></div>

| language count | project count |
| -------------- | ------------- |
| 1 | 886,559 |
| 2 | 951,009 |
| 3 | 989,025 |
| 4 | 1,070,987 |
| 5 | 1,012,686 |
| 6 | 845,898 |
| 7 | 655,510 |
| 8 | 542,625 |
| 9 | 446,278 |
| 10 | 392,212 |
| 11 | 295,810 |
| 12 | 204,291 |
| 13 | 139,021 |
| 14 | 110,204 |
| 15 | 87,143 |
| 16 | 67,602 |
| 17 | 61,936 |
| 18 | 44,874 |
| 19 | 34,740 |
| 20 | 32,041 |
| 21 | 25,416 |
| 22 | 24,986 |
| 23 | 23,634 |
| 24 | 16,614 |
| 25 | 13,823 |
| 26 | 10,998 |
| 27 | 9,973 |
| 28 | 6,807 |
| 29 | 7,929 |
| 30 | 6,223 |
| 31 | 5,602 |
| 32 | 6,614 |
| 33 | 12,155 |
| 34 | 15,375 |
| 35 | 7,329 |
| 36 | 6,227 |
| 37 | 4,158 |
| 38 | 3,744 |
| 39 | 3,844 |
| 40 | 1,570 |
| 41 | 1,041 |
| 42 | 746 |
| 43 | 1,037 |
| 44 | 1,363 |
| 45 | 934 |
| 46 | 545 |
| 47 | 503 |
| 48 | 439 |
| 49 | 393 |
| 50 | 662 |
| 51 | 436 |
| 52 | 863 |
| 53 | 393 |
| 54 | 684 |
| 55 | 372 |
| 56 | 366 |
| 57 | 842 |
| 58 | 398 |
| 59 | 206 |
| 60 | 208 |
| 61 | 177 |
| 62 | 377 |
| 63 | 450 |
| 64 | 341 |
| 65 | 86 |
| 66 | 78 |
| 67 | 191 |
| 68 | 280 |
| 69 | 61 |
| 70 | 209 |
| 71 | 330 |
| 72 | 171 |
| 73 | 190 |
| 74 | 142 |
| 75 | 102 |
| 76 | 32 |
| 77 | 57 |
| 78 | 50 |
| 79 | 26 |
| 80 | 31 |
| 81 | 63 |
| 82 | 38 |
| 83 | 26 |
| 84 | 72 |
| 85 | 205 |
| 86 | 73 |
| 87 | 67 |
| 88 | 21 |
| 89 | 15 |
| 90 | 6 |
| 91 | 12 |
| 92 | 10 |
| 93 | 8 |
| 94 | 16 |
| 95 | 24 |
| 96 | 7 |
| 97 | 30 |
| 98 | 4 |
| 99 | 1 |
| 100 | 6 |
| 101 | 7 |
| 102 | 16 |
| 103 | 1 |
| 104 | 5 |
| 105 | 1 |
| 106 | 19 |
| 108 | 2 |
| 109 | 2 |
| 110 | 1 |
| 111 | 3 |
| 112 | 1 |
| 113 | 1 |
| 114 | 3 |
| 115 | 5 |
| 116 | 5 |
| 118 | 1 |
| 120 | 5 |
| 124 | 1 |
| 125 | 1 |
| 131 | 2 |
| 132 | 1 |
| 134 | 2 |
| 136 | 1 |
| 137 | 1 |
| 138 | 1 |
| 142 | 1 |
| 143 | 2 |
| 144 | 1 |
| 158 | 1 |
| 159 | 2 |

### Projects with TypeScript but not JavaScript

Ah the modern world of TypeScript. But for projects that are using TypeScipt how many are using TypeScript exclusively?

| pure TypeScript projects |
| --------------- |
| 27,026 projects |

Have to admit, I am a little surprised by that number. While I understand mixing JavaScript with TypeScript is fairly common I would have thought there would be more projects using the new hotness. This may however be mostly down to the projects I was able to pull though and I suspect a refreshed project list with newer projects would change this number drastically.

### Anyone using CoffeeScript and TypeScript?

| using TypeScript and CoffeeScript |
| --------------- |
| 7,849 projects |

I have a feeling some TypeScript developers are dry heaving at the very thought of this. If it is of any comfort I suspect most of these projects are things like `scc` which uses examples of all languages mixed together for testing purposes.


### What's the typical path length, broken up by language

Given that you can either dump all the files you need in a single directory, or span them out using file paths whats the typical path length and number of directories?

This is done by counting the number of path separators `/` for each file and its location and averaging it out. I didn't know what to expect here other that I would expect java to be close to the top as its file paths are usually quite deep.

[skip table to next section](#yaml-or-yml)

<div class="table-10"></div>

| language | average path length |
| -------- | ------------------- |
| ABAP | 4.406555175781266 |
| ASP | 6.372800350189209 |
| ASP.NET | 7.25 |
| ATS | 4.000007286696899 |
| AWK | 4.951896171638623 |
| ActionScript | 8.139775436837226 |
| Ada | 4.00042700953189 |
| Agda | 3.9126438455441743 |
| Alchemist | 3.507827758789091 |
| Alex | 5.000001311300139 |
| Alloy | 5.000488222547574 |
| Android Interface Definition Language | 11.0048217363656 |
| Arvo | 5.9999994741776135 |
| AsciiDoc | 3.5 |
| Assembly | 4.75 |
| AutoHotKey | 2.2087400984292067 |
| Autoconf | 5.8725585937792175 |
| BASH | 2.1289059027401294 |
| Basic | 3.003903865814209 |
| Batch | 6.527053831937014 |
| Bazel | 3.18005371087348 |
| Bitbake | 2.015624999069132 |
| Bitbucket Pipeline | 2.063491820823401 |
| Boo | 4.010679721835899 |
| Bosque | 4.98316764831543 |
| Brainfuck | 4.2025654308963425 |
| BuildStream | 3.4058846323741645 |
| C | 4.923767089530871 |
| C Header | 4.8744963703211965 |
| C Shell | 3.027952311891569 |
| C# | 3.9303305113013427 |
| C++ | 3.765686050057411 |
| C++ Header | 5.0468749664724015 |
| CMake | 4.474763816174707 |
| COBOL | 2.718678008809146 |
| CSS | 3.158353805542812 |
| CSV | 2.0005474090593514 |
| Cabal | 2.0234456174658693 |
| Cargo Lock | 2.602630615232607 |
| Cassius | 3.56445312181134 |
| Ceylon | 4.750730359584461 |
| Clojure | 3.992209411809762 |
| ClojureScript | 4.905477865257108 |
| Closure Template | 6.800760253008946 |
| CoffeeScript | 4.503051759227674 |
| ColdFusion | 6.124976545410084 |
| ColdFusion CFScript | 6.188602089623717 |
| Coq | 4.000243186950684 |
| Creole | 3.124526690922411 |
| Crystal | 3.1243934621916196 |
| Cython | 5.219657994911814 |
| D | 9.291626930357722 |
| Dart | 3.939864161220478 |
| Device Tree | 6.530643464186369 |
| Dhall | 0.12061593477278201 |
| Docker ignore | 2.9984694408020562 |
| Dockerfile | 3.1281526535752064 |
| Document Type Definition | 6.3923129292499254 |
| Elixir | 3.9999989270017977 |
| Elm | 2.968016967181992 |
| Emacs Dev Env | 4.750648772301943 |
| Emacs Lisp | 2.0156250001746203 |
| Erlang | 4.756546300111156 |
| Expect | 5.126907349098477 |
| Extensible Stylesheet Language Transformations | 4.519531239055546 |
| F# | 5.752862453457055 |
| F* | 4.063724638864983 |
| FIDL | 4.484130888886213 |
| FORTRAN Legacy | 6.117128185927898 |
| FORTRAN Modern | 5.742561882347131 |
| Fish | 3.993835387425861 |
| Flow9 | 9.462829245721366 |
| Forth | 4.016601327653859 |
| Fragment Shader File | 3.8598623261805187 |
| Freemarker Template | 11.122007250069213 |
| Futhark | 6.188476562965661 |
| GDScript | 3.2812499999872675 |
| GLSL | 6.6093769371505005 |
| GN | 3.497192621218512 |
| Game Maker Language | 4.968749999941792 |
| Game Maker Project | 3.8828125 |
| Gherkin Specification | 3.999099795268081 |
| Go | 3.9588454874029275 |
| Go Template | 4 |
| Gradle | 2.655930499769198 |
| Groovy | 11.499969503013528 |
| HEX | 3.98394775342058 |
| HTML | 4.564478578133282 |
| Hamlet | 3.4842224120074867 |
| Handlebars | 4.998766578761208 |
| Happy | 5.699636149570479 |
| Haskell | 2.000140870587468 |
| Haxe | 5.999999999999997 |
| IDL | 6.249999993495294 |
| Idris | 3.515075657458509 |
| Intel HEX | 3.983397483825683 |
| Isabelle | 4.18351352773584 |
| JAI | 7.750007518357038 |
| JSON | 3.9999972562254724 |
| JSONL | 5.751412352804029 |
| JSX | 5.0041952044625715 |
| Jade | 4.744544962807595 |
| Janet | 3.0312496423721313 |
| Java | 11.265740856469563 |
| JavaScript | 4.242187985224513 |
| JavaServer Pages | 7.999993488161865 |
| Jenkins Buildfile | 2.000000000087315 |
| Jinja | 6.937498479846909 |
| Julia | 3.9999848530092095 |
| Julius | 3.187606761406953 |
| Jupyter | 2.375 |
| Just | 4.312155187124516 |
| Korn Shell | 7.0685427486899925 |
| Kotlin | 6.455277973786039 |
| LD Script | 5.015594720376608 |
| LESS | 5.999999999999886 |
| LEX | 5.6996263030493495 |
| LOLCODE | 3.722656242392418 |
| LaTeX | 4.499990686770616 |
| Lean | 4.1324310302734375 |
| License | 4.7715609660297105 |
| Lisp | 6.00048828125 |
| Lua | 3.999999057474633 |
| Lucius | 3.0000303482974573 |
| Luna | 4.758178874869392 |
| MQL Header | 5.421851994469764 |
| MQL4 | 5.171874999953652 |
| MQL5 | 4.069171198975555 |
| MSBuild | 4.8931884765733855 |
| MUMPS | 4.999999672174454 |
| Macromedia eXtensible Markup Language | 3.9139365140181326 |
| Madlang | 3.625 |
| Makefile | 4.717208385332443 |
| Mako | 4.0349732004106045 |
| Markdown | 2.25 |
| Meson | 3.342019969206285 |
| Modula3 | 3.980173215190007 |
| Module-Definition | 8.875000973076205 |
| Monkey C | 3.0672508481368164 |
| Mustache | 6.000003708292297 |
| Nim | 3.7500824918105313 |
| Nix | 2.0307619677526234 |
| OCaml | 3.269392550457269 |
| Objective C | 3.526367187490962 |
| Objective C++ | 5.000000834608569 |
| Opalang | 4.500069382134143 |
| Org | 5.953919619084296 |
| Oz | 4.125 |
| PHP | 7.999984720368943 |
| PKGBUILD | 4.875488281252839 |
| PSL Assertion | 5.004394620715175 |
| Pascal | 5.0781240425935845 |
| Patch | 3.999999999999819 |
| Perl | 4.691352904239976 |
| Plain Text | 5.247085583343509 |
| Polly | 2.953125 |
| Pony | 2.9688720703125 |
| Powershell | 4.596205934882159 |
| Processing | 3.999931812300937 |
| Prolog | 4.4726600636568055 |
| Properties File | 3.5139240025278604 |
| Protocol Buffers | 6.544742336542192 |
| Puppet | 6.662078857422106 |
| PureScript | 4.000007774680853 |
| Python | 5.4531080610843805 |
| Q# | 3.7499999999999996 |
| QCL | 2.992309644818306 |
| QML | 7.042003512360623 |
| R | 3.0628376582587578 |
| Rakefile | 4.78515574071335 |
| Razor | 8.062499530475186 |
| ReStructuredText | 5.061766624473476 |
| Report Definition Language | 5.996573380834889 |
| Robot Framework | 4.0104638249612155 |
| Ruby | 5.1094988621717725 |
| Ruby HTML | 5.57654969021678 |
| Rust | 3.2265624976654292 |
| SAS | 4.826202331129183 |
| SKILL | 6.039547920227052 |
| SPDX | 4.000203706655157 |
| SQL | 7.701822280883789 |
| SRecode Template | 3.500030428171159 |
| SVG | 5.217570301278483 |
| Sass | 6.000000000056957 |
| Scala | 4.398563579539738 |
| Scheme | 6.999969714792911 |
| Scons | 5.010994006631478 |
| Shell | 4.988665378738929 |
| Smarty Template | 5.000527858268356 |
| Softbridge Basic | 4.87873840331963 |
| Specman e | 5.765624999999318 |
| Spice Netlist | 3.9687499998835882 |
| Standard ML (SML) | 4.031283043158929 |
| Stata | 6.27345275902178 |
| Stylus | 3.5000006667406485 |
| Swift | 3 |
| Swig | 5.246093751920853 |
| SystemVerilog | 2.9995259092956985 |
| Systemd | 3.9960937500000284 |
| TCL | 2.508188682367951 |
| TOML | 2.063069331460588 |
| TaskPaper | 2.003804363415667 |
| TeX | 3.500000000931251 |
| Thrift | 4.956119492650032 |
| Twig Template | 8.952746974652655 |
| TypeScript | 4.976589231140677 |
| TypeScript Typings | 5.832031190521718 |
| Unreal Script | 4.22499089783372 |
| Ur/Web | 4.41992186196147 |
| Ur/Web Project | 5.1147780619789955 |
| V | 4.251464832544997 |
| VHDL | 4.000000961231823 |
| Vala | 3.99804687498741 |
| Varnish Configuration | 4.006103516563625 |
| Verilog | 3.6906727683381173 |
| Verilog Args File | 8.93109059158814 |
| Vertex Shader File | 3.8789061926163697 |
| Vim Script | 3.9995117782528147 |
| Visual Basic | 4.5 |
| Visual Basic for Applications | 3.6874962672526417 |
| Vue | 7.752930045514701 |
| Wolfram | 3.075198844074798 |
| Wren | 4 |
| XAML | 4.515627968764219 |
| XCode Config | 6.969711296260638 |
| XML | 6 |
| XML Schema | 5.807670593268995 |
| Xtend | 4.315674404631856 |
| YAML | 3.2037304108964673 |
| Zig | 3.4181210184442534 |
| Zsh | 2.0616455049940288 |
| gitignore | 2.51172685490884 |
| ignore | 10.6434326171875 |
| m4 | 3.7519528857323934 |
| nuspec | 4.109375 |
| sed | 4.720429063539986 |


### YAML or YML?

Sometime back on the company slack there was a "discussion" with many dying on one hill or the other over the use of .yaml or .yml

The debate can finally(?) be ended. Although I suspect some will still prefer to die on their chosen hill.

| extension | count |
| ----------- | ----- |
| yaml | 3,572,609 |
| yml | 14,076,349 |

### Upper lower or mixed case?

What case style is used on filenames? This includes the extension so you would expect it to be mostly mixed case.

| style | count |
| ----- | ----- |
| mixed | 9,094,732 |
| lower | 2,476 |
| upper | 2,875 |

Which of course is not very interesting because generally file extensions are lowercase. What about if we ignore the file extension?

| style | count |
| ----- | ----- |
| mixed | 8,104,053 |
| lower | 347,458 |
| upper | 614,922 |

Not what I would have expected. Mostly mixed is normal, but I would have thought lower would be more popular.

### Java Factories

Another one that came up in the internal company slack when looking through some old Java code. I thought why not add a check for any Java code that has Factory, FactoryFactory or FactoryFactoryFactory in the name. The idea being to see how many factories are out there.

| type | count | percent |
| ---- | ----- | ------- |
| not factory | 271,375,574 | 97.9% |
| factory | 5,695,568 | 2.09% |
| factoryfactory | 25,316 | 0.009% |
| factoryfactoryfactory | 0 | 0% |

So slightly over 2% of all the Java code that I checked appeared to be a factory or factoryfactory. Thankfully there are no factoryfactoryfactories and perhaps that joke can finally die, although I am sure at least one non-ironic one exist somewhere in some Java 5 monolith that makes more money every day than I will see over my entire working life.

### Ignore files

The .ignore file idea was hammered out by burntsushi and ggreer in a [Hacker News thread](https://news.ycombinator.com/item?id=12564442) and is possibly one of the greatest cases of "competing" open source tools working together to a good outcome and done in record time. It has become the defacto way to add things into source control yet have tools ignore them. As it turns out `scc` also implements .ignore files but counts them as well. Lets see how well the idea has spread.

[skip table to next section](#future-ideas)

<div class="table-13"></div>

| .ignore count | project count |
| -----------------| ------------- |
| 0 | 9,088,796 |
| 1 | 7,848 |
| 2 | 1,258 |
| 3 | 508 |
| 4 | 333 |
| 5 | 43 |
| 6 | 130 |
| 7 | 8 |
| 8 | 14 |
| 9 | 83 |
| 10 | 49 |
| 11 | 35 |
| 12 | 112 |
| 13 | 736 |
| 15 | 4 |
| 17 | 1 |
| 18 | 4 |
| 20 | 2 |
| 21 | 1 |
| 23 | 2 |
| 24 | 3 |
| 26 | 2 |
| 27 | 1 |
| 34 | 31 |
| 35 | 19 |
| 36 | 9 |
| 38 | 2 |
| 39 | 1 |
| 43 | 12 |
| 44 | 1 |
| 45 | 2 |
| 46 | 5 |
| 49 | 7 |
| 50 | 7 |
| 51 | 12 |
| 52 | 2 |

## Future ideas

Id love to do some analysis of tabs vs spaces. Scanning for things like AWS AKIA keys and the like would be pretty neat as well. Id also love to expand out the bitbucket and gitlab coverage and get it broken down via each to see if groups of developers from different camps hang out in different areas.

Shortcomings id love to overcome in the above if I decide to do this again.

 - Keeping the URL properly in the metadata somewhere. Using a filename to store this was a bad idea as it was lossy and means it can be hard to identify the file source and location.
 - Not bother with S3. There is little point to pay the bandwidth cost when I was only using it for storage. Better to just stuff into the tar file from the beginning.
 - Invest some time in learning some tool to help with plotting and charting of results.
 - Use a trie or some other data type to keep a full count of filenames rather than the slightly lossy approach I used.
 - Add an option to scc to check the type of the file based on keywords as examples such as https://bitbucket.org/abellnets/hrossparser/src/master/xml_files/CIDE.C was picked up as being a C file despite obviously being HTML when the content is inspected. To be fair all code counters I tried behave the same way.
 - There appears to be a bug in scc where if a file has no extension but is named as one it will match that file which is incorrect. A bug has been raised in scc to address this https://github.com/boyter/scc/issues/114
 - I'd like to add shebang detection into scc https://github.com/boyter/scc/issues/115
 - Some sort of check against number of github stars would be pretty neat.
 - Analysis against the number of commits would be very interesting.
 - I want to add maintainability index calculations at some point. It would be very cool to see what projects are considered the most maintainable based on their size.

## So why bother?

Well I can take some of this information and plug it into searchcode.com and scc. Even if only some useful data points. The stated goal was pretty much this and it is potentially very useful to know how your project compares to others. Besides it was a fun way to spend a few days solving some interesting problems. Also I think it is safe to say that `scc` is a fairly reliable tool at this point.

In addition, I am working on a tool that helps senior-developer or manager types analyze code looking for languages, large files, flaws etc... with the assumption you have to watch multiple repositories. You put in some code and it will tell you how maintainable it is and what skills you need to maintain it. Useful for determining if you should buy or maintain some code-base and getting an overview of what your development team is producing. Should in theory help teams scale through shared resources. Something like AWS Macie but for code is the angle I am working with. It's something I need for my day job and I suspect others may find use in it, or at least thats the theory.

I should probably put an email sign up for that here at some point to gather interest for that.

## Raw / Processed Files

I have included a link to the [processed files](/static/an-informal-survey/results.zip) (20 MB) for those who wish to do their own analysis and corrections. If someone wants to host the raw files to allow others to download it let me know. It is a 83 GB tar.gz file which uncompressed is just over 1 TB in size. It contents consists of just over 9 million JSON files of various sizes.