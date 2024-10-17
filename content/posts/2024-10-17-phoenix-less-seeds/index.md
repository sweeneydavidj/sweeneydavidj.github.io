+++
title = "Phoenix Less Seeds"
author = ["david"]
date = 2024-10-17
tags = ["Phoenix"]
url = "phoenix-less-seeds"
draft = false
summary = "Reducing the burden of maintaining database seed scripts in Phoenix"
+++

## Reducing the burder of seeds.exs scripts {#reducing-the-burder-of-seeds-dot-exs-scripts}

At some point the migration files become so numerous that it is preferable to consolidate them into a `structure.sql` file so that they can be loaded using `mix ecto.load`. This is already well documented in this post from fly.io [Developing after mix ecto.dump](https://fly.io/phoenix-files/developing-after-a-mix-ecto-dump/). But what about the `seeds.exs` scripts?

In my current project there is a large set of what we refer to as "lookup" tables, they could also be referred to as a data dictionary. All the data shown in dropdown lists etc. comes from these lookup tables and they are (by convention in this project) prefixed with `lk_` and usually just contain an `ID` and a `name`. These "lookup" tables are populated by `seeds.exs` scripts. Over time the number and complexity of these scripts increased and they need to be maintained as items are re-ordered or certain items are no longer needed etc.

In order to reduce the maintenance burden all the seeds files were consolidated into the second part of `structure.sql`. The following shows the commands that were used to create the sql statements.

```bash
sudo -iu postgres
pg_dump --dbname=myapp_dev --file=myapp-dev.sql --table="lk_*" --data-only --inserts
```

That command can be run on a local dev machine against a clean database. The contents of the produced file `myapp-dev.sql` is then append to the `structure.sql` file.

And that's it - short and sweet but it gives us a new base to work from and less code to maintain - happy days!
