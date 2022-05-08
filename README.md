# Django ORM Cheatsheet

If you wanted to read a lot of text you'd be reading the Django docs, especially these pages:

- [Queryset API (select_related, bulk_update, etc.)](https://docs.djangoproject.com/en/dev/ref/models/querysets/)
- [Query Expressions (Subquery, F, annotate, aggregate, etc.)](https://docs.djangoproject.com/en/dev/ref/models/expressions/)
- [Aggregation](https://docs.djangoproject.com/en/dev/topics/db/aggregation/)
- [Database Functions](https://docs.djangoproject.com/en/dev/ref/models/database-functions/)

But you don't want that, so you're here. I'll get to the point.

# Our models

```python
from datetime import date

from django.db import models


class Blog(models.Model):
    name = models.CharField(max_length=100)
    tagline = models.TextField()

    def __str__(self):
        return self.name


class Author(models.Model):
    name = models.CharField(max_length=200)
    email = models.EmailField()

    def __str__(self):
        return self.name


class Entry(models.Model):
    blog = models.ForeignKey(Blog, on_delete=models.CASCADE)
    headline = models.CharField(max_length=255)
    body_text = models.TextField()
    pub_date = models.DateField()
    mod_date = models.DateField(default=date.today)
    authors = models.ManyToManyField(Author)
    number_of_comments = models.IntegerField(default=0)
    number_of_pingbacks = models.IntegerField(default=0)
    rating = models.IntegerField(default=5)

    def __str__(self):
        return self.headline
```

# Starting point

```python
Entry.objects.all()

# SELECT "blog_entry"."id", "blog_entry"."blog_id", ...
# FROM "blog_entry" ...;

# [<Entry: Off foot official kitchen another turn.>,
#  <Entry: Yet year marriage yes her she.>,
#  <Entry: Special mission in who son sort.>,
#  ...
# ]
```

# Make fewer queries

These are the biggest bang for your buck. Use them often.

## `select_related()`

\*-to-one relationships (one-to-one, many-to-one)

### without `select_related()`

```python
entry = Entry.objects.first()
# SELECT ... FROM "blog_entry" ...;

# this attribute access runs a second query
blog = entry.blog
# SELECT ... FROM "blog_blog" WHERE ...;
```

### with `select_related()`

```python
entry = Entry.objects.select_related("blog").first()
# SELECT "blog_entry"."id", ... "blog_blog"."id", ...
# FROM "blog_entry"
# INNER JOIN "blog_blog" ...;

blog = entry.blog
# no query is run because we JOINed with the blog table above
```

## `prefetch_related()`

\*-to-many relationships (one-to-many, many-to-many)

### without `prefetch_related()`

```python
entry = Entry.objects.first()
# SELECT ... FROM "blog_entry" ...;

# this related query hits the database again
authors = list(entry.authors.all())
# SELECT ...
# FROM "blog_author" INNER JOIN "blog_entry_authors"...
# WHERE "blog_entry_authors"."entry_id" = 4137;
```

### with `prefetch_related()`

```python
entry = Entry.objects.prefetch_related("authors").first()
# SELECT ... FROM "blog_entry" ...;
# SELECT ...
# FROM "blog_author" INNER JOIN "blog_entry_authors" ...
# WHERE "blog_entry_authors"."entry_id" IN (4137);

authors = list(entry.authors.all())
# no query is run because we have an in-memory
# lookup of the relevant authors from above
```

## `update()`

### without `update()`

```python
david = (
    Author.objects.filter(name__startswith="David")
    .prefetch_related("entry_set")
    .first()
)
# SELECT ... FROM "blog_author" WHERE "blog_author"."name" LIKE 'David%' ...;
# SELECT ... FROM "blog_entry" INNER JOIN "blog_entry_authors" ...;

# There are many ineffcient ways to update all of David's entries.
# This is one of them.
for entry in david.entry_set.all():
    entry.rating = 5
    entry.save()
# One query for each Entry. Even if we used bulk_update(),
# we're still making more queries than we need.
```

### with `update()`

```python
Entry.objects.filter(authors__name__startswith="David").update(rating=5)
# UPDATE "blog_entry" SET "rating" = 5 WHERE "blog_entry"."id" IN ...;
```

# Query things other than columns, AKA do your work in the DB, not in Python

## `annotate()` (first look)

`annotate()` adds an attribute to each row of the result. Use this with `F()` to add one field from a related object (this makes more sense when you have objects separated by a long chain of relations).

`annotate()` is very powerful and makes sense to use in many situations, so just bear with this first unrealistic example.

```python
entries = Entry.objects.annotate(blog_name=F("blog__name"))[:5]
# SELECT "blog_entry"."id", ... "blog_blog"."name" AS "blog_name"
# FROM "blog_entry" INNER JOIN "blog_blog" ...;

# Now we have Entry objects with one extra attribute: blog_name
[entry.blog_name for entry in entries]
# ['Hunter-Rhodes',
#  'Mcneil PLC',
#  'Banks, Hicks and Carpenter',
#  'Anderson PLC',
#  'George-Bray']
```

## `filter()` and `Q()`

Use `Q()` to do more filtering in the database, and less in your application.

```python
low_engagement_posts = Entry.objects.filter(
    Q(number_of_comments__lt=20) | Q(number_of_pingbacks__lt=20)
)

list(low_engagement_posts)
# SELECT ... FROM "blog_entry"
# WHERE
#   ("blog_entry"."number_of_comments" < 20 OR
#   "blog_entry"."number_of_pingbacks" < 20);
```

## Query Expressions

Many parts of the Django ORM expect a [query expression](https://docs.djangoproject.com/en/dev/ref/models/expressions/). Query expressions include:

- references to fields (possibly on related objects) `F()`
- SQL functions like `CASE`, `NOW`, etc.
- Subqueries with `Subquery()`

### `F()`

The classic example is an increment, but recall that `F()` can be used anywhere a query expression is required.

```python
Entry.objects.filter(authors__name__startswith="David").first().rating
# 5

Entry.objects.filter(authors__name__startswith="David").update(
    rating=F("rating") - 1
)
# UPDATE "blog_entry"
#   SET "rating" = ("blog_entry"."rating" - 1) WHERE ...;

Entry.objects.filter(authors__name__startswith="David").first().rating
# 4
```

### `CASE` and conditional logic

[Django docs page](https://docs.djangoproject.com/en/dev/ref/models/conditional-expressions/)

`Value()` below is a query expression for a string, integer, bool, etc. literal value. You could use `F()` or another expression as the `then` value or even in place of the `rating=` condition.

```python
entries = Entry.objects.annotate(
    coolness=Case(
        When(rating=5, then=Value("super cool")),
        When(rating=4, then=Value("pretty cool")),
        default=Value("not cool"),
    )
)
# SELECT ...
# CASE
#   WHEN "blog_entry"."rating" = 5 THEN 'super cool'
#   WHEN "blog_entry"."rating" = 4 THEN 'pretty cool'
#   ELSE 'not cool'
# END AS "coolness"
# FROM "blog_entry" LIMIT 5;

[f"Entry {e.pk} is {e.coolness}" for e in entries[:5]]
# ['Entry 4137 is super cool',
#  'Entry 4138 is not cool',
#  'Entry 4139 is not cool',
#  'Entry 4140 is pretty cool',
#  'Entry 4141 is not cool']
```

### `Subquery()` and `OuterRef()`

Annotate each blog with the headline of the most recent entry.

This pattern is the only use for `Subquery()` I have ever found: query a \*-to-many relation, `OuterRef("pk")` to "join" the rows, use `values()` to return one column and `[:1]` to return one row from the subquery. This is basically a copy of the example in the Django docs.

```python
blogs = Blog.objects.annotate(
    most_recent_headline=Subquery(
        Entry.objects.filter(blog=OuterRef("pk"))
        .order_by("-pub_date")
        .values("headline")[:1]
    )
)
[(blog, str(blog.most_recent_headline)) for blog in blogs[:5]]
# SELECT
#   "blog_blog"."id",
#   "blog_blog"."name",
#   "blog_blog"."tagline",
#   (
#     SELECT
#       U0."headline"
#     FROM
#       "blog_entry" U0
#     WHERE
#       U0."blog_id" = ("blog_blog"."id")
#     ORDER BY
#       U0."pub_date" DESC
#     LIMIT
#       1
#   ) AS "most_recent_headline"
# FROM
#   "blog_blog"
# LIMIT
#   5;


# [(<Blog: Robinson-Wilson>, 'Three space maintain subject much.'),
#  (<Blog: Anderson PLC>, 'Rock authority enjoy hundred reduce behavior.'),
#  (<Blog: Mcneil PLC>, 'Visit beyond base.'),
#  (<Blog: Smith, Baker and Rodriguez>, 'Tree look culture minute affect.'),
#  (<Blog: George-Bray>, 'Then produce tree quality top similar.')]
```

# Select less data

## `values()` (part 1)

Select the specified fields only, as dictionaries. You can also select annotations, like this:

```python
Entry.objects.annotate(num_authors=Count("authors")).values(
    "rating", "num_authors"
)[:5]
# SELECT
#   "blog_entry"."rating",
#   COUNT("blog_entry_authors"."author_id") AS "num_authors"
# FROM "blog_entry" LEFT OUTER JOIN "blog_entry_authors" ...;

# [{'num_authors': 3, 'rating': 1},
#  {'num_authors': 4, 'rating': 4},
#  {'num_authors': 4, 'rating': 2},
#  {'num_authors': 3, 'rating': 2},
#  {'num_authors': 3, 'rating': 5}]
```

## `exists()`

If you need a yes/no answer that something exists, this is faster than fetching the whole row.

```python
unrealistic_data_exists = Entry.objects.filter(
    mod_date__lt=F("pub_date")
).exists()
# SELECT
#  (1) AS "a"
#   FROM "blog_entry"
#   WHERE "blog_entry"."mod_date" < ("blog_entry"."pub_date")
#   LIMIT 1;

unrealistic_data_exists
# True
```

## `only()`

Like `values()`, but you get a model instance back instead of a dictionary. Be warned, though: you will make an extra DB query if you access any of the fields you didn't fetch originally.

# Grouping data

## `values()` (part 2) and `GROUP BY`

The Django docs contain [this hidden gem](https://docs.djangoproject.com/en/dev/topics/db/aggregation/#values):

> "Ordinarily, annotations are generated on a per-object basis - an annotated QuerySet will return one result for each object in the original QuerySet. However, when a `values()` clause is used to constrain the columns that are returned in the result set... the original results are grouped according to the unique combinations of the fields specified in the `values()` clause. An annotation is then provided for each unique group; the annotation is computed over all members of the group."

The word "group" is in there because this is how you make a `GROUP BY` clause with Django's ORM. Very useful for reports.

Which day has the highest average blog entry rating?

```python
[
    (str(e["pub_date"]), e["avg_rating"])
    for e in Entry.objects.values("pub_date").annotate(avg_rating=Avg("rating"))
]
# SELECT
#   "blog_entry"."pub_date",
#   AVG("blog_entry"."rating") AS "avg_rating"
# FROM "blog_entry"
# GROUP BY "blog_entry"."pub_date";

# [
#  ('2022-04-07', 2.235294117647059),
#  ('2022-04-08', 2.8157894736842106),
#  ('2022-04-09', 2.0285714285714285),
#  ('2022-04-10', 2.96875),
#  ('2022-04-11', 2.3636363636363638),
#  ('2022-04-12', 2.725),
#  ('2022-04-13', 2.5),
#  ('2022-04-14', 2.761904761904762),
#  ...
# ]
```

## `aggregate()`

Like `annotate()`, but instead of adding a value for each row it reduces the query to a single row.
