# Dispatcher Filter Best Practices

  * [Disclaimer](#disclaimer)
  * [Purpose](#purpose)
  * [Runmode](#runmode)
  * [Dispatcher Filter Capability](#dispatcher-filter-capability)
  * [Rule Sequence and General Behaviour](#rule-sequence-and-general-behaviour)
  * [Essential Knowledge](#essential-knowledge)
    + [AEM (Sling) Requests](#aem-sling-requests)
    + [Filtering of Shortened or "Pretty" URLs](#filtering-of-shortened-or-pretty-urls)
    + [Rule Semantics](#rule-semantics)
    + [Glob Matching Syntax](#glob-matching-syntax)
    + [Regular Expressions](#regular-expressions)
  * [Filter Configuration Principles and Techniques](#filter-configuration-principles-and-techniques)


## Disclaimer

This document provides best practice guidelines and also looks at common problems in filter rulesets and improvements to overcome them. There is no guarantee that following this guide makes the system bullet-proof. The AEM Dispatcher is not a replacement for a full-fledged Web Application Firewall. 

## Purpose

This page provides dispatcher filter configuration guidance and best-practice recommendations.

## Runmode 

This documentation is focused on the protection of publish instances. 

The principles are valid for all use, but being so prescriptive as described here is not appropriate for an author instance where ACL protection is in place and generally requests are more complex.  Note that AEM as a Cloud Service author instances do not have dispatchers.

## Dispatcher Filter Capability

The dispatcher filter is a type of simple, stateless firewall.  It deals only with individual request URIs and has no knowledge of anything about its environment, traffic quantity, request 'signatures', body or header size, post request payload or other such features that a more sophisticated firewall might use to control traffic.

The following example of capabilities is for the submission of an HTML form to an AEM endpoint using a POST request:

The filter CAN

- limit the POST request to a very tightly specified endpoint in AEM based on URL pattern matching.

The filter CANNOT:

- control the rate of POSTs to this endpoint (prevent flooding).
- control the size of the POST request payload.
- block requests based on IP addresses.

A Web-Application firewall would normally be used to provide this additional security functionality, although the Apache or IIS web-servers that are used in the dispatcher can also be used to 

## Rule Sequence and General Behaviour

Dispatcher rules should follow the sequence:

1. Deny all requests.

2. Rules giving the minimum access necessary to browse the website, such as a GET request with no selectors or suffixes allowed, and file extensions needed for basic HTML content. 

3. Additional rules to allow access for more specialized requests (e.g. /content/mysite/searchresults.page1.html?search=abc)

4. Denies to block possible weaknesses.

Requests are tested against all rules in sequence.  The last rule which matches a particular request will be applied, independently of other rules.  For example a filter rule set in which the last rule is "deny all" would let no requests through regardless of how many earlier "allows" had matched, and vice versa.  

## Essential Knowledge

### AEM (Sling) Requests

This is an example of an AEM (Sling) URL using all the elements that can be used in dispatcher filters:   /content/mysite/apage.aselector.html/123.html?a=1&b=2

This and other examples are broken down in the table below.

|Path|Selectors|Extension|Suffix|Params|
|---|---|---|---|---|
|/content/mysite/apage|aselector.anotherselector|html|/123.html|a=1&b=2|
|/etc/clientlibs/ascript|min|js|||
|/libs/dam/components/scene7/common/clientlibs/viewer|min|js||

### Filtering of shortened or "pretty" URLs

Rewriting of shortened "pretty" AEM page URLs (without `/content/mysite/` and possibly without `.html`) happens in the Apache or IIS webserver before the request is processed by the dispatcher filter. Therefore Dispatcher filter rules for AEM website content are applied to raw AEM URLs starting with `/content/`.

Using Adobe HelpX as an example, the URL `https://helpx.adobe.com/dispatcher.html`, is rewritten to a path such as  `/content/helpx/topics/dispatcher`.  

### Rule Semantics

While filter rules themselves are independent, the last matching rule in a filter set wins.  No relationship between rules is supported.

Rules can be written using Glob or Regular expression patterns to match URL elements.  

Filter rules are named (usually numbered) but these names have no effect on filter function, therefore:
- Rules are processed in order of appearance irrespective of any numbering sequence
- Rules with the same name/number are processed as normal, with the conflict reported in the log

Optionally human-readable names can be used and if a complex rules set, namespaces, for example:

/forms_001

/forms_002

/search_001

/search_002

### Glob Matching Syntax

Glob pattern matching is used when filters use double quote, for example:
```
/0001 { /type "deny" /url "*" } 
```
Here the `/url` is compared to the Glob wildcard * and anything matches.

Note that the use of /glob in filter rules is deprecated. The alternative `/url` should be used instead.

### Matching using /url

`/url` allows matching agains the entire URL string (except host), making `/path`, `/suffix`, `/selectors` and `/extension` unecessary, although they are still processed.  In order to avoid errors resulting from overlapping matching (eg. `/url` and `/extension are both matching the extension in the same rule), only /type and /method should be used in combination with /url.

For example:

```
/url '/content/a/b/c.+' /extension "html" 
````

would not match /content/a.b.c.json as the extension is restricted to `html`, however preferable would be:

```
/url '/content/a/b/c[^.]+\.html'
```

Here the use of [^.]+ is ensuring that no dots occur until .html, so excluding selectors and extensions other than html.

### Regular Expressions

Regular expressions matching patterns are contained within single quotes and use the Python pattern. 

- String start and end is implicit, ie. ^ and $ start and stop characters are not necessary.  
- Escaping of / is not necessary.

```
/0001 { /type "deny" /url '.*' }
```

Here the /url uses the Regex * to define that zero or more of . (any character) are required for a match.  

#### Regex Limitations

Some advanced regex features are not supported, for example Negative Lookahead.  

! Contributions welcome: An improvement to this document would be to fully explore this and list examples of what can and cannot be done.  


## Filter Configuration Principles and Techniques

### 1. Initial Deny rule 

The start of all dispatcher rule sets should be a blanket deny.

/0001 { /type "deny" /url "*" } or similar.

### 2. Broad Access Rules Restrict Extensions, Suffixes, Selectors

Early allow rules, which give general access should restrict extensions, suffixes and selectors.

### Example 1 - Insufficient restrictions in the first Content rule

The first allow rules in a dispatcher configuration are extremely important. Many live configurations contain imprecise rules which allow far too many requests to pass, and therefore create a need to apply deny rules later in the ruleset.  

An example overly permissive and commonly found content rule is given below. After this rule achieving a secure best-practice filter configuration would be difficult or even impossible.

```
# ! Not sufficiently restrictive !

/0010 { 
  /type "allow" 
  /extension '(gif|ico|jpe?g|gif|pdf|png|svg|html)' 
  /path "/content/*" 
} 
```

#### Intention

The explicit /extension will prevent .json and .xml except in crafted exploit URLs.

It should and does match URLs like

```
/content/assets/logo.jpg
/content/home/jcr:content/par/logo.image.jpg
```

And it does not allow access to URLs like

```
/content/home.xml
/content/home.infinity.json
/content/home.foobar
```



But the selectors are not restricted (no use of _/selectors ''_ ). This allows calling the resource with undesired selector that might leak information or pollute the cache directory:

```
GET /content/myhome.foo.bar.001.html
GET /content/myhome.foo.bar.002.html
...
```

Also POST / DELETE requests are allowed (no use of _/method_ )

Suffixes are not restricted (no use of _/suffix ''_ ). This allows calling the resource with undesired suffixes that might leak information or pollute the cache directory:

```
GET /content/myhome.html/foo/bar.baz 
..
```

Moreover, the path is too imprecise. It might undesireably match private resources, future sites or internal (legacy) configuration if this is mixed into the /content structure:  

```
GET /content/sites/internal-documentation.html
GET /content/sites/site-a/configuration/homepage.html
...

```

#### Improvements

```
/0010 { 
  /type "allow" 
  /method '(GET|HEAD|OPTIONS)' 
  /extension '(gif|ico|jpe?g|gif|pdf|png|svg|html)' 
  /path '/content/(dam|cq:tags|experience-fragments)?/?mysite/.+' 
  /suffix '' 
  /selectors '' 
} 
```

<!-- TODO: check request methods - do we need OPTIONS in most cases?  HEAD for Ajax checks? -->

 Would be a better rule. Here the path has been restricted, request method defined and no selectors or suffixes are allowed.

<!-- single quotes denote regular expressions. Maybe for empty filters this is a bit overkill? And maybe even wrong. An empty regex matches everything... but wait - no Dispatcher regexes are always a perfect match, right? Anyway, a simple "" seems faster and more less confusing than '' -->

### Example 2. Clientlibs Rules - Not Sufficiently Restrictive, Selectors used:

The example below is a commonly found rule for allowing access to clientlibs.  It is almost certainly too permissive.

```
/0013 { 
  /type "allow" 
  /method "GET" 
  /url "/etc.clientlibs/*" 
} 
```

#### Intention

The explicit _/method "GET"_ will prevent POST and DELETE (writing) requests to the repository.

The /url should and does match URLs like

```
/etc.clientlibs/somejavascript.js
/etc.clientlibs/somejavascript.1234567890abc.js
```

The remaining limitations are though only on the entire URL, and only that it must start with _/etc.clientlibs_.

There is no limitation on the use of suffixes, selectors or file extensions.  The following URLs would for example be accepted by this filter:

```
/etc.clientlibs/y/b/c.foo.bar.-1.json
/etc.clientlibs/y/b/c.foo.bar.-1.xml/a.json
```

#### Improvement - With a common and major weakness

```
# ! rule with major flaw in use of selectors !

/0012 { 
  /type "allow" 
  /method "GET" /url "/etc.clientlibs/*"  
  /suffix '' 
  /selectors '(min|[a-f0.9]+)'
  /extension '(js|css)' } 
```

This rule in an improvement, with limitations on the use of suffixes and the extension. However the use of _/selectors_ here is not achieving the intended results.

Any selector here is allowed regardless of the  _/selectors_ regex. A Sling URL can have many selectors. If any one of matches the content of the /selectors rule, then the /selectors restriction will be satisfied.

The _/selectors_ rule is designed for deny rules and should never be used in an allow rule except for the empty _/selectors ''_.

#### Improvement - Solving the /selectors problem

```
TODO: check if /suffix and /extension are required here!
TODO: review REGEX

/0012 { 
  /type "allow" 
  /method "GET" 
  /url '/etc\.clientlibs/[^.]+(\.min)?\.[a-f0-9]+\.(js|css)'  
} 
```

### 3. Avoiding Complexity

Many dispatcher filter configurations become unecessarily complex over time, as developers add additional rules to support features without understanding the mechanism or entire file.

To avoid this the following are recommended:

- Avoid overly complex regexes, and if necessary comment regexes carefully to explain what they do.
- Follow a logical sequence of rules
- Apply the principle of separation of concerns, by ensuring that each rule has a sensible scope. Naming rules will help here, for example the following rule is probably undesirable and this would be obvious as soon as a descriptive rule name is used:

```
/contentForTenant1Tenant2Tenant4 { ...  }
```

better would probably be:

```
/contentAllTenants { ...  }

/contentForTenant1 { ...  }

/contentForTenant2 { ...  }
```

- Avoid overlapping rules except when in a logical sequence such as shown in the Tenant examples above.

### 4. Imprecise Regexes for Paths

Regexes should match as closely as reasonably possible.  This example rule would allow json access to a home page of a we-retail site (perhaps for a custom json servlet):

```
# ! path regex too greedy !

/primaryContent { 
  /type "allow" 
  /method "GET" 
  /path '/content/we-retail/[^/.]+/[^/.]' 
  /extension 'json' 
  /selectors ''
  /suffix ''
} 
```

This rule follows the best practices describe so far, but the wildcard path matches [^/.]+ are too greedy here and would allow, fof example /content/mysite/us/**jcr:content**.json

#### Improvement

The regex should be more prescriptive.  An improved rule would be:

```
/primaryContent { 
  /type "allow" 
  /method "GET" 
  /path '/content/we-retail/[a-z]{2}/[a-z]{2}]'
  /extension 'json'
  /selectors ''
  /suffix ''
```

###  5. URL Parameters 

The points so far have ignored a further filter function, /query, which handles URL Parameters. These are the properties following a `?` at the end of a URL, eg:

`http://domain.com/some/path/page.html?aqueryparemeter=1&anotherqueryparameter=2`

#### /query in Deny Rules

The /query switch is definitely useful in _deny_ rules, for example in avoiding a common undesirable username/password popup the following could be used:

```
/denyAccessToSlingAuth {
    /type "deny"
    /query 'sling:authRequestLogin=.*'
}
```

#### /query in Allow Rules

The use and benefits of using /query in _allow_ rules are less clear:

- Queries are used unpredictably for content and asset URLs by common tools such as web analytics.
- Regex use in allow rules in queries is difficult, for example:
 ``` 
 # request must include ONLY parameters a, b and c, in the specified order with any value.  Different order would not match.
  /query 'a=[^&]+&b=[^&]+&c=[^&]+' 
 ``` 
- It is questionable whether there are any significant security benefits from limiting queries, given a solid ruleset as described above.

#### Conclusion

This document currently has no best practice recommendations for the use of query constructs in allow rules.  Contributions are of course welcome.

