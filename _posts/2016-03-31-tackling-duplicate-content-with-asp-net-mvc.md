---
title: Tackling duplicate content with ASP.NET MVC
date: 2016-03-31 00:40:25
categories: [MVC, .NET, Nuget]
tags: [SEO, Nuget, DuplicateContent, C#]
---
**TLDR;** I have researched the duplicate content issue and created a NuGet package that has some flexibility and could solve the problem in a much more elegant way than most other solutions. Link to Github: <https://github.com/dipunm/SEO.DuplicateContent>

_**Disclaimer:** I apologise in advance for using the american spelling of canoncalize._

## The duplicate content problem
Recently, I was tasked with removing duplicate URL's from our website. The initial spec was to force 301 (permanent) redirects from uppercase URL's to lowercase as well as using permanent redirects for removing trailing slashes. The problem with a blanket rule like this, is that it can be limiting, and it doesn't actually solve the whole problem.
### So what is the problem?
When search engines are crawling your website, they are ranking your pages, attributing keywords to them, and these algorithms determine where your webpages end up when a user searches for anything related to your site. Search engines can't easily make assumptions about your pages and their URL's because different websites, using different frameworks can behave differently when considering a URL variation. Therefore, whenever a search engine lands on a page multiple times with slightly varied URL's, it may either penalise your score, or spread your score thin across multiple page entries.

According to RFC 3986, the scheme and host of a URI should be lowercase, but the rest of the URI is considered case sensitive. This means any change to your path, or querystring is technically a duplicate URI; a few examples are: consecutive slashes, trailing slashes, extra querystring parameters, and different querystring keys/values.

## ASP.NET MVC isn't helping
ASP.NET MVC makes a few assumptions when matching URL's to routes, the most commonly known are:
- URL paths are case insensitive
- Trailing slashes are acceptable
- Querystring keys are case insensitive
- Routes with default values can match even if the URL requested explicitly contains those default values.

There are more assumptions that are not so obvious, for example:
- Multiple concurrent forward slahes are acceptable
- Querystring order is not important

I'm being a little unfair with the title; the rules above were made for the convenience of your users and are very sensible rules to live by day to day. They just don't sit well with search engines.

## Handling duplicate content
When attempting to handle duplicate content, you need to tackle two questions:
- What is my canonical URL?
- How will I handle requests that are not purely canonical.

The first question can be tricky, url case and trailing slashes are not the only things to tackle here:
- `http://my-site.com/Home/Index` and `http://my-site.com/` could point to the same page with the same content.
    - You need to choose **one** as your _canonical URL_. Most developers would choose the shorter URL.
- If a querystring parameter can affect the content of your site dramatically, then it should be considered part of your canonical URL
    - eg: `http://my-site.com/product-details?id=2093`.
    - Typically you might solve this by embedding these values within the URL path when designing your routes, but sometimes it is difficult to do, or change, and sometimes it just isn't feasible.
- The casing of querystring keys and values will cause duplicate content as well as the order of parameters. The following examples represent duplicate content. eg:
    - `http://my-site.com/product-details?ID=2093&amp;CustomerID=3`
    - `http://my-site.com/product-details?id=2093&amp;customerid=3`
    - `http://my-site.com/product-details?customerid=3&amp;id=2093`
- Not all querystring parameters affect content; a common example of this is the tracking parameter used to track referrals. eg:
    - `http://my-site.com/?referred_by=Dipun+Mistry`

For the second question, I have explored three options:

1. Ignore any URL variations
    - One problem with this, is that users may mistype your webpage URL, and if your site is pedantic about these changes, it could put them off.
    - Another issue is that external sites linking to yours may also make this mistake and you'd lose out on some sweet sweet ranking points.
    - Finally, some sites will append tracking querystrings when linking to your site, this helps you to track referrers and affiliates. If you start ignoring querystrings, you may lose this ability to track.
2. Redirect permanently to your canonical URL
    - Redirects tell search engines with absolute certainty that the URL was simply a variation of the canonical URL.
    - Typically, this should be reserved for pages that have been moved from one URL to another, or to attempt to consolidate the score of your duplicate pages pro-actively after accruing some valuable points over time.
    - If your redirects are too aggressive, you may loose some cross cutting functionality (eg. tracking referrers using querystrings).
3. Add a canonical meta tag
    - Canonical meta tags allow you to consolidate your scores from a dynamic page to one canonical page.
    - Canonical meta tags allow you to define which querystrings are canonical, and which the search engine can ignore.
    - Canonical meta tags don't affect the functional aspect of your web application. They are the least intrusive option.

Ignoring URL's tend to be unfriendly and tend to be very unhealthy for your site. Permanent redirects can be very aggressive, and are not very affective at handling extra non-canonical querystrings; either they remove them for the sake of the search engine, or leave them in for the sake of the application; either way, you will probably be making a compromise. Canonical URL's tend to be the best solution for most issues, but a lot of articles online still favour the permanent redirect where possible.
## My Solution (The code)
I decided to create a solution that solves the problem of duplicate content, whilst making it as modular as possible. Here are the main assumptions made:
- Of the 3 mechanisms to handle duplicate content, permanent redirects and canonical meta tags are the most useful.
- The canonical parts of a URL should be defined at the action rather than at the route or earlier.
- The criterion used to determine how to canonicalize a request URL should be modular.
- "Case sensitive URL's" doesn't necessarily mean "lowercase URL's".
- Permanent redirects are the first line of defence, and canonical meta tags are used to complete the canonicalization unobtrusively.

The first thing to do, is apply the `HandleDuplicateContentFilter` to your application. This filter does nothing by itself, so I recommend applying it as a global filter.
``` csharp
public class FilterConfig
{
    public static void RegisterGlobalFilters(GlobalFilterCollection filters)
    {
        // ...
        filters.Add(new HandleDuplicateContentFilter());
        // ...
    }
}
```

The next thing is to decide what rules should apply to your application, and which rules should cause a redirect when violated, and which rules are only handled by the canonical meta tag. To get you started, I have created a recommended helper, but if you require more control, or would like to understand the rules better, feel free to read the documentation further on Github. Also note, that you can create multiple rulesets with different names for your application if you require.
```csharp
public class SeoConfig
{
    public static void SetupDuplicateContentRules(SeoRequestRulesetCollection rules)
    {
       rules.Add("Default", SeoRequestRuleset
            .Recommended(new Uri("https://www.my-site.com")));
    }
}
```

The recommended ruleset will redirect for querystring case matching, url path case matching, omitting default values (eg. redirecting from /home/index to /), and trailing and repeating slashes. The canonical meta tag will contain fixes for: unordered querystrings, and removing non-canonical querystrings. If you provide a Uri as in this example, the recommended ruleset will also verify and redirect if the scheme or hostname of the request does not match. There is the ability to provide a slugProvider for validating and correcting slugs in your urls, and you can always create and append your own custom rules by extending ICanonicalRule.

At this point, we will still have no effective changes to your application. The next step is to define your canonical actions; you can do this by adding a Canonical attribute to your action.
```csharp
[Canonical(Ruleset = "Default", Query = new[] { "name" }, Sensitive = new[] { "id" })]
public ActionResult About(string id, string name)
{
    return View(new {id, name});
}
```
This is where it gets interesting. The ruleset parameter should match the name given when configuring your rules. "Query" is a list of querystring parameters that, when present, contribute towards changing the content of the page (also known as canonical querystrings). "Sensitive" represents the parameters that should not be affected when canonicalizing the URL. This applies to both querystring parameters as well as route parameters (eg. the id in "{controller}/{action}/{id}").

The canonical attribute can also be placed on the controller, and any actions will inherit the tag. If the attribute was placed on the controller and on the action, the ruleset will come from the action unless undefined, in which case it will come from the controller and the Query and Sensitive lists will be merged.

Knowing which querystrings are canonical is very important for the rules provided; apart the rule which removes non-canonical querystrings, other rules will only affect canonical querystrings. Knowing the sensitive parameters is also important; if the casing of the parameter may change the content of your page, or could lead to a route mismatch, you should add the parameter name. This will prevent that parameter from changing when generating the new canonical URL. A real world example of this, is the youtube id parameter which uses a base64 string for its video ids. Changing the case of these ids would change the video being loaded. Adding the parameter to the Sensitive list would prevent the rules from validating or changing that parameter value.

Finally, there is one last thing to do: create the canonical tag in your layout file:
```html
<meta rel="canonical" href="@Url.Canonical()" />
```
That's it! To learn more, please check out my [Github page](https://github.com/dipunm/SEO.DuplicateContent), and feel free to leave any comments below.