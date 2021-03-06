---
layout: post
title:  Your code evolved
date: 2016-05-25
output:
  md_document:
    variant: markdown_github
---

Let's do some training and apply some new tools to this code!

## A smarter pokedex

Hadley Wickham has released a package called [rvest](https://cran.r-project.org/web/packages/rvest/index.html) since dsparks initial study, so we'll use that to pull the data.


{% highlight r %}
library(rvest)
library(magrittr)

baseStats <-
  read_html(
    x = "http://bulbapedia.bulbagarden.net/wiki/List_of_Pok%C3%A9mon_by_base_stats_(Generation_I)"
    ) %>%
  html_node(
    css = "div table"
    ) %>%
  html_table()
{% endhighlight %}

The [rvest](https://cran.r-project.org/web/packages/rvest/vignettes/selectorgadget.html) documentation suggests using [SelectorGadget](http://selectorgadget.com/), a web tool for breaking down web pages by css. I did use this initially, but found that the item classes in this case (e.g "div table") was an acceptable argument whereas with the point and click interface the suggestion was in fact ".jquery-tablesorter", however this returned an error when run.

The funny `%>%` operators are 'pipes', initially created in the [magrittr](http://www.r-statistics.com/2014/08/simpler-r-coding-with-pipes-the-present-and-future-of-the-magrittr-package/) package, Hadley uses them extensively in the [Hadleyverse](http://adolfoalvarez.cl/the-hitchhikers-guide-to-the-hadleyverse/) as they prevent nesting functions and have a straightforward left to right syntax.

So, in a three command one liner we've read the data straight through into a workable `data.frame` format. However, [data.tables](https://cran.r-project.org/web/packages/data.table/index.html) are faster to work with, so lets convert it and fix up those headers.


{% highlight r %}
library(data.table)

baseStats <- as.data.table(baseStats)

baseStats[, "" := NULL] # Remove second col with only "NA"
setnames(baseStats, 1:2, c("DexNo", "Pokemon")) # Rename cols 1 and 2 to something workable
{% endhighlight %}

The first line uses the `data.table` native assignment operator `:=` to assign a value of `NULL`, effectively deleting it. `setnames` is one of the many `set*` family functions which assign in place, without copying a new object, modifying the copy in RAM, and deleting the original. This is more memory efficient and faster. Not necessarily needed for data at this scale, but doesn't hurt either.

## A better pokeball

The method to grab the urls for each image was pretty complex before. This method refines two areas.

* Has no dependency on custom gists to modify strings (neat though they are)
* More efficiently identifies urls by using `html_nodes` and not needing to scrape the full webpage transcript


{% highlight r %}
library(stringr)

baseStats[, imgURL := read_html(
    x = "http://bulbapedia.bulbagarden.net/wiki/List_of_Pok%C3%A9mon_by_base_stats_(Generation_I)"
    ) %>% 
  html_nodes(
      css = "#mw-content-text img"
      ) %>% # css to identify all strings for pokemon urls
  str_split_fixed(
    "src=\"", 
    n = 2
    ) %>% .[,2] %>% # splits first part out then drops first part
  str_split_fixed(
    "\" width=", 
    n = 2) %>% .[,1]]
{% endhighlight %}

This code also uses the `stringr` package to split the strings down at fixed points, returning concise urls via `string_split_fixed` which looks for specific characters and chops them into 2 values at that point. By calling all of this within the data.table we just assign it into each row in order.

To grab the actual pngs I initially played around a lot with calling `readPNG(getURLContent(...))` in the data.table  in a form like: `baseStats[, readPNG(getURLContent(imgURL))]`, . However after many failures I realised that data.table wasn't playing with `getURLContent()` well, as it needed to open a different connection for each Pokemon. So I've left this part out. If anyone has any suggestions please get in touch (on Github I guess, I haven't got commenting up on here yet >.> <.<;)

## Gotta plot 'em all

To create the PCA model we can call it from our new data.table


{% highlight r %}
pokemonPCA <- prcomp(baseStats[, .(HP, Attack, Defense, Speed, Special)])
{% endhighlight %}

Outside of the base plots a more pleasant graphing package is available for biplots based on the ggplot approach


{% highlight r %}
# devtools::install_github("vqv/ggbiplot")

library(ggbiplot)

ggbiplot(pokemonPCA, labels = baseStats$Pokemon)
{% endhighlight %}

![plot of chunk ggbiplot]({{ site.github.url  }}/assets/Rfig/ggbiplot-1.svg)

This functions actually has a number of neat 'ggplot-like' features. One of which is grouping by factor. The most obvious factors in pokemon are their type classifications, so lets get that data.


{% highlight r %}
pageTables <- 
  read_html(
    x = "http://bulbapedia.bulbagarden.net/wiki/List_of_Pok%C3%A9mon_by_Kanto_Pok%C3%A9dex_number"
    ) %>% 
  html_nodes(
    css = "div table"
    )

pokeType <- html_table(pageTables[c(2,3,4)]) %>%
  rbindlist()

setnames(pokeType, c(4,6), c("Pokemon", "Type2"))
pokeType[, DualType := paste(Type, Type2)]

baseStats <- merge(baseStats, pokeType[, .(Pokemon, Type, Type2, DualType)], by = "Pokemon", sort = FALSE)
{% endhighlight %}

Here I use the same httr approach as before, but allow multiple nodes to be returned, giving me 7 "div table" nodes. I then subset down to the useful ones within `html_table`, which gives me a list of 3 data.frames, and collapse them all down to one with a call to data.tables `rbindlist`. This is then subset and merged back onto baseStats.


{% highlight r %}
ggbiplot(pokemonPCA, labels = baseStats$Pokemon, groups = baseStats$Type) +
  ggtitle("Pokemon Gen1 PCA by Primary Type")
{% endhighlight %}

![plot of chunk ggbiplot2]({{ site.github.url  }}/assets/Rfig/ggbiplot2-1.svg)

{% highlight r %}
ggbiplot(pokemonPCA, labels = baseStats$Pokemon, groups = baseStats$Type2) +
  ggtitle("Pokemon Gen1 PCA by Secondary Type")
{% endhighlight %}

![plot of chunk ggbiplot2]({{ site.github.url  }}/assets/Rfig/ggbiplot2-2.svg)

{% highlight r %}
ggbiplot(pokemonPCA, labels = baseStats$Pokemon, groups = baseStats$DualType) +
  ggtitle("Pokemon Gen1 PCA by Combined Type")
{% endhighlight %}

![plot of chunk ggbiplot2]({{ site.github.url  }}/assets/Rfig/ggbiplot2-3.svg)

So as I noted before, the Psychic types and ghost types do sit out at the edge of speed and special. Similarly the Rock and Fighting types are clustered at the other edge of the graph. Electric types also push towards the top of the chart due to their Special and Speed stats, but so do other speedy normals, special water attackers, and special bug types.

However, as my friends Paul and Steph pointed out to me last night, PCA is probably the wrong tool for this. We're not seeing the kind of clustering here that would denote groups within the data set that would remain even after the dimensions are reduced.


{% highlight r %}
screeplot(pokemonPCA)
{% endhighlight %}

![plot of chunk scree plot]({{ site.github.url  }}/assets/Rfig/scree plot-1.svg)

This plot is sometimes referred to as a scree plot, and indicates the successive strength of each of the variables supplied to the PCA. If some of these could be discarded, there would be an obvious drop off where the values on the left were significantly larger than the values on the right([like this](http://fabian-kostadinov.github.io/2015/05/31/pca-in-r/)). This seems to indicate that there is no useful dimensional reduction that can be done on this data.

This would make sense though, as Pokemon is a purposefully designed game by humans, and also theoretically 'balanced' no one group of pokemon (should) be more useful than any other. However, this is only part of the picture. Evolutionary lines and typing are not considered here. Yet...
