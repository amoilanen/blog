---
layout: default
title: "Measuring Language Popularity Is Harder Than Many Think"
date: 2013-02-17
---

# Contents

[What languages are most popular?](#what_languages)
[Measurement method](#measurement_method)
[Technical details](#techical_details)
[Comparing with other measurements](#comparing_with_other_results)
[No comprehensive research anywhere](#no_comprehensive_research_anywhere)

<a name="what_languages"></a>

## What languages are most popular?

Obviously it is interesting to know which programming languages are most popular. If some language is popular, this means that you will have plenty of resources such as books and articles to learn it, also, probably, good community support and you will be able to find a lot of ready libraries and tools for that language as well just because so many other developers are already using it. Also it may be interesting to see how our most popular programming language, which we all tend to have once in a while, scores relative to other languages.

<a name="measurement_method"></a>

## Measurement method

But how to measure the language popularity on the Web? The first thing that came to mind was to use different search engines and compare numbers of results for different languages. This seemed like the simplest and the most obvious thing to do. But not so fast! Unfortunately, it turns out, that search counts returned by most popular search engines are just rough estimates of the number of the results they *think* they should give to you based on your search query, not *all* the possible results. More details are explained in this [article](http://searchengineland.com/why-google-cant-count-results-properly-53559). In other words, search engines are doing well what they are supposed to do: context based search. Nobody designed them to compute exact aggregations over huge amounts of data and they do not usually do this well.

What other options can be there? For once, we can select a few sites with job postings, such as [monster.com](http://www.monster.com), or with software development articles and presentations like [infoq.com](http://infoq.com), various forums for software engineers, etc. On these sites we can search for certain programming languages, and by comparing the results we can estimate the relative popularity of the languages.

However, searching just one such resource may not be enough, for example, Java developers can really like one site and Ruby developers at the same time can like completely another site. As later we will see this is actually the case with [github.com](http://www.github.com), which is really popular with JavaScript developers and with [stackoverflow.com](http://www.stackoverflow.com), which has a large number of C# related questions. But at least we can try to search one of such sites and compare the results with the data we already have from other sources to be more sure in our measurements.

I chose [stackoverflow.com](http://www.stackoverflow.com) as it is a really good and popular site with questions and answers on every software development topic you can think about.

<a name="techical_details"></a>

## Technical details

So, now I will take the list of all the programming languages from somewhere and search for them on [stackoverflow.com](http://stackoverflow.com). Let's take, for example, the list of all the languages that are used on [github.com](http://github.com). Then we would have to search for each language and write down the number of search results for each one of them. But since it is a really boring and time consuming task and computers were invented a long time ago, let's write a simple script that will help us do the mundane work  and execute around 90 searches. By automating a bit we will also have more confidence in the results as manually doing something is usually more error-prone.

For automation we will use a headless WebKit browser [PhantomJS](http://phantomjs.org/) and will generate an HTML report right from our [PhantomJS](http://phantomjs.org/) script. The result will be just a simple bar chart rendered with [Google Chart Tools](https://developers.google.com/chart/).

The complete version of the code for the script is available [here](https://github.com/antivanov/misc/blob/master/JavaScript/top_languages.js).

Some of the code highlights from the key parts of the script are listed below.

Getting the list of all the languages from [github.com](http://www.github.com)

```javascript
function getAllGithubLanguages(callback) {
    page.open("https://github.com/languages", function (status) {
        var allLanguages = page.evaluate(function() {
            var links = document.querySelectorAll(".all_languages a");
            return Array.prototype.slice.call(links, 0).map(function(link) {
                return link.innerHTML;
            });
        });
        callback(allLanguages);
    });
};
```

By the way, notice how easy it is to work with DOM in JavaScript: all the API specifically adapted for this is already there, so PhantomJS allows us to use **querySelectorAll** and CSS selectors.

Getting the number of results once they are displayed in the browser.

```javascript
function getSummaryCount() {
    var resultStats = document.querySelector("div.summarycount"),
        regex = /[\d,.]+/g,
        resultsNumber = -1;

    if (resultStats) {
        resultsNumber = regex.exec(resultStats.innerHTML)[0];
        resultsNumber = resultsNumber.replace(/[,\.]/g, "");
    };
    return parseInt(resultsNumber);
};
```

Searching for each language with two URLs in case the first URL produces no results.

```javascript
function openResultsURL(url, callback) {
    page.open(url, function (status) {
        callback(page.evaluate(getSummaryCount));
    });
};

function search(term, callback) {
    var urls = [
        "http://stackoverflow.com/search?q=" + encodeURIComponent(term),
        "http://stackoverflow.com/tags/" + encodeURIComponent(term)
    ];

    openResultsURL(urls[0], function(resultsCount) {
        if (resultsCount > 0) {
            callback(term, resultsCount);
        } else {
            openResultsURL(urls[1], function(resultsCount) {
                callback(term, resultsCount);
            });
        }
    });
};
```

Also you may notice how we pass callbacks everywhere. This may seem a bit strange at first if you have not programmed in JavaScript a lot, but this is actually the most common style of programming in JavaScript both on the client and the server side. Here PhantomJS encourages asynchronous programming as well because interaction with the browser is also asynchronous. Each callback is executed once the results are ready at an appropriate point in time. This provides for a more declarative style of programming too.

The entry point into the script, collecting all the search results and saving a generated report.

```javascript
function saveReport(html) {
    fs.write("top_languages_report.html", html, "w");
};

getAllGithubLanguages(function (languages) {
    var statistics = {},
        int;

    languages = Array.prototype.slice.call(languages, 0);
    console.log("Number of languages = ", languages.length);
    int = setInterval(function waitForStatistics() {
        if (0 == activeSearches) {
            if (languages.length > 0) {
                activeSearches++;
                search(languages.shift(), function (term, count) {
                    console.log(term + " found " + count + " times");
                    statistics[term] = count;
                    activeSearches--;
                });
            } else {
                console.log("Finished all searches!");
                clearInterval(int);
                saveReport(generateReport(statistics));
                phantom.exit();
            };
        };
    }, TIMEOUT);
});
```

The code for generating reports that is also in the script is a bit awkward, largely due to the lack of a standard JavaScript library for working efficiently with data structures. It takes quite a bit of effort to transform the results to the format needed for rendering the chart, but the code for doing this is not really what this script is all about, it is just some utility boiler plate that unfortunately cannot be avoided here.So let's just omit this part.

And, voila the final chart with the top 30 most popular on [stackoverflow.com](http://stackoverflow.com) programming languages.

![](http://smthngsmwhr.files.wordpress.com/2012/11/top_languages_chart1.png)

**However, we cannot reach any conclusions based just on the results from one site. Hardly C# is the most popular language, this must be a stackoverflow.com thing.**

We can go one step further and search some other sites like [Amazon.com](http://www.amazon.com) but will it give us any more confidence? Let's stop at this point and compare our results with the results of other similar researches obtained by using slightly different methods.

<a name="comparing_with_other_results"></a>

## Comparing with other results

So the top ten languages we got are: C#, Java, PHP, JavaScript, C++, Python, Objective-C, C, Ruby, and Perl.

First, let's look at [TIOBE](http://www.tiobe.com/index.php/content/paperinfo/tpci/index.html) which uses search engine result counts and we discussed above why it may be not the best idea. It is basically the same 10 languages, but instead of JavaScript it is Visual Basic. Well, maybe Visual Basic has a very lively online community? I doubt it somehow, probably, it is just a lot of Visual Basic books and articles that make it so popular in this index, but everything is possible.

OK, what about the number of projects in different languages at [github.com](http://www.github.com)? The following [statistics](https://github.com/languages) is provided on the site. Also very close to the list that we obtained but instead of C# there is Shell which, probably, can be explained by a lot of people with Linux background who use github.com. It also seems that C# developers do not favor [github.com](http://www.github.com) for some reason.

I would say we have a good correlation between the results for the top languages. Still I will be very careful with saying how much the top languages are popular relative to each other since different sources yielded very different results. But, at least, we get the following 12 most popular at the moment programming languages:

**C#, Java, PHP, JavaScript, C++, Python, Objective-C, C, Ruby, Perl, Shell, Visual Basic**

<a name="no_comprehensive_research_anywhere"></a>

## No comprehensive research anywhere

The problem with measuring the popularity of programming languages seems to be more complex than we initially thought. Developers of a specific language tend to be concentrated around a few sites and different sites give very different results like stackoverflow.com and github.com in the present article. In a way the problem is similar to measuring the popularity of human languages by randomly going to different large world cities. After visiting a couple dozen of cities we may start to get an idea which human languages are popular, but we will have hard time measuring the relative popularity of these languages, and especially hard time with less popular languages: we may just not visit a single large city where those languages are spoken. So, in order to make a good research we should statistically compare results from many different cities (sites), and know the structure of the world (Web) well. Unfortunately, I could not find any such research on the Web and doing it myself requires much more effort than just a weekend project. But even then, is the language popularity only limited to its online popularity?

## Links

[Git-based collaboration in the cloud github.com](http://www.github.com)
[Software development Q&A stackoverflow.com](http://stackoverflow.com)
[PhantomJS](http://phantomjs.org/)
[Google Chart Tools](https://developers.google.com/chart)
[Google Chart Tools: Bar Chart](https://developers.google.com/chart/interactive/docs/gallery/barchart)
[Count results](http://searchengineland.com/why-google-cant-count-results-properly-53559)
[Github.com top languages](https://github.com/languages)
[TIOBE programming community index](http://www.tiobe.com/index.php/content/paperinfo/tpci/index.html)
