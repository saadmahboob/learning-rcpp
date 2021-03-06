My main goal in this educational endeavor is to be able to use the [MLPACK](http://www.mlpack.org/), [Shark](http://image.diku.dk/shark/), and [dlib](http://dlib.net/) C++ machine learning libraries in R for computationally intensive problems. Now, there is a [RcppMLPACK](https://cran.r-project.org/package=RcppMLPACK), but that one apparently uses version 1 of MLPACK (which is now in version 2) and doesn't include any supervised learning methods, just unsupervised learning methods.

-   [Setup](#setup)
    -   [Software Libraries](#software-libraries)
    -   [Mac OS X](#mac-os-x)
    -   [Ubuntu/Debian](#ubuntudebian)
        -   [Building Shark library](#building-shark-library)
    -   [R Packages](#r-packages)
-   [Rcpp](#rcpp)
    -   [Basics](#basics)
    -   [Using Libraries](#using-libraries)
        -   [Armadillo vs RcppArmadillo](#armadillo-vs-rcpparmadillo)
        -   [Fast K-Means](#fast-k-means)
    -   [Fast Classification](#fast-classification)
        -   [External Pointers](#external-pointers)
    -   [Other Libraries](#other-libraries)
        -   [Shark](#shark)
            -   [Classification](#classification)
        -   [DLib](#dlib)
    -   [Random Number Generation](#random-number-generation)
    -   [Serialization](#serialization)
        -   [Armadillo Serialization (Not Working)](#armadillo-serialization-not-working)
-   [References](#references)

Setup
=====

Software Libraries
------------------

Mac OS X
--------

``` bash
## To install Homebrew:
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
## Then:
brew tap homebrew/versions && brew tap homebrew/science && brew update
# brew install gcc --enable-cxx && brew link --overwrite gcc && brew link cmake
brew install boost --c++11
# Installing cmake may require: sudo chown -R `whoami` /usr/local/share/man/man7
brew install mlpack shark dlib
```

Ubuntu/Debian
-------------

``` bash
sudo apt-get install libmlpack-dev libdlib-dev
```

### Building Shark library

If `sudo apt-get install libshark-dev` is no go, we have to build the library ourselves. See [these installation instructions](http://image.diku.dk/shark/sphinx_pages/build/html/rest_sources/getting_started/installation.html) for more details.

``` bash
# Install dependencies
sudo apt-get install cmake cmake-curses-gui libatlas-base-dev wget

# Download and unpack
wget -O Shark-3.0.0.tar.gz https://github.com/Shark-ML/Shark/archive/v3.0.0.tar.gz
tar -zxvf Shark-3.0.0.tar.gz
mv Shark-3.0.0 Shark

# Configure and build
mkdir Shark/build/
cd Shark/build
# cmake "-DENABLE_OPENMP=OFF" "-DCMAKE_INSTALL_PREFIX=/usr/local/" ../
cmake ../
make
make install
```

R Packages
----------

``` r
install.packages(c(
  "BH", # Header files for 'Boost' C++ library
  "Rcpp", # R and C++ integration
  "RcppArmadillo", # Rcpp integration for 'Armadillo' linear algebra library
  "Rcereal", # header files of 'cereal', a C++11 library for serialization
  "microbenchmark", # For benchmarking performance
  "devtools", # For installing packages from GitHub
  "magrittr", # For piping
  "knitr", # For printing tables & data.frames as Markdown
  "toOrdinal" # Cardinal to ordinal number conversion function (e.g. 1 => "1st")
), repos = "https://cran.rstudio.com/")
devtools::install_github("yihui/printr") # Prettier table printing
```

If you get "ld: library not found for -lgfortran" error when trying to install RcppArmadillo, run:

``` bash
curl -O http://r.research.att.com/libs/gfortran-4.8.2-darwin13.tar.bz2
sudo tar fvxz gfortran-4.8.2-darwin13.tar.bz2 -C /
```

See "[Rcpp, RcppArmadillo and OS X Mavericks "-lgfortran" and "-lquadmath" error](http://thecoatlessprofessor.com/programming/rcpp-rcpparmadillo-and-os-x-mavericks-lgfortran-and-lquadmath-error/)"" for more info.

Rcpp
====

See [this section](http://rmarkdown.rstudio.com/authoring_knitr_engines.html#rcpp) in [RMarkdown documentation](http://rmarkdown.rstudio.com/) for details on Rcpp chunks.

``` r
library(magrittr)
library(BH)
library(Rcpp)
library(RcppArmadillo)
library(Rcereal)
library(microbenchmark)
library(knitr)
library(printr)
```

Basics
------

``` cpp
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
NumericVector cumSum(NumericVector x) {
  int n = x.size();
  NumericVector out(n);
  out[0] = x[0];
  for (int i = 1; i < n; ++i) {
    out[i] = out[i-1] + x[i];
  }
  return out;
}
```

``` r
x <- 1:1000
microbenchmark(
  native = cumsum(x),
  loop = (function(x) {
    output <- numeric(length(x))
    output[1] <- x[1]
    for (i in 2:length(x)) {
      output[i] <- output[i-1] + x[i]
    }
    return(output)
  })(x),
  Rcpp = cumSum(x)
) %>% summary(unit = "ms") %>% knitr::kable(format = "markdown")
```

| expr   |     min|      lq|    mean|  median|      uq|     max|  neval|
|:-------|-------:|-------:|-------:|-------:|-------:|-------:|------:|
| native |  0.0025|  0.0032|  0.0049|  0.0043|  0.0057|  0.0182|    100|
| loop   |  0.8891|  1.0334|  1.4934|  1.2278|  1.8407|  4.0855|    100|
| Rcpp   |  0.0043|  0.0061|  0.0145|  0.0112|  0.0155|  0.1262|    100|

Using Libraries
---------------

### Armadillo vs RcppArmadillo

Use the **depends** attribute to bring in [RcppArmadillo](https://cran.r-project.org/package=RcppArmadillo), which is an Rcpp integration of the templated linear algebra library [Armadillo](http://arma.sourceforge.net/). The code below is an example of a fast linear model from Dirk Eddelbuettel.

``` cpp
// [[Rcpp::depends(RcppArmadillo)]]
#include <RcppArmadillo.h>
using namespace Rcpp;
using namespace arma;
// [[Rcpp::export]]
List fastLm(const colvec& y, const mat& X) {
  int n = X.n_rows, k = X.n_cols;
  colvec coef = solve(X, y);
  colvec resid = y - X*coef;
  double sig2 = as_scalar(trans(resid) * resid/(n-k));
  colvec stderrest = sqrt(sig2 * diagvec( inv(trans(X)*X)) );
  return List::create(_["coefficients"] = coef,
                      _["stderr"]       = stderrest,
                      _["df.residual"]  = n - k );
}
```

``` r
data("mtcars", package = "datasets")
microbenchmark(
  lm = lm(mpg ~ wt + disp + cyl + hp, data = mtcars),
  fastLm = fastLm(mtcars$mpg, cbind(1, as.matrix(mtcars[, c("wt", "disp", "cyl", "hp")]))),
  RcppArm = fastLmPure(cbind(1, as.matrix(mtcars[, c("wt", "disp", "cyl", "hp")])), mtcars$mpg)
) %>% summary(unit = "ms") %>% knitr::kable(format = "markdown")
```

| expr    |     min|      lq|    mean|  median|      uq|     max|  neval|
|:--------|-------:|-------:|-------:|-------:|-------:|-------:|------:|
| lm      |  0.8488|  1.1605|  2.1720|  1.5225|  2.5655|  13.095|    100|
| fastLm  |  0.0928|  0.1129|  0.2903|  0.1383|  0.2211|   4.413|    100|
| RcppArm |  0.1213|  0.1671|  0.3172|  0.2150|  0.3276|   3.044|    100|

### Fast K-Means

Unfortunately, [RcppMLPACK](https://cran.r-project.org/package=RcppMLPACK) uses version 1 of [MLPACK](http://www.mlpack.org/) (now in version 2) and only makes the unsupervised learning methods accessible. (Supervised methods would require returning a trained classifier object to R, which is actually a really difficult problem.)

Okay, let's try to get a fast version of <span title="Bradley, P. S., &amp; Fayyad, U. M. (1998). Refining Initial Points for K-Means Clustering. Icml." style="font-weight: bold;">k-means</span>.

First, install the MLPACK library (see [§ Software Libraries](#software-libraries)), then:

``` r
# Thanks to Kevin Ushey for suggesting Rcpp plugins (e.g. Rcpp:::.plugins$openmp)
registerPlugin("mlpack11", function() {
  return(list(env = list(
    USE_CXX1X = "yes",
    CXX1XSTD="-std=c++11",
    PKG_LIBS = "-lmlpack"
  )))
})
```

We refer to the documentation for [KMeans](http://www.mlpack.org/docs/mlpack-1.0.6/doxygen.php?doc=kmtutorial.html#kmeans_kmtut) shows, although it seems to incorrectly use `arma::Col<size_t>` for cluster assigments while in practice the cluster assignments are returned as an `arma::Row<size_t>` object.

``` cpp
// [[Rcpp::plugins(mlpack11)]]
// [[Rcpp::depends(RcppArmadillo)]]

#include <RcppArmadillo.h>
using namespace Rcpp;
#include <mlpack/core/util/log.hpp>
#include <mlpack/methods/kmeans/kmeans.hpp>
using namespace mlpack::kmeans;
using namespace arma;

// [[Rcpp::export]]
NumericVector mlpackKM(const arma::mat& data, const size_t& clusters) {
  Row<size_t> assignments;
  KMeans<> k;
  k.Cluster(data, clusters, assignments);
  // Let's change the format of the output to be a little nicer:
  NumericVector results(data.n_cols);
  for (int i = 0; i < assignments.n_cols; i++) {
    results[i] = assignments(i) + 1; // cluster assignments are 0-based
  }
  return results;
}
```

(Alternatively: `sourceCpp("src/fastKm.cpp")` which creates `fastKm()` from [src/fastKm.cpp](src/fastKm.cpp))

``` r
data(trees, package = "datasets"); data(faithful, package = "datasets")
# kmeans coerces data frames to matrix, so it's worth doing that beforehand
mtrees <- as.matrix(trees)
mfaithful <- as.matrix(faithful)
# KMeans in MLPACK requires observations to be in columns, not rows:
ttrees <- t(trees); tfaithful <- t(faithful)
microbenchmark(
  kmeans_trees = kmeans(mtrees, 3),
  mlpackKM_trees = mlpackKM(ttrees, 3),
  kmeans_faithful = kmeans(mfaithful, 2),
  mlpackKM_faithful = mlpackKM(tfaithful, 2)
) %>% summary(unit = "ms") %>% knitr::kable(format = "markdown")
```

| expr               |     min|      lq|    mean|  median|      uq|     max|  neval|
|:-------------------|-------:|-------:|-------:|-------:|-------:|-------:|------:|
| kmeans\_trees      |  0.1904|  0.2035|  0.3100|  0.2233|  0.3602|  1.5158|    100|
| mlpackKM\_trees    |  0.0215|  0.0314|  0.0545|  0.0389|  0.0570|  0.2402|    100|
| kmeans\_faithful   |  0.1986|  0.2174|  0.3409|  0.2353|  0.3692|  2.3983|    100|
| mlpackKM\_faithful |  0.0917|  0.1251|  0.1721|  0.1435|  0.1779|  0.5402|    100|

Fast Classification
-------------------

In this exercise, we will train a [Naive Bayes classifier from MLPACK](http://www.mlpack.org/docs/mlpack-2.1.0/doxygen.php?doc=classmlpack_1_1naive__bayes_1_1NaiveBayesClassifier.html). First, we train and classify in a single step. Then we will store the trained classifier in memory, and then later we will be able to save the model. Storing the trained model requires [serialization](https://en.wikipedia.org/wiki/Serialization), the topic of the next section.

``` cpp
// [[Rcpp::plugins(mlpack11)]]
// [[Rcpp::depends(RcppArmadillo)]]

#include <RcppArmadillo.h>
using namespace Rcpp;

#include <mlpack/core/util/log.hpp>
#include <mlpack/methods/naive_bayes/naive_bayes_classifier.hpp>
using namespace mlpack::naive_bayes;

// [[Rcpp::export]]
NumericVector mlpackNBC(const arma::mat& training_data, const arma::Row<size_t>& labels, const size_t& classes, const arma::mat& new_data) {
  // Initialization & training:
  NaiveBayesClassifier<> nbc(training_data, labels, classes);
  // Prediction:
  arma::Row<size_t> predictions;
  nbc.Classify(new_data, predictions);
  // Let's change the format of the output to be a little nicer:
  NumericVector results(predictions.n_cols);
  for (int i = 0; i < predictions.n_cols; ++i) {
    results[i] = predictions(i);
  }
  return results;
}
```

For the classification example, we'll use the Iris dataset. (Of course.)

``` r
data(iris, package = "datasets")
set.seed(0)
training_idx <- sample.int(nrow(iris), 0.8 * nrow(iris), replace = FALSE)
training_x <- unname(as.matrix(iris[training_idx, 1:4]))
training_y <- unname(iris$Species[training_idx])
testing_x <- unname(as.matrix(iris[-training_idx, 1:4]))
testing_y <- unname(iris$Species[-training_idx])
# For fastNBC:
ttraining_x <- t(training_x)
ttraining_y <- matrix(as.numeric(training_y) - 1, nrow = 1)
classes <- length(levels(training_y))
ttesting_x <- t(testing_x)
ttesting_y <- matrix(as.numeric(testing_y) - 1, nrow = 1)
```

I kept getting "Mat::col(): index out of bounds" error when trying to compile. I debugged the heck out of it until I finally looked in **naive\_bayes\_classifier\_impl.hpp** and saw:

``` cpp
for (size_t j = 0; j < data.n_cols; ++j)
{
  const size_t label = labels[j];
  ++probabilities[label];
  
  arma::vec delta = data.col(j) - means.col(label);
  means.col(label) += delta / probabilities[label];
  variances.col(label) += delta % (data.col(j) - means.col(label));
}
```

Hence why we run into a problem when we use `as.numeric(training_y)` in R and turn that factor into 1s, 2s, and 3s. This makes sense in retrospect but would have been nice to explicitly know that MLPACK expects training data class labels to be 0-based.

``` r
# Naive Bayes via e1071
naive_bayes <- e1071::naiveBayes(training_x, training_y)
predictions <- e1071:::predict.naiveBayes(naive_bayes, testing_x, type = "class")
confusion_matrix <- caret::confusionMatrix(
  data = predictions,
  reference = testing_y
)
confusion_matrix$table
```

| Prediction/Reference |  setosa|  versicolor|  virginica|
|:---------------------|-------:|-----------:|----------:|
| setosa               |       9|           0|          0|
| versicolor           |       0|          11|          1|
| virginica            |       0|           0|          9|

``` r
print(confusion_matrix$overall["Accuracy"])
```

    ## Accuracy 
    ##   0.9667

``` r
# Naive Bayes via MLPACK
predictions <- mlpackNBC(ttraining_x, ttraining_y, classes, ttesting_x)
confusion_matrix <- caret::confusionMatrix(
  data = predictions,
  reference = ttesting_y
)
confusion_matrix$table
```

| Prediction/Reference |    0|    1|    2|
|:---------------------|----:|----:|----:|
| 0                    |    9|    0|    0|
| 1                    |    0|   11|    1|
| 2                    |    0|    0|    9|

``` r
print(confusion_matrix$overall["Accuracy"])
```

    ## Accuracy 
    ##   0.9667

``` r
# Performance Comparison
microbenchmark(
  naiveBayes = {
    naive_bayes <- e1071::naiveBayes(training_x, training_y)
    predictions <- e1071:::predict.naiveBayes(naive_bayes, testing_x, type = "class")
  },
  fastNBC = mlpackNBC(ttraining_x, ttraining_y, classes, ttesting_x)
) %>% summary(unit = "ms") %>% knitr::kable(format = "markdown")
```

| expr       |     min|      lq|    mean|  median|      uq|      max|  neval|
|:-----------|-------:|-------:|-------:|-------:|-------:|--------:|------:|
| naiveBayes |  4.6171|  5.4975|  7.5072|   6.904|  8.5516|  18.9154|    100|
| fastNBC    |  0.0152|  0.0195|  0.0413|   0.041|  0.0472|   0.1733|    100|

### External Pointers

In the next step, we'll train a Naive Bayes classifier and keep that trained object in memory to make classification a separate step. Notice that we have to:

-   declare a pointer: `NaiveBayesClassifier<>* nbc = new NaiveBayesClassifier<>(...)`
-   use Rcpp's external pointers (`Rcpp::XPtr`) and
-   return an [S-expression](http://adv-r.had.co.nz/C-interface.html#c-data-structures) (`SEXP`).

``` cpp
// [[Rcpp::plugins(mlpack11)]]
// [[Rcpp::depends(RcppArmadillo)]]

#include <RcppArmadillo.h>
using namespace Rcpp;

#include <mlpack/core/util/log.hpp>
#include <mlpack/methods/naive_bayes/naive_bayes_classifier.hpp>
using namespace mlpack::naive_bayes;

// [[Rcpp::export]]
SEXP mlpackNBTrainXPtr(const arma::mat& training_data, const arma::Row<size_t>& labels, const size_t& classes) {
  // Initialization & training:
  NaiveBayesClassifier<>* nbc = new NaiveBayesClassifier<>(training_data, labels, classes);
  Rcpp::XPtr<NaiveBayesClassifier<>> p(nbc, true);
  return p;
}
```

``` r
fit <- mlpackNBTrainXPtr(ttraining_x, ttraining_y, classes)
str(fit)
```

    ## <externalptr>

`fit` is an external pointer to some memory. When we pass it to a C++ function, it's passed as an R data type (SEXP) that we have to convert to an external pointer before we can use the object's methods. Notice that we're now calling `nbc->Classify()` instead of `nbc.Classify()`.

``` cpp
// [[Rcpp::plugins(mlpack11)]]
// [[Rcpp::depends(RcppArmadillo)]]

#include <RcppArmadillo.h>
using namespace Rcpp;

#include <mlpack/core/util/log.hpp>
#include <mlpack/methods/naive_bayes/naive_bayes_classifier.hpp>
using namespace mlpack::naive_bayes;

// [[Rcpp::export]]
NumericVector mlpackNBClassifyXPtr(SEXP xp, const arma::mat& new_data) {
  XPtr<NaiveBayesClassifier<>> nbc(xp);
  // Prediction:
  arma::Row<size_t> predictions;
  nbc->Classify(new_data, predictions);
  // Let's change the format of the output to be a little nicer:
  NumericVector results(predictions.n_cols);
  for (int i = 0; i < predictions.n_cols; ++i) {
    results[i] = predictions(i);
  }
  return results;
}
```

``` r
fit_e1071 <- e1071::naiveBayes(training_x, training_y)
# Performance Comparison
microbenchmark(
  `e1071 prediction` = e1071:::predict.naiveBayes(fit_e1071, testing_x, type = "class"),
  `MLPACK prediction` = mlpackNBClassifyXPtr(fit, ttesting_x)
) %>% summary(unit = "ms") %>% knitr::kable(format = "markdown")
```

| expr              |     min|      lq|   mean|  median|      uq|    max|  neval|
|:------------------|-------:|-------:|------:|-------:|-------:|------:|------:|
| e1071 prediction  |  3.5747|  4.0661|  4.924|  4.5684|  5.4075|  14.18|    100|
| MLPACK prediction |  0.0091|  0.0108|  0.045|  0.0278|  0.0344|   2.08|    100|

See [Exposing C++ functions and classes with Rcpp modules](http://dirk.eddelbuettel.com/code/rcpp/Rcpp-modules.pdf) for more information.

Other Libraries
---------------

### Shark

For this one, I'm putting my work/notes into the [LearningRcppShark](LearningRcppShark/) package, which is how I'm learning the [Shark library](http://image.diku.dk/shark/), creating R bindings to it via RcppShark, and learning how to make an R package that makes use of an Rcpp-based library wrapper.

``` r
# devtools::install_github("bearloga/learning-rcpp/LearningRcppShark")
library(LearningRcppShark)
```

``` r
fit <- shark_kmeans(training_x, 3)
predictions <- predict(fit, testing_x)
```

``` r
# This version does not check if x is a numeric matrix
sharkKM <- function(x, k) {
  return(LearningRcppShark:::SharkKMeansTrain(x, k))
}

microbenchmark(
  kmeans_trees = kmeans(mtrees, 3),
  mlpackKM_trees = mlpackKM(ttrees, 3),
  shark_km_trees = shark_kmeans(mtrees, 3),
  sharkKM_trees = sharkKM(mtrees, 3),
  kmeans_faithful = kmeans(mfaithful, 2),
  mlpackKM_faithful = mlpackKM(tfaithful, 2),
  shark_km_faithful = shark_kmeans(mfaithful, 2),
  sharkKM_faithful = sharkKM(mfaithful, 2)
) %>% summary(unit = "ms") %>% knitr::kable(format = "markdown")
```

| expr                |     min|      lq|    mean|  median|      uq|      max|  neval|
|:--------------------|-------:|-------:|-------:|-------:|-------:|--------:|------:|
| kmeans\_trees       |  0.2019|  0.2302|  0.3185|  0.2561|  0.3615|   0.9179|    100|
| mlpackKM\_trees     |  0.0181|  0.0389|  0.0560|  0.0488|  0.0647|   0.1740|    100|
| shark\_km\_trees    |  0.1034|  0.1317|  0.1985|  0.1579|  0.2143|   0.7948|    100|
| sharkKM\_trees      |  0.0691|  0.0914|  0.1311|  0.1077|  0.1425|   0.4000|    100|
| kmeans\_faithful    |  0.2146|  0.2515|  0.3660|  0.2800|  0.3459|   1.8320|    100|
| mlpackKM\_faithful  |  0.0799|  0.1241|  0.1704|  0.1435|  0.1904|   0.4892|    100|
| shark\_km\_faithful |  0.2967|  0.3423|  0.9020|  0.3677|  0.4326|  47.8698|    100|
| sharkKM\_faithful   |  0.2600|  0.3016|  0.3852|  0.3223|  0.3723|   0.9881|    100|

#### Classification

### DLib

``` r
registerPlugin("dlib", function() {
  return(list(env = list(
    PKG_LIBS = "-ldlib"
  )))
})
```

``` cpp
// [[Rcpp::plugins(dlib)]]
#include <Rcpp.h>
using namespace Rcpp;
```

...

Random Number Generation
------------------------

[Rcpp sugar](http://adv-r.had.co.nz/Rcpp.html#rcpp-sugar) provides a bunch of useful features and high-level abstractions, such as statistical distribution functions. Let's create a function that yields a sample of *n* independent draws from Bernoulli(*p*):

``` cpp
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
NumericVector bernie(const int& n, const float& p) {
  return rbinom(n, 1, p);
}
```

> Section 6.3 of [Writing R Extensions](http://cran.r-project.org/doc/manuals/r-release/R-exts.html#Random-number-generation) describes an additional requirement for calling the R random number generation functions: you must call GetRNGState prior to using them and then PutRNGState afterwards. These functions (respectively) read .Random.seed and then write it out after use. When using Rcpp attributes (as we do via the // \[\[Rcpp::export\]\] annotation on the functions above) it is not necessary to call GetRNGState and PutRNGState because this is done automatically within the wrapper code generated for exported functions. In fact, since these calls don’t nest it is actually an error to call them when within a function exported via Rcpp attributes. ([Random number generation](http://gallery.rcpp.org/articles/random-number-generation/))

Let's just do a quick test to see if setting seed does what it should:

``` r
n <- 100; p <- 0.5; x <- list()
set.seed(0); x[[1]] <- bernie(n, p)
set.seed(0); x[[2]] <- bernie(n, p)
set.seed(42); x[[3]] <- bernie(n, p)
set.seed(42); x[[4]] <- bernie(n, p)
```

<table>
<colgroup>
<col width="20%" />
<col width="19%" />
<col width="19%" />
<col width="20%" />
<col width="20%" />
</colgroup>
<thead>
<tr class="header">
<th align="left"></th>
<th align="left">Seed: 0, 1st draw</th>
<th align="left">Seed: 0, 2nd draw</th>
<th align="left">Seed: 42, 1st draw</th>
<th align="left">Seed: 42, 2nd draw</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">Seed: 0, 1st draw</td>
<td align="left">All draws match</td>
<td align="left">All draws match</td>
<td align="left">Draws don't match</td>
<td align="left">Draws don't match</td>
</tr>
<tr class="even">
<td align="left">Seed: 0, 2nd draw</td>
<td align="left">All draws match</td>
<td align="left">All draws match</td>
<td align="left">Draws don't match</td>
<td align="left">Draws don't match</td>
</tr>
<tr class="odd">
<td align="left">Seed: 42, 1st draw</td>
<td align="left">Draws don't match</td>
<td align="left">Draws don't match</td>
<td align="left">All draws match</td>
<td align="left">All draws match</td>
</tr>
<tr class="even">
<td align="left">Seed: 42, 2nd draw</td>
<td align="left">Draws don't match</td>
<td align="left">Draws don't match</td>
<td align="left">All draws match</td>
<td align="left">All draws match</td>
</tr>
</tbody>
</table>

Let's see how the performance differs between it and an R-equivalent when *n*=1,000:

``` r
rbern <- function(n, p) {
  return(rbinom(n, 1, p))
}
microbenchmark(
  R = rbern(1e3, 0.49),
  Rcpp = bernie(1e3, 0.49)
) %>% summary(unit = "ms") %>% knitr::kable(format = "markdown")
```

| expr |     min|      lq|    mean|  median|      uq|     max|  neval|
|:-----|-------:|-------:|-------:|-------:|-------:|-------:|------:|
| R    |  0.0483|  0.0495|  0.0580|  0.0500|  0.0589|  0.2322|    100|
| Rcpp |  0.0286|  0.0296|  0.0348|  0.0299|  0.0313|  0.0989|    100|

Serialization
-------------

[Serialization](https://en.wikipedia.org/wiki/Serialization) is the process of translating data structures and objects into a format that can be stored. Earlier, we trained a Naive Bayes classifier and kept the trained object in memory, returning an external pointer to it, allowing us to classify new observations as long as it is done within the same session.

This one requires: C++11 capability (if compiled supports it, enable via `// [[Rcpp::plugins(cpp11)]]`) and [cereal](http://uscilab.github.io/cereal/) serialization library, available via [Rcereal package](https://cran.r-project.org/package=Rcereal).

Roughly, we're going to create a wrapper for NumericMatrix that is serializable. **Note**: unlike previous section where the Rcpp chunks were complete, this section has multiple Rcpp chunks that are stitched together using the "ref.label" knitr chunk parameter, allowing me to have notes in-between different functions that should be together. See [Combining Chunks](http://rmarkdown.rstudio.com/authoring_knitr_engines.html#combining-chunks) for more details.

``` cpp
// [[Rcpp::plugins(cpp11)]]
// [[Rcpp::depends(Rcereal)]]

// Enables us to keep serialization method internally:
#include <cereal/access.hpp>
// see http://uscilab.github.io/cereal/serialization_functions.html#non-public-serialization

// Use std::vector and make it serializable:
#include <vector>
#include <cereal/types/vector.hpp>
// see http://uscilab.github.io/cereal/stl_support.html for more info

// Cereal's binary archiving:
#include <sstream>
#include <cereal/archives/binary.hpp>
// see http://uscilab.github.io/cereal/serialization_archives.html
// and http://uscilab.github.io/cereal/quickstart.html for more info

#include <Rcpp.h>
using namespace Rcpp;

class SerializableNumericMatrix
{
private:
  int nrows;
  int ncols;
  std::vector<double> data;
  friend class cereal::access;
  // This method lets cereal know which data members to serialize
  template<class Archive>
  void serialize(Archive& ar)
  {
    ar( data, nrows, ncols ); // serialize things by passing them to the archive
  }
public:
  SerializableNumericMatrix(){};
  SerializableNumericMatrix(NumericMatrix x) {
    std::vector<double> y = as<std::vector<double>>(x); // Rcpp::as
    data = y;
    nrows = x.nrow();
    ncols = x.ncol();
  };
  NumericVector neuemericMatrix() {
    // Thanks to Kevin Ushey for the tip to use dim attribute
    // http://stackoverflow.com/a/19866956/1091835
    NumericVector d = wrap(data);
    d.attr("dim") = Dimension(nrows, ncols);
    return d;
  }
  NumericMatrix numericMatrix() {
    NumericMatrix d(nrows, ncols, data.begin());
    return d;
  }
};
```

Let's just test that everything works before adding \[de-\]serialization capabilities.

``` cpp
// [[Rcpp::export]]
NumericMatrix testSNM(NumericMatrix x) {
  SerializableNumericMatrix snm(x);
  return snm.numericMatrix();
}
```

``` r
x <- matrix(0:9, nrow = 2, ncol = 5)
(y <- testSNM(x))
```

|     |     |     |     |     |
|----:|----:|----:|----:|----:|
|    0|    2|    4|    6|    8|
|    1|    3|    5|    7|    9|

Yay! Okay, for this we are going to use the serialization/deserialization example from [Rcereal README.md](https://github.com/wush978/Rcereal/blob/master/README.md).

``` cpp
// [[Rcpp::export]]
RawVector serializeSNM(NumericMatrix x) {
  SerializableNumericMatrix snm(x);
  std::stringstream ss;
  {
    cereal::BinaryOutputArchive oarchive(ss); // Create an output archive
    oarchive(snm);
  }
  ss.seekg(0, ss.end);
  RawVector retval(ss.tellg());
  ss.seekg(0, ss.beg);
  ss.read(reinterpret_cast<char*>(&retval[0]), retval.size());
  return retval;
}
```

``` cpp
//[[Rcpp::export]]
NumericVector deserializeSNM(RawVector src) {
  std::stringstream ss;
  ss.write(reinterpret_cast<char*>(&src[0]), src.size());
  ss.seekg(0, ss.beg);
  SerializableNumericMatrix snm;
  {
    cereal::BinaryInputArchive iarchive(ss);
    iarchive(snm);
  }
  return snm.numericMatrix();
}
```

``` r
(raw_vector <- serializeSNM(x))
```

    ##  [1] 0a 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 f0
    ## [24] 3f 00 00 00 00 00 00 00 40 00 00 00 00 00 00 08 40 00 00 00 00 00 00
    ## [47] 10 40 00 00 00 00 00 00 14 40 00 00 00 00 00 00 18 40 00 00 00 00 00
    ## [70] 00 1c 40 00 00 00 00 00 00 20 40 00 00 00 00 00 00 22 40 02 00 00 00
    ## [93] 05 00 00 00

``` r
deserializeSNM(raw_vector)
```

|     |     |     |     |     |
|----:|----:|----:|----:|----:|
|    0|    2|    4|    6|    8|
|    1|    3|    5|    7|    9|

### Armadillo Serialization (Not Working)

Basically, we want to be able to serialize Armadillo matrices because then we can serialize things like the Naive Bayes classifier that rely on `arma::Mat`'s.

``` r
# Enables us to include files in src/
registerPlugin("local", function() {
  return(list(env = list(
    PKG_CXXFLAGS = paste0('-I"', getwd(), '/src"')
  )))
})
```

``` cpp
// [[Rcpp::plugins(local)]]
// [[Rcpp::plugins(mlpack11)]]
// [[Rcpp::depends(BH)]]
// [[Rcpp::depends(RcppArmadillo)]]
```

Include everything we'll need for `serialize()`:

``` cpp
#include <boost/serialization/serialization.hpp>
#include <boost/serialization/nvp.hpp>
#include <boost/serialization/array.hpp>
```

In the next chunk we copy and modify the `RcppArmadillo__RcppArmadilloForward__h` definition from **RcppArmadilloForward.h** so as to use our own Mat extra proto and meat, which is really just:

``` cpp
//! Add a serialization operator.
template<typename Archive>
void serialize(Archive& ar, const unsigned int version);

#include <RcppArmadillo/Mat_proto.h>
```

The **Mat\_extra\_bones.hpp** is included for serialization and **Mat\_proto.h** is included so RcppArmadillo plays nice.

``` cpp
#ifndef RcppArmadillo__RcppArmadilloForward__h
#define RcppArmadillo__RcppArmadilloForward__h

#include <RcppCommon.h>
#include <Rconfig.h>
#include <RcppArmadilloConfig.h>

// Costom Mat extension that combines MLPACK with RcppArmadillo's Mat extensions:
#define ARMA_EXTRA_MAT_PROTO mat_extra_bones.hpp
#define ARMA_EXTRA_MAT_MEAT  mat_extra_meat.hpp

// Everything else the same:
#define ARMA_EXTRA_COL_PROTO RcppArmadillo/Col_proto.h
#define ARMA_EXTRA_COL_MEAT  RcppArmadillo/Col_meat.h
#define ARMA_EXTRA_ROW_PROTO RcppArmadillo/Row_proto.h
#define ARMA_EXTRA_ROW_MEAT  RcppArmadillo/Row_meat.h
#define ARMA_RNG_ALT         RcppArmadillo/Alt_R_RNG.h
#include <armadillo>
/* forward declarations */
namespace Rcpp {
    /* support for wrap */
    template <typename T> SEXP wrap ( const arma::Mat<T>& ) ;
    template <typename T> SEXP wrap ( const arma::Row<T>& ) ;
    template <typename T> SEXP wrap ( const arma::Col<T>& ) ;
    template <typename T> SEXP wrap ( const arma::field<T>& ) ;
    template <typename T> SEXP wrap ( const arma::Cube<T>& ) ;
    template <typename T> SEXP wrap ( const arma::subview<T>& ) ;
    template <typename T> SEXP wrap ( const arma::SpMat<T>& ) ;
    
    template <typename T1, typename T2, typename glue_type> 
    SEXP wrap(const arma::Glue<T1, T2, glue_type>& X ) ;
    
    template <typename T1, typename op_type>
    SEXP wrap(const arma::Op<T1, op_type>& X ) ;
    
    template <typename T1, typename T2, typename glue_type> 
    SEXP wrap(const arma::eGlue<T1, T2, glue_type>& X ) ;
    
    template <typename T1, typename op_type>
    SEXP wrap(const arma::eOp<T1, op_type>& X ) ;
    
    template <typename T1, typename op_type>
    SEXP wrap(const arma::OpCube<T1,op_type>& X ) ;
    
    template <typename T1, typename T2, typename glue_type>
    SEXP wrap(const arma::GlueCube<T1,T2,glue_type>& X ) ;
    
    template <typename T1, typename op_type>
    SEXP wrap(const arma::eOpCube<T1,op_type>& X ) ;
    
    template <typename T1, typename T2, typename glue_type>
    SEXP wrap(const arma::eGlueCube<T1,T2,glue_type>& X ) ;

    template<typename out_eT, typename T1, typename op_type>
    SEXP wrap( const arma::mtOp<out_eT,T1,op_type>& X ) ;

    template<typename out_eT, typename T1, typename T2, typename glue_type>
    SEXP wrap( const arma::mtGlue<out_eT,T1,T2,glue_type>& X );
    
    template <typename eT, typename gen_type>
    SEXP wrap( const arma::Gen<eT,gen_type>& X) ;
    
    template<typename eT, typename gen_type>
    SEXP wrap( const arma::GenCube<eT,gen_type>& X) ;
    
    namespace traits {

    /* support for as */
    template <typename T> class Exporter< arma::Mat<T> > ;
    template <typename T> class Exporter< arma::Row<T> > ;
    template <typename T> class Exporter< arma::Col<T> > ;
    template <typename T> class Exporter< arma::SpMat<T> > ;
    
    template <typename T> class Exporter< arma::field<T> > ;
    // template <typename T> class Exporter< arma::Cube<T> > ;

    } // namespace traits 

    template <typename T> class ConstReferenceInputParameter< arma::Mat<T> > ;
    template <typename T> class ReferenceInputParameter< arma::Mat<T> > ;
    template <typename T> class ConstInputParameter< arma::Mat<T> > ;
    
    template <typename T> class ConstReferenceInputParameter< arma::Col<T> > ;
    template <typename T> class ReferenceInputParameter< arma::Col<T> > ;
    template <typename T> class ConstInputParameter< arma::Col<T> > ;
    
    template <typename T> class ConstReferenceInputParameter< arma::Row<T> > ;
    template <typename T> class ReferenceInputParameter< arma::Row<T> > ;
    template <typename T> class ConstInputParameter< arma::Row<T> > ;
    
}

#endif
```

Okay, now that that's been defined, let's include **RcppArmadillo.h** which will include **RcppForward.h** but which will (hopefully) *not* redefine our slightly customized `RcppArmadillo__RcppArmadilloForward__h`.

``` cpp
#include <RcppArmadillo.h>
```

MLPACK's Naive Bayes Classifier:

``` cpp
#include <mlpack/core/util/log.hpp>
#include <mlpack/methods/naive_bayes/naive_bayes_classifier.hpp>
using namespace mlpack::naive_bayes;
```

Boost's Archives:

``` cpp
// Include everything we'll need for archiving.
#include <boost/archive/binary_oarchive.hpp>
#include <boost/archive/binary_iarchive.hpp>
#include <sstream>
```

``` cpp
using namespace Rcpp;
using namespace arma;
```

``` cpp
// [[Rcpp::export]]
mat test() {
  mat A(3, 6, fill::randu);
  return A;
}
```

``` r
set.seed(0); (x1 <- test())
```

|        |        |        |        |        |        |
|-------:|-------:|-------:|-------:|-------:|-------:|
|  0.8967|  0.5729|  0.8984|  0.6291|  0.1766|  0.7698|
|  0.2655|  0.9082|  0.9447|  0.0618|  0.6870|  0.4977|
|  0.3721|  0.2017|  0.6608|  0.2060|  0.3841|  0.7176|

``` r
set.seed(0); x2 <- test()
identical(x1, x2)
```

    ## [1] TRUE

``` cpp
// [[Rcpp::export]]
RawVector test_serialization(Mat<double> m) {
  std::stringstream ss;
  // save data to archive
  {
    boost::archive::binary_oarchive oarchive(ss); // create an output archive
    oarchive & m; // write class instance to archive
    // archive and stream closed when destructors are called
  }
  ss.seekg(0, ss.end);
  RawVector retval(ss.tellg());
  ss.seekg(0, ss.beg);
  ss.read(reinterpret_cast<char*>(&retval[0]), retval.size());
  return retval;
}
```

``` r
# (y1 <- test_serialization(x1))
```

``` cpp
```

References
==========

-   Eddelbuettel, D. (2013). Seamless R and C++ Integration with Rcpp. New York, NY: Springer Science & Business Media. <http://doi.org/10.1007/978-1-4614-6868-4>
-   Wickham, H. A. (2014). Advanced R. Chapman and Hall/CRC. <http://doi.org/10.1201/b17487>
