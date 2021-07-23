# Proposal: URL-based Annotation

This document describes a proposal to allow a URL to specify, or be, an [Annotation Source](https://github.com/bokand/web-annotations#ideas--concepts). It does this by introducing a new [fragment directive](https://wicg.github.io/scroll-to-text-fragment/#the-fragment-directive), `note()`. `note()` is a functional style directive and can take one of two forms which are mutually exclusive.

The first form allows the URL to specify an external annotation source using an `href` parameter:

```
https://news.annotations.example.com#:~:note(href=https://annotations.example.com)
```

Will tell the user agent to fetch annotations from `https://annotations.example.com`.

The second form allows embedding annotation content directly into the URL using a `content` parameter, making the URL the annotation source:

```
https://example.org#:~:note(content={<<Serialized JSON-LD here>>})
```

The user agent will parse the JSON-LD string using the [WebAnnotation Data Model](https://www.w3.org/TR/annotation-model/) and present the annotations without requiring a separate fetch. We'll call this a "URL-embedded annotation".

## Use Cases
We wish to enable basic user sharing scenarios, some examples:

1. The user adds one or more notes to the text of a page they are reading
2. Once finished, the user clicks a “share notes” button
3. The user receives a special URL they share with a friend.
4. When the friend opens the URL, they are navigated to the original page but now see the notes the first user made

A similar but less point-to-point use case is the user sharing a link like this via social media, or their blog. Instead of sharing the URL directly with a friend (via SMS or instant messaging), they can post it to Facebook or Twitter, allowing their followers to see their comments with the content.

> 📝 Some extension-based annotation tools already support such use cases but use a non-standard mechanism to achieve this (also usually encoding something in the URL fragment) and rely on the receiving party to have the same tool installed (or load the content through a proxy and inject the annotations into the content). The `note()` mechanism could be used by those tools as well so that the recipient can read annotations in any compatible tool or user agent.

Another interesting use case is aggregators providing links with commentary overlaid. For example:

1. The user visits `https://hackerhappenings.com` which contains a feed of user generated links to interesting articles, along with some user generated comments in a  “comment section” hosted on `hackerhappenings.com`
2. HackerHappenings provides a new additional `[annotated]` link that adds a `#:~:note(href=notes.hackerhappenings.com/article123)` directive to an article link (in this case, `article123`). 
3. The user clicks on the `[annotated]` link which takes them to the external article hosted on, say, `example.com`.
4. The user’s browser reads the attached `note()` directive and issues a fetch to `notes.hackerhappenings.com/article123`.
5. `hackerhappenings.com` reads the comments it has for `article123`, renders them in the WebAnnotation data model (attaching comments that quote the article to the relevant text), and responds with the comments as annotation data.
6. The user now sees the HackerHappenings comments section as annotations, while they read the `example.com` article.

## Embedded URL format

A critical piece is how to embed annotations into the URL. We propose that annotations are JSON-LD serialized WebAnnotations, [e.g](https://www.w3.org/TR/annotation-model/#h-embedded-textual-body:~:text=EXAMPLE%205%3A%20Textual%20Body). As a minimal example:

```
{
  "@context": "http://www.w3.org/ns/anno.jsonld",
  "id": "http://localhost/annotation",
  "type": "Annotation",
  "body": {
    "type" : "TextualBody",
    "value" : "This is the comment I wrote on the page"
  },
  "target": {
    "Source": "http://example.org/article",
    "selector": {
  "type": "TextQuoteSelector",
  "exact": "fun with notes"
    }
  }
}
```

User agents may vary in which fields of the WebAnnotation format they can interpret and render.

Questions:

* Some of the fields, as currently specified, don’t make sense for URL-embedded annotations, e.g. what is the IRI in this case? The target source is implied by the URL. How do we deal with these?
* There’s a lot of boilerplate, perhaps we can specify implied values for these in URL-embedded annotations
* Should we specify some minimum-set of properties that all user agents MUST interpret?

To embed into the URL, the content is added as a note directive (this makes it [invisible](https://github.com/WICG/scroll-to-text-fragment#fragment-directive)[[*](http://crbug.com/1096983)] to the page):

```
https://example.org#:~:note(content={<<Serialized JSON-LD here>>})
```

Multiple annotations can be included by encoding an “[annotation collection](https://www.w3.org/TR/annotation-model/#annotation-collection)”. 

Questions:

* Should we support adding multiple directives? e.g.
  ```
  https://example.org#:~:note(content={<<note1>>})&note(content={<<note2>>})
  ```
  Embedded notes can use a single directive with an annotation collection so this doesn't seem necessary. However, with the `href` param multiple notes could specify multiple sources:
  ```
  https://example.org#:~:note(href=https://notes.example.org)&note(href=https://notes.personal.com)
  ```

### URL Length

An obvious issue for URL-embedded notes is URL length. For example, the percent-encoded version of the example annotation above is:

```
https://example.org#:~:note(content=%7B%22%40context%22%3A%22http%3A%2F%2Fwww.w3.org%2Fns%2Fanno.jsonld%22%2C%22id%22%3A%22http%3A%2F%2Flocalhost%2Fannotation%22%2C%22type%22%3A%22Annotation%22%2C%22body%22%3A%7B%22type%22%3A%22TextualBody%22%2C%22value%22%3A%22ThisisthecommentIwroteonthepage%22%7D%2C%22target%22%3A%7B%22Source%22%3A%22http%3A%2F%2Fexample.org%2Farticle%22%2C%22selector%22%3A%7B%22type%22%3A%22TextQuoteSelector%22%2C%22exact%22%3A%22funwithnotes%22%7D%7D%7D)
```

We could shorten these by making some boilerplate implicit. Another idea is to specify a compression scheme. Using a compression scheme + base64 encoding has two advantages: compressed data and no percent-encoding expansion. Here’s an example of the above note after Brotli compression in base64:

```
https://example.org#:~:note(content=Gx4BYIzTFfMjhHUfSgvR3FK/UBYWitrMGspgL/8w/sFtKKxZ+nQUvm3V2C/VFXh8gq/lbAZBs914jZfNjUIyi+M+ZjIYCl9sfZxzL0MM+mVzR++puMfnE36tAqsHEQV0HIm1AScfzr4J2ElWcbrlH7O0YylYlUWtWx8qrrbTlggfKlo6QQm0olA3NjyaSJvVVkrlFi84PgSTq6iY+2zPdG8xsqAa)
```

This is analogous to other parts of the platform where resources can be specified using `data:` URLs.

## Annotation Indicators

The existence of an annotation over a snippet of text is typically shown by highlighting the text. Pages should have the ability to style the highlights. We propose reusing the [::target-text pseudo class](https://developer.mozilla.org/en-US/docs/Web/CSS/::target-text) added for text fragments:

```
::target-text {
  background-color: rgba(255, 0, 0, 1);
}
```

* Perhaps we should introduce a new, related `::annotated-text` pseudo instead?

## Page Author Control

Authors should be able to signal whether they prefer annotations not be shown (or shown). The exact interpretation of this should be left to the user agent so that the agent can use it to make the best decision on the user’s behalf. This could be a signal combined with others such as user configured settings and user agent smarts/learning.

Propose introducing a new type of script block to provide origin-based rules on annotations sources:

```
<script type="annotationrules">
  // Show only annotations coming from notes.example.com
  "Annotations": [
    { "rule": "hide", "href": "*" },
    { "rule": "show", "href": "notes.example.com"}
  ]
</script>
```

The likely effect is this would determine if an annotation can be shown on page load with or without a user action. A lot here will depend on UA decisions and UX.

As a possible example: a UA may normally load a page with highlights showing available annotations, clicking on a highlight would show the annotation content or bring up an annotation side-bar. If a page declares that it’d rather not have annotations showing, the user agent may completely hide both the annotations and highlights until the user explicitly clicks a “show annotations” button in its UI.

Questions:

* This precludes a server checking for annotation rules without downloading the whole page (i.e. in contrast to just checking for a robots.txt). Is this a use case we care about?
* Is being able to have different rules for different sources beneficial?
* I’ve seen reference to existing thought and discussion on this, has any work been done?

