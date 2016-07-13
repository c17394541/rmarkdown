---
title: "knitr Language Engines"
output:
  html_document:
    toc: true
    toc_depth: 2
    toc_float: true
---

## Overview

In addition to executing R code chunks, the [kntir](http://yihui.name/knitr/) package can also execute chunks in a variety of other languages. Some of the available language engines include:

- Rcpp
- Python
- SQL
- Bash
- Stan
- JavaScript
- CSS

For additional documentation and examples as well as a list of all supported engines see the [language engines](http://yihui.name/knitr/demo/engines/) section of the knitr website.

To process a code chunk using an alternate language engine you simply use the name of the engine in place of `r` in your chunk declaration, for example:

<pre class="markdown"><code>&#96;&#96;&#96;{bash}
cat flights1.csv flights2.csv flights3.csv > flights.csv
&#96;&#96;&#96;
</code></pre>

## Rcpp

The [Rcpp](http://rcpp.org) engine enables compilation of C++ into R functions via the Rcpp `sourceCpp` function. For example:


<pre class="markdown"><code>&#96;&#96;&#96;{Rcpp}
#include &lt;Rcpp.h&gt;
using namespace Rcpp;

// [[Rcpp::export]]
NumericVector timesTwo(NumericVector x) {
  return x * 2;
}
&#96;&#96;&#96;
</code></pre>

Executing this chunk will compile the code and make the `timesTwo` C++ function available to R.

### Caching

You can cache the compilation of C++ code chunks using standard knitr caching. Note however that this feature currently requires development versions of both the Rcpp and knitr packages, which you can install as follows:

```r
devtools::install_github("RcppCore/Rcpp")
devtools::install_github("yihui/knitr")
```

To cache the compilation of an Rcpp chunk simply add the `cache = TRUE` option to the chunk:

<pre class="markdown"><code>&#96;&#96;&#96;{Rcpp, cache=TRUE}
#include &lt;Rcpp.h&gt;
using namespace Rcpp;

// [[Rcpp::export]]
NumericVector timesTwo(NumericVector x) {
  return x * 2;
}
&#96;&#96;&#96;
</code></pre>


### Combining Chunks

In some cases it's desirable to combine all of the Rcpp code chunks in a document into a single compilation unit. This is especially useful when you want to intersperse narrative between pieces of C++ code (e.g. for a tutorial or user guide). It also reduces total compilation time for the document (since there is only a single invocation of the C++ compiler rather than multiple).

To combine all Rcpp chunks into a single compilation unit you use the `ref.label` chunk option along with the `knitr::all_rcpp_labels()` function to collect all of the Rcpp chunks in the document. Here's a simple example:

<pre class="markdown"><code>&#96;&#96;&#96;{Rcpp, ref.label=knitr::all_rcpp_labels(), cache=TRUE, include=FALSE}
&#96;&#96;&#96;
</code></pre>

<pre class="markdown"><code>&#96;&#96;&#96;{Rcpp, eval = FALSE}
#include &lt;Rcpp.h&gt;
&#96;&#96;&#96;
</code></pre>

<pre class="markdown"><code>&#96;&#96;&#96;{Rcpp, eval = FALSE}
// [[Rcpp::export]]
int timesTwo(int x) {
  return x * 2;
}
&#96;&#96;&#96;
</code></pre>

The two Rcpp chunks that include code will be collected and compiled together in the first Rcpp chunk via the `ref.label` chunk option. Note that we set the `eval = FALSE` option on the Rcpp chunks with code in them to prevent them from being compiled again.


## Python

The [python](https://www.python.org/) engine enables execution of python code via an external python interpreter (the interpreter returned by `Sys.which("python")` is used to execute the code). Here's a simple example:

<pre class="markdown"><code>&#96;&#96;&#96;{python}
x = 'hello, python world!'
print(x.split(' '))
&#96;&#96;&#96;
</code></pre>

Note that chunk options like `echo` and `results` are all valid when using a language engine like python. If your python code is generating raw HTML or LaTeX then the `results='asis'` option will ensure that it's passed straight into the document's output stream.

### Data Exchange

Since the python engine executes code in an external process, exchanging data between R chunks and python chunks is done via the file system. If you are exchanging data frames, you can use the [feather](https://blog.rstudio.org/2016/03/29/feather/) package for very high performance transfer of even large data frames between python and R. Here's an example that uses feather to transfer a data frame created with pandas to R for plotting with ggplot2:

<pre class="markdown"><code>&#96;&#96;&#96;{python}
import pandas
import feather

# Read flights data and select flights to O'Hare
flights = pandas.read_csv("flights.csv")
flights = flights[flights['dest'] == "ORD"]

# Select carrier and delay columns and drop rows with missing values
flights = flights[['carrier', 'dep_delay', 'arr_delay']]
flights = flights.dropna()
print flights.head(10)

# Write to feather file for reading from R
feather.write_dataframe(flights, "flights.feather")
&#96;&#96;&#96;
</code></pre>

Now we read the feather file from R and plot the data frame using ggplot2:

<pre class="markdown"><code>&#96;&#96;&#96;{r}
library(feather)
library(ggplot2)

# Read from feather and plot
flights <- read_feather("flights.feather")
ggplot(flights, aes(carrier, arr_delay)) + geom_point() + geom_jitter()
&#96;&#96;&#96;
</code></pre>

## SQL

The [SQL](https://en.wikipedia.org/wiki/SQL) engine uses the [DBI](https://github.com/rstats-db/DBI) package to execute SQL queries, print their results, and optionally assign the results to a data frame. The SQL engine is currently only available in the development version of knitr, which you can install as follows:

```r
devtools::install_github("yihui/knitr")
```

To use the knitr SQL engine you first need to establish a DBI connection to a database (typically via the `dbConnect` function). You can make use of this connection in a SQL chunk via the `connection` option. For example:

<pre class="markdown"><code>&#96;&#96;&#96;{r}
library(DBI)
db <- dbConnect(RSQLite::SQLite(), dbname = "sql.sqlite")
&#96;&#96;&#96;
</code></pre>

<pre class="markdown"><code>&#96;&#96;&#96;{sql, connection=db}
SELECT * FROM trials
&#96;&#96;&#96;
</code></pre>

By default, this will display records from the `trials` table within the document. 

### Data Frames

If you want to assign the results of the SQL query to an R data frame, you can do this using the `output.var` option, for example:

<pre class="markdown"><code>&#96;&#96;&#96;{sql, connection=db, output.var=trials}
SELECT * FROM trials
&#96;&#96;&#96;
</code></pre>

### Parameters

If you need to bind the values of R variables into SQL queries, you can do so by prefacing R variable references with a `?`. For example:

<pre class="markdown"><code>&#96;&#96;&#96;{r}
subjects <- 10
&#96;&#96;&#96;
</code></pre>

<pre class="markdown"><code>&#96;&#96;&#96;{sql, connection=db, output.var="trials"}
SELECT * FROM trials WHERE subjects >= ?subjects
&#96;&#96;&#96;
</code></pre>


## Stan

The [Stan](http://mc-stan.org/) engine enables embedding of the Stan probabilistic programming language within R Markdown documents. The Stan model within the code chunk is compiled into a `stanmodel` object and is assigned it to a variable with the name given by the `x` option. For example:

<pre class="markdown"><code>&#96;&#96;&#96;{stan, x="ex1"}
parameters {
  real y[2]; 
} 
model {
  y[1] ~ normal(0, 1);
  y[2] ~ double_exponential(0, 2);
} 
&#96;&#96;&#96;
</code></pre>

<pre class="markdown"><code>&#96;&#96;&#96;{r}
library(rstan)
fit <- sampling(ex1) 
print(fit)
&#96;&#96;&#96;
</code></pre>


## Bash

The [bash](https://en.wikipedia.org/wiki/Bash_(Unix_shell)) engine enables the execution of shell scripts via the bash interpreter (note that `sh` and `zsh` engines are also available). For example:

<pre class="markdown"><code>&#96;&#96;&#96;{bash}
cat flights1.csv flights2.csv flights3.csv > flights.csv
&#96;&#96;&#96;
</code></pre>

## JavaScript

If you are using an R Markdown format that targets HTML output (e.g. [html_document](html_document_format), [ioslides_presenation](ioslides_presentation_format.html), etc.) then you can include JavaScript to be executed within the HTML page using the [JavaScript](https://en.wikipedia.org/wiki/JavaScript) engine.

For example, the following chunk uses [jQuery](https://jquery.com/) (which is included in most R Markdown HTML formats) to hide the document title:

<pre class="markdown"><code>&#96;&#96;&#96;{js}
$('.title').remove()
&#96;&#96;&#96;
</code></pre>

Note that the JavaScript engine is specified using the abbreviation `js`.

## CSS

If you are using an R Markdown format that targets HTML output (e.g. [html_document](html_document_format), [ioslides_presenation](ioslides_presentation_format.html), etc.) then you can include CSS to applied to the HTML page using the [CSS](https://en.wikipedia.org/wiki/CSS) engine.

For example, the following code chunk turns text within the document body red:

<pre class="markdown"><code>&#96;&#96;&#96;{css}
body {
  color: red;
}
&#96;&#96;&#96;
</code></pre>
