RNN made easy with MXNet R
================

> This tutorial presents an example of application of one-to-one RNN applied to text generation using the reworked functionnalities in MXNet R package. Requires to train on GPU with CUDA.

Example based on [Obama's speech](http://data.mxnet.io/data/char_lstm.zip).

Load some packages

``` r
require("readr")
require("dplyr")
require("plotly")
require("stringr")
require("stringi")
require("scales")
require("mxnet")
```

Load utility functions

``` r
source("mx.io.bucket.iter.nomask.R")
source("rnn.graph.R")
source("model.rnn.R")
source("rnn.infer.R")
```

Data preparation
----------------

Data preparation is performed by the script: `data_preprocessing_one_to_one.R`.

The following steps are executed:

-   Import speach data as a single character string.
-   Remove non printable characters.
-   Split text into individual characters.
-   Group characters into sequences of a fixed length, each sequence being a sample to the model.

``` r
corpus_bucketed_train <- readRDS(file = "data/train_buckets_one_to_one.rds")
corpus_bucketed_test <- readRDS(file = "data/eval_buckets_one_to_one.rds")

vocab <- length(corpus_bucketed_test$dic)

### Create iterators
batch.size = 32

train.data <- mx.io.bucket.iter(buckets = corpus_bucketed_train$buckets, batch.size = batch.size, 
                                data.mask.element = 0, shuffle = TRUE)

eval.data <- mx.io.bucket.iter(buckets = corpus_bucketed_test$buckets, batch.size = batch.size,
                               data.mask.element = 0, shuffle = FALSE)
```

Model arhcitecture
------------------

A one-to-one model configuration is specified since for each character, we want to predict the next one. For a sequence of length 100, there are also 100 labels, corresponding the same sequence of characters but offset by a position of +1.

The parameters `output_last_state` is set to `TRUE` in order to access the state of the RNN cells when performing inference.

``` r
rnn_graph_one_one <- rnn.graph(num.rnn.layer = 2, 
                               num.hidden = 80,
                               input.size=vocab+1,
                               num.embed=50, 
                               num.label=vocab,
                               dropout=0.5, 
                               ignore_label = 0,
                               cell.type="lstm",
                               masking = F,
                               output_last_state = T,
                               config = "one-to-one")

graph.viz(rnn_graph_one_one, type = "graph", direction = "LR", 
          graph.height.px = 100, graph.width.px = 800, shape=c(100, 64))
```

![](README_one_to_one_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-4-1.png)

Fit a LSTM model
----------------

``` r
devices <- mx.gpu(0)

initializer <- mx.init.Xavier(rnd_type = "gaussian", factor_type = "avg", magnitude = 4)

optimizer <- mx.opt.create("adadelta", rho = 0.90, eps = 1e-5, wd = 1e-8,
                           clip_gradient = 5, rescale.grad = 1/batch.size)

# optimizer <- mx.opt.create("adam", learning.rate = 0.001, beta1 = 0.9, beta2 = 0.999, epsilon = 1e-8, wd = 1e-5, rescale.grad = 1/batch.size, clip_gradient = 1)

# optimizer <- mx.opt.create("rmsprop", learning.rate = 0.002, gamma1 = 0.95, gamma2 = 0.95,
#                            wd = 1e-8, clip_gradient=NULL, rescale.grad=1/batch.size)

logger <- mx.metric.logger()
epoch.end.callback <- mx.callback.log.train.metric(period = 1, logger = logger)
batch.end.callback <- mx.callback.log.train.metric(period = 50)

mx.metric.custom_nd <- function(name, feval) {
  init <- function() {
    c(0, 0)
  }
  update <- function(label, pred, state) {
    m <- feval(label, pred)
    state <- c(state[[1]] + 1, state[[2]] + m)
    return(state)
  }
  get <- function(state) {
    list(name=name, value=(state[[2]]/state[[1]]))
  }
  ret <- (list(init=init, update=update, get=get))
  class(ret) <- "mx.metric"
  return(ret)
}

mx.metric.Perplexity <- mx.metric.custom_nd("Perplexity", function(label, pred) {
  label_probs <- as.array(mx.nd.choose.element.0index(pred, label))
  batch <- length(label_probs)
  NLL <- -sum(log(pmax(1e-15, as.array(label_probs)))) / batch
  Perplexity <- exp(NLL)
  return(Perplexity)
})

model <- mx.rnn.buckets(symbol = rnn_graph_one_one,
                        train.data = train.data, eval.data = eval.data,
                        num.round = 40, ctx = devices, verbose = TRUE,
                        metric = mx.metric.Perplexity, 
                        initializer = initializer, optimizer = optimizer, 
                        batch.end.callback = batch.end.callback, 
                        epoch.end.callback = epoch.end.callback)

mx.model.save(model, prefix = "models/model_one_to_one_lstm", iteration = 40)

p <- plot_ly(x = seq_len(length(logger$train)), y = logger$train, type = "scatter", mode = "markers+lines", name = "train") %>% 
  add_trace(y = logger$eval, type = "scatter", mode = "markers+lines", name = "eval")

plotly::export(p, file = "logger_one_to_one.png")
```

![](logger_one_to_one.png)

Inference on test data
----------------------

Setup inference data. Need to apply preprocessing to inference sequence and convert into a infer data iterator.

``` r
ctx <- mx.gpu(0)
batch.size <- 1

corpus_bucketed_train <- readRDS(file = "data/train_buckets_one_to_one.rds")
dic <- corpus_bucketed_train$dic
rev_dic <- corpus_bucketed_train$rev_dic

infer_raw <- c("The United States are")
infer_split <- dic[strsplit(infer_raw, '') %>% unlist]
infer_length <- length(infer_split)

infer_buckets <- list("21"=list(data=matrix(infer_split, ncol=1), 
                                label=matrix(infer_split, ncol=1)))
infer_buckets <- list(buckets = infer_buckets, dic = dic, rev_dic = rev_dic)

infer.data <- mx.io.bucket.iter(buckets = infer_buckets$buckets, batch.size = 1, 
                                data.mask.element = 0, shuffle = FALSE)
```

### Inference with most likely term

Here the predictions are performed by picking the character whose associated probablility is the highest.

``` r
model <- mx.model.load(prefix = "models/model_one_to_one_lstm", iteration = 40)

internals <- model$symbol$get.internals()

sym_state <- internals$get.output(which(internals$outputs %in% "lstm_2_layer_state"))
sym_state_cell <- internals$get.output(which(internals$outputs %in% "lstm_2_layer_state_cell"))
sym_output <- internals$get.output(which(internals$outputs %in% "loss_output"))
symbol <- mx.symbol.Group(sym_output, sym_state, sym_state_cell)

predict <- numeric()

infer <- mx.rnn.infer.buckets.one(infer.data = infer.data, 
                                  symbol = symbol,
                                  arg.params = model$arg.params,
                                  aux.params = model$aux.params,
                                  input.params = NULL, 
                                  ctx = ctx)

pred_prob <- mx.nd.slice.axis(infer$loss_output, axis=0, begin = infer_length-1, end = infer_length)
pred <- mx.nd.argmax(data = pred_prob, axis = 1, keepdims = T)
predict <- c(predict, as.numeric(as.array(pred)))

for (i in 1:100) {
  
  infer_buckets <- list("1"=list(data=as.matrix(pred), label=as.matrix(pred)))
  infer_buckets <- list(buckets = infer_buckets, dic = dic, rev_dic = rev_dic)
  infer.data <- mx.io.bucket.iter(buckets = infer_buckets$buckets, batch.size = 1, 
                                  data.mask.element = 0, shuffle = FALSE)
  
  infer <- mx.rnn.infer.buckets.one(infer.data = infer.data, 
                                    symbol = symbol,
                                    arg.params = model$arg.params,
                                    aux.params = model$aux.params,
                                    input.params = list(rnn.state = infer[[2]], 
                                                        rnn.state.cell = infer[[3]]), 
                                    ctx = ctx)
  
  pred <- mx.nd.argmax(data = infer$loss_output, axis = 1, keepdims = T)
  predict <- c(predict, as.numeric(as.array(pred)))
  
}

predict_txt <- paste0(rev_dic[as.character(predict)], collapse = "")
predict_txt_tot <- paste0(infer_raw, predict_txt, collapse = "")
```

Generated sequence: The United States are the progress the progress the progress the progress the progress the progress the progress the progr

Key ideas appear somewhat overemphasized.

### Inference from random sample

Noise is now inserted in the predictions by sampling each character based on their modeled probability.

``` r
set.seed(4)

infer_raw <- c("The United States are")
infer_split <- dic[strsplit(infer_raw, '') %>% unlist]
infer_length <- length(infer_split)

infer_buckets <- list("21"=list(data=matrix(infer_split, ncol=1), 
                                label=matrix(infer_split, ncol=1)))
infer_buckets <- list(buckets = infer_buckets, dic = dic, rev_dic = rev_dic)

infer.data <- mx.io.bucket.iter(buckets = infer_buckets$buckets, batch.size = 1, 
                                data.mask.element = 0, shuffle = FALSE)

predict <- numeric()

infer <- mx.rnn.infer.buckets.one(infer.data = infer.data, 
                                  symbol = symbol,
                                  arg.params = model$arg.params,
                                  aux.params = model$aux.params,
                                  input.params = NULL, 
                                  ctx = ctx)

pred_prob <- as.numeric(as.array(mx.nd.slice.axis(
  infer$loss_output, axis=0, begin = infer_length-1, end = infer_length)))
pred <- sample(length(pred_prob), prob = pred_prob, size = 1) - 1
predict <- c(predict, pred)

for (i in 1:100) {
  
  infer_buckets <- list("1"=list(data=as.matrix(pred), label=as.matrix(pred)))
  infer_buckets <- list(buckets = infer_buckets, dic = dic, rev_dic = rev_dic)
  infer.data <- mx.io.bucket.iter(buckets = infer_buckets$buckets, batch.size = 1, 
                                  data.mask.element = 0, shuffle = FALSE)
  
  infer <- mx.rnn.infer.buckets.one(infer.data = infer.data, 
                                    symbol = symbol,
                                    arg.params = model$arg.params,
                                    aux.params = model$aux.params,
                                    input.params = list(rnn.state = infer[[2]], 
                                                        rnn.state.cell = infer[[3]]), 
                                    ctx = ctx)
  
  pred_prob <- as.numeric(as.array(infer$loss_output))
  pred <- sample(length(pred_prob), prob = pred_prob, size = 1, replace = T) - 1
  predict <- c(predict, pred)
}

predict_txt <- paste0(rev_dic[as.character(predict)], collapse = "")
predict_txt_tot <- paste0(infer_raw, predict_txt, collapse = "")
```

Generated sequence: The United States are their numberal year Doi't Movementate Curplomy.They'll good togethering at the just through their mo

Now we get a more alembicated political speech.