---
layout: post
title: "Mining the screenplay of The Room"
date: "January 25, 2018"
type: post
published: true
status: publish
tags:
- rstats
- text mining
- the room
- disaster artist
---


<p style="margin-top:60px">While sitting in the office, my colleague <a href="https://www.wiso.uni-hamburg.de/fachbereich-sowi/professuren/schnapp/team/dietrich-brian.html">Brian</a> and I stumbled upon this dead-on <a href="https://www.youtube.com/watch?v=Z9cB0TjfIkM">honest trailer</a> of a movie called <em>The Room</em> a.k.a. one of the worst movies of all time (written, produced, directed by and starring Tommy Wiseau).</p>
  
And since the movie features such Shakespearian quotes as *"I did not hit her, it's not true! It's bullshit! I did not hit her! I did naaht! Oh hi, Mark"*, we joked about mining the screenplay for other hidden gems.

Well, no sooner said than done.
 
## Setup

Conducting the analysis below requires the packages `dplyr`, `ggplot2`, `magrittr`, `pdftools`, `reshape2`, `stringr`, 
`tidytext`, and `wordcloud`.

For loading multiple packages at once, I recommend `p_load()` from the `pacman` package, which is a wrapper function for `library()` and `require()` and installs missing packages if necessary.

``` r
# Install and load pacman if not already installed
if (!require("pacman")) install.packages("pacman")
library(pacman)

# Load packages
p_load(dplyr, ggplot2, magrittr, pdftools, reshape2, stringr, tidytext, wordcloud)
```

## Importing data

The full movie script can be found <a href="https://theroomscriptblog.files.wordpress.com/2016/04/the-room-original-script-by-tommy-wiseau.pdf">here</a>. To download and import it to `R`, simply run

```r
# Download pdf
download.file("https://theroomscriptblog.files.wordpress.com/2016/04/the-room-original-script-by-tommy-wiseau.pdf",
              "the-room-original-script-by-tommy-wiseau.pdf")
              
# Extract text from pdf file
room <- pdf_text("the-room-original-script-by-tommy-wiseau.pdf")
```

to extract the text via `pdf_text()` from the package `pdftools`. 

## Text cleaning

After extracting the raw text from The Room's `pdf` screenplay, it needs some cleaning prior to analyzing. 

Concretely, this means separating the lines of the raw text (`\n` indicating line breaks), removing redundant text parts such as the cover page, headers and footers, blank lines, and directing instructions as well as punctuation (except for apostrophes), non-alphabetic characters, and stopwords. (Note that I didn't stem words.)

For most of these steps, `lapply()` can be used to apply the respective function to each element of the list. 

While performing these steps, the cleaned text, consisting of a sequence of strings, is splitted into single words -- a process called *tokenization*.

```r
# Separate lines with \n indicating line breaks
room_tidy <- strsplit(room, "\n")

# Remove cover page
room_tidy <- room_tidy[-1]

# Remove page numbers and headers
room_tidy <- lapply(room_tidy, function(x) x[-(1:2)])

# Remove footers
room_tidy <- lapply(room_tidy, function(x) x[1:(length(x)-2)])

# Remove information on act and scene
room_tidy <- lapply(room_tidy, function(x) gsub("END SCENE", "", x))
room_tidy <- lapply(room_tidy, function(x) gsub("ACT.*", "", x))
room_tidy <- lapply(room_tidy, function(x) gsub("SCENE.*", "", x))

# Remove punctuation (except for apostrophes) and numbers
room_tidy <- lapply(room_tidy, function(x) gsub("[^[:alpha:][:blank:]']", "", x))

# Remove directing instructions
room_tidy <- lapply(room_tidy, function(x) x[!grepl("^[A-Z ']+$", x), drop = FALSE])

# Convert to lowercase
room_tidy <- lapply(room_tidy, function(x) tolower(x))

# Split strings
room_tidy <- lapply(room_tidy, function(x) strsplit(x, " "))

# Turn list to data.frame 
room_df <- data.frame(matrix(unlist(room_tidy), nrow = 10357, byrow = T), stringsAsFactors = FALSE)

# Remove introductory part and last two lines
room_df <- tail(room_df, -102)
room_df <- head(room_df, -2)

# Rename column to match anti_join(stopwords)
colnames(room_df) <- "word"

# Remove blank lines from text
room_df %<>% filter(word != "") 

# Remove stopwords
room_df %<>% anti_join(stop_words)
```

## Visualizing most common words

After processing the raw text and turning it into a tidy format, we can extract the most common words from the movie script by counting their appearances within the script via `count()` from the `plyr` package and plot them by running the following code

```r
# Find most common words
room_df_wordfreq <- room_df %>%
  count(word, sort = TRUE)

# Plot words
room_df %>%
  count(word, sort = TRUE) %>%
  filter(n > 20) %>%
  mutate(word = reorder(word, n)) %>%
  ggplot(aes(word, n)) +
  geom_col() +
  xlab("") + ylab("") + ggtitle("Most common words in The Room", subtitle = "Written by Tommy Wiseau") +
  ylim(0, 100) + coord_flip()
```  

which creates this graph:

<p align="center"><img src="https://raw.githubusercontent.com/lhehnke/lhehnke.github.io/master/img/text-mining-room/plot1.png" width="550px" height="500px" vspace="50px"/></p>

It comes as little surprise that the most common words in the movie script are of a similar quality to the quote in the beginning of this post with the most frequently used words being "Johnny", "Lisa", and "Mark" -- the main characters' names -- and more elaborate expressions such as "yeah" or "ha" (sadly, *naaht* didn't make it to the list). 

## Sentiment analysis

In a similar manner, we can conduct a basic sentiment analysis and visualize the results. For this purpose we will use the `get_sentiments()` functions from the `tidytext` package and use both the `NRC` Emotion Lexicon from Saif Mohammad and Peter Turney (all sentiments) and the sentiment lexicon from `Bing` Liu and collaborators (positive/negative sentiments).

```r
# Get sentiments (nrc) and join
nrc <- get_sentiments("nrc") 
room_nrc <- room_df %>%
  inner_join(nrc) %>%
  count(word, sort = TRUE)

# Calculate total sentiment scores
nrc_counts <- data.frame(table(nrc$sentiment))

# Plot sentiment scores
ggplot(data = nrc_counts, aes(x = Var1, y = Freq)) +
  geom_bar(aes(fill = Var1), stat = "identity") +
  xlab("") + ylab("") + ggtitle("Total sentiment scores in The Room", subtitle = "Written by Tommy Wiseau") +
  ylim(0, 4000) + theme(legend.position = "none") 
```

yielding the following plot:

<p align="center"><img src="https://raw.githubusercontent.com/lhehnke/lhehnke.github.io/master/img/text-mining-room/plot2.png" width="550px" height="500px" vspace="50px"/></p>

We can see in the `NRC` sentiments plot that most words in The Room's screenplay are negatively scored, followed by positively scored words.

## Positive and negative words

By calculating `Bing` sentiment scores next, we can explore the previous finding further.

```r
# Calculate contributions to positive and negative sentiments (bing) by word 
bing_counts <- room_df %>%
  inner_join(get_sentiments("bing")) %>%
  count(word, sentiment, sort = TRUE) %>%
  ungroup()

# Calculate top word contributors
bing_counts_plot <- bing_counts %>%
  group_by(sentiment) %>%
  top_n(10) %>%
  ungroup() %>%
  mutate(word = reorder(word, n)) 

# Plot most common positive and negative words
ggplot(bing_counts_plot, aes(word, n, fill = sentiment)) +
  geom_col(show.legend = FALSE) +
  facet_wrap(~sentiment, scales = "free_y") +
  xlab("") + ylab("") + 
  ggtitle("Most common positive and negative words in The Room", subtitle = "Written by Tommy Wiseau") +
  coord_flip()
  ```
  
<p align="center"><img src="https://raw.githubusercontent.com/lhehnke/lhehnke.github.io/master/img/text-mining-room/plot3.png" width="550px" height="500px" vspace="50px"/></p>

As the second sentiment graph shows, the most common positive words in The Room are "love", "happy", and "fine", while the most common negative words are "worry", "crazy", and "wrong". 
  
## Word clouds

To finish up, we finally plot one of the infamous word clouds (albeit in a slightly more advanced version) by constrasting the most common positive and negative words.  
  
```r
# Plot comparison cloud in ggplot2 colors
## Run > unique(g$data[[1]]["fill"]) after ggplot_build() to extract colors
room_df %>%
  inner_join(get_sentiments("bing")) %>%
  count(word, sentiment, sort = TRUE) %>%
  acast(word ~ sentiment, value.var = "n", fill = 0) %>%
  comparison.cloud(colors = c("#F8766D", "#00BFC4"), max.words = 60)
```

<p align="center"><img src="https://raw.githubusercontent.com/lhehnke/lhehnke.github.io/master/img/text-mining-room/plot4.png" width="550px" height="500px" vspace="50px"/></p>

Wrapping it up, the findings of this analysis can be summarized as follows:<br>
<p align="center"> :heart: :grin: :relieved: :worried: :angry: :x: </p>
