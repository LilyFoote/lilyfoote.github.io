+++
title = "Quirks in Django's template language part 3"
date = 2026-01-25
draft = false
categories = ["Django", "Python"]
tags = ["django", "templates", "python"]
+++

## Django Rusty Templates

Since September 2024, I (and some collaborators) have been building a reimplementation of Django's templating language in Rust, called [Django Rusty Templates](https://github.com/LilyFirefly/django-rusty-templates). During this process, we have discovered several weird edge-cases and bugs, some of which I've talked about before ([Part 1](/posts/django-template-quirks/), [Part 2](/posts/django-template-quirks-2/)).

### The `{% now %}` tag

[The `{% now %}` tag](https://docs.djangoproject.com/en/6.0/ref/templates/builtins/#now) is provided by Django to display the current date or time. It uses a format specifier language [designed for compatibility with PHP](https://docs.djangoproject.com/en/6.0/ref/templates/builtins/#std-templatefilter-date), which feels like an odd decision in retrospect, since most Python programmers are used to the [`strftime` style formatting provided by the Python `datetime` module](https://docs.python.org/3/library/datetime.html#format-codes).

The `now` tag can be used like:

```django
It is {% now "jS F Y H:i" %}
```

which will render as:

> It is 25th Jan 2026 20:26

While [Pravin was implementing support in Django Rusty Templates](https://github.com/LilyFirefly/django-rusty-templates/pull/278) we realised that the `now` tag doesn't support template variables for the format string. For example, you might expect to be able to write:

```django
It is {% now date_format %}
```

and call `template.render` with a `context` providing the `date_format` dynamically:

{{< narrow >}}
```python
context = {
  "date_format": "jS F Y H:i",
}
template.render(context=context)
```
{{< /narrow >}}
{{< wide >}}
```python
template.render(context={"date_format": "jS F Y H:i" })
```
{{< /wide >}}

However, instead the `date_format` is treated as a string, not a variable. In addition, to handle the quote characters around a format string, Django trims away the first and last character. This means that Django actually treats `{% now date_format %}` as `{% now "ate_forma" %}`, which will render something like:

> p.m.31GMT_8:262026Sun, 26 Jan 2026 20:26:08 +00:0001p.m.

This trimming of the first and last character is the cause of many similar rendering quirks. For example, instead of raising a `TemplateSyntaxError` for a number, the `now` tag will strip the first and last digit and render that: `{% now 1234 %}` becomes `23`.

### Invalid numeric literals

Django supports rendering literal numeric values in variables. For example:

```django
{{ 123 }}: {{ 4.6 }}
```

would render as:

> 123: 4.6

However, when a variable is provided with an invalid literal, such as `1.2.3`, Django doesn't raise a `TemplateSyntaxError`:

```django
{{ 1.2.3 }}
```

Instead, it is treated as a dotted variable lookup in the `context`. Usually nothing in the `context` matches and an empty string is rendered. But it's also possible to construct a `context` to match:

{{< narrow >}}
```python
context = {
  1: {2: {3: "foo"}},
}
template.render(context=context)
```
{{< /narrow >}}
{{< wide >}}
```python
template.render(context={1: {2: {3: "foo"}}})
```
{{< /wide >}}

would render as `foo`.

I discovered this while implementing the `{% if %}` tag and [reported it in ticket 36658](https://code.djangoproject.com/ticket/36658), but as I described above this applies to any variable.

### The `{% lorem %}` tag

[Django provides the `{% lorem %}` tag](https://docs.djangoproject.com/en/6.0/ref/templates/builtins/#lorem) for providing sample data in templates. The `lorem` tag takes up to three arguments: `count`, `method` and `random`. `method` chooses between three variants: `w` for words, `p` for HTML paragraphs and `b` for plain-text paragraphs. `count` controls the number of words or paragraphs rendered. `random` controls whether the standard lorem paragraph is used or if random words are used.

When `count` is positive, this all works as expected. However, if `count` is negative, the behaviour changes based on the `method`. If `method` is `p` or `b` for paragraphs, the empty string is rendered. If `method` is `w` for words and `random` is not provided, then Django will remove the last `-count` words from the common lorem paragraph.

```django
{% lorem -5 w %}
```

will render:

> lorem ipsum dolor sit amet consectetur adipisicing elit sed do eiusmod tempor incididunt ut

instead of the full paragraph:

> lorem ipsum dolor sit amet consectetur adipisicing elit sed do eiusmod tempor incididunt ut labore et dolore magna aliqua

## Closing thoughts

Most Django developers will never try passing a negative `count` to the `lorem` tag, so this oddity doesn't really matter.

Providing an invalid numeric literal is more likely to be a typo than intentionally matching a lookup in `context` so I think this should be changed to become a `TemplateSyntaxError`. This is reinforced for me by the fact that it isn't possible to look up the first or second level from a `context` like `{"1": {"2": "content"}}`, since `{{ 1 }}` or `{{ 1.2 }}` are numeric literals instead of variables.

I also think that the `{% now %}` tag should be updated to support a variable to provide the `format` argument. This would provide some useful flexibility to template authors. I also would like to see the validation of the `format` be much stricter, requiring a valid string literal instead of the current behaviour of arbitrarily stripping the first and last character.

While these quirks and bugs vary in seriousness, I continue to think the Django template language is very well designed overall, with behaviours like these rare to run into in practice.

Finally, I want to thank Pravin for the hard work he's put into implementing the `now` and `lorem` template tags in Django Rusty Templates.

## Update

I have opened issues on the Django new-feature tracker for [the `lorem`](https://github.com/django/new-features/issues/115) bug and [the variable bug](https://github.com/django/new-features/issues/116).
