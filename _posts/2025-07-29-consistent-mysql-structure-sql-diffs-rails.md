---
layout: post
title: Consistent MySQL structure.sql Diffs for Rails
excerpt: "A guide on how to remove noise from `structure.sql` diffs in Rails when working with MySQL."
---

<a href="#recipe">Jump to recipe ↓</a>

---

Having worked most of my Rails career with PostgreSQL and `schema.rb` files, when I had to switch to MySQL and `structure.sql` for a new project, the proverb _"We don't appreciate what we have until it's gone"_ grinned at me again.

It grinned because I hadn't appreciated that, when running a migration, only the relevant part of the structure dump would change. Nor had I appreciated the `db:schema:dump` Rake task (and the now obsolete `db:structure:dump`) which would yield consistent results, unaffected by local data.

With MySQL, the story is different. For one, the structure dump contains the pesky `AUTO_INCREMENT` option for each table. The value of this option depends on the data at the time of the dump, and there's no flag to exclude it from the output. If multiple people work on the same codebase, they can get different dumps, despite running the same migrations. This bug has been [open since June 2006](https://bugs.mysql.com/bug.php?id=20786), way back when I had just finished first grade (times which I now appreciate).<sup>[^1]</sup>

Then, not everybody uses the same MySQL client. For example, I prefer local development and use MySQL installed via Homebrew, which ships with [`mysqldump`](https://dev.mysql.com/doc/refman/8.4/en/mysqldump.html). Some colleagues work in containerized environments [that come with MariaDB](https://github.com/rails/rails/blob/40a3f2fedc12e70750e2b14ad096c135e7bb1df7/.devcontainer/Dockerfile#L9) and, by extension, [`mariadb-dump`](https://mariadb.com/docs/server/clients-and-utilities/backup-restore-and-import-clients/mariadb-dump). The output of `mysqldump` and `mariadb-dump` is similar, but not identical.

On the new project, the way to commit `structure.sql` was to fish out your schema changes from the diff soup, which looked a bit like this:

{% highlight patch %}
{% raw %}
--- a/db/structure.sql
+++ b/db/structure.sql
@@ -1 +1 @@
-/*!999999\- enable the sandbox mode */
+
@@ -5 +5 @@
-/*!40101 SET NAMES utf8mb4 */;
+/*!50503 SET NAMES utf8mb4 */;
@@ -2075 +2075 @@ CREATE TABLE `a` (
-/*!40101 SET character_set_client = utf8mb4 */;
+/*!50503 SET character_set_client = utf8mb4 */;
@@ -2105 +2105 @@ CREATE TABLE `b` (
-/*!40101 SET character_set_client = utf8mb4 */;
+/*!50503 SET character_set_client = utf8mb4 */;
@@ -2116 +2116 @@ CREATE TABLE `c` (
-) ENGINE=InnoDB DEFAULT CHARSET=utf8mb3;
+) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8mb3;
@@ -2132 +2132 @@ CREATE TABLE `d` (
-/*!40101 SET character_set_client = utf8mb4 */;
+/*!50503 SET character_set_client = utf8mb4 */;
@@ -2151 +2151 @@ CREATE TABLE `e` (
-) ENGINE=InnoDB DEFAULT CHARSET=utf8mb3;
+) ENGINE=InnoDB AUTO_INCREMENT=10 DEFAULT CHARSET=utf8mb3;
@@ -2160 +2160 @@ CREATE TABLE `f` (
-/*!40101 SET character_set_client = utf8mb4 */;
+/*!50503 SET character_set_client = utf8mb4 */;
@@ -2169 +2169 @@ CREATE TABLE `g` (
-/*!40101 SET character_set_client = utf8mb4 */;
+/*!50503 SET character_set_client = utf8mb4 */;
...
{% endraw %}
{% endhighlight %}

These diffs would repeat for all tables. More than once, they have caused accidentally committed lines and bad merge conflict resolutions. We kind of accepted the fact that `db/structure.sql` was to be found in unstaged changes at all times, even if your work didn't touch the schema.

I don't remember exactly which straw broke this camel's back, but at some point I became determined to:

1. resolve all differences between `mysqldump`'s and `mariadb-dump`'s outputs, and
2. make structure dumps idempotent (i.e., dump the same schema twice, get the same output),

so that migrations update only the parts of the structure dump you intend to change and nothing else. The solution turned out to be normalizing the structure dump with some regexes. It's a simple recipe, but it goes a long way.

## Recipe

<div id="recipe"></div>

{% highlight ruby %}
{% raw %}
# lib/tasks/db/schema/dump.rake
Rake::Task['db:schema:dump'].enhance do
  structure_sql_path = Rails.root.join('db/structure.sql')

  if File.exist?(structure_sql_path)
    sql = File.read(structure_sql_path)

    # see https://dev.mysql.com/doc/refman/8.4/en/example-auto-increment.html
    sql.gsub!(/ AUTO_INCREMENT=[0-9]+/, '')

    # see https://mariadb.org/mariadb-dump-file-compatibility-change/
    sql.gsub!(/^.+enable the sandbox mode.+$\R/, '')

    # mariadb-dump prints the former, mysqldump prints the latter
    sql.gsub!('/*!40101 SET NAMES utf8mb4 */;', '/*!50503 SET NAMES utf8mb4 */;')
    sql.gsub!('/*!40101 SET character_set_client = utf8mb4 */;', '/*!50503 SET character_set_client = utf8mb4 */;')

    File.write(structure_sql_path, sql)
  end
end
{% endraw %}
{% endhighlight %}

With this script, when you invoke `bundle exec rails db:schema:dump`, Rails will first create a new `structure.sql` file, and then it will run the block.

The logic first checks if `db/structure.sql` exists. This is a precaution in case the dump fails for whatever reason, so that we don't try to operate on a nonexistent file. If it does exist, we normalize it:

### Removing AUTO_INCREMENT

The first step is to remove all traces of the `AUTO_INCREMENT` option:

{% highlight ruby %}
{% raw %}
sql.gsub!(/ AUTO_INCREMENT=[0-9]+/, '')
{% endraw %}
{% endhighlight %}

which will in practice adjust all `CREATE TABLE` statements like so:

{% highlight patch %}
{% raw %}
 CREATE TABLE `foo` (
   `id` int NOT NULL AUTO_INCREMENT,
   ...
   PRIMARY KEY (`id`)
-) ENGINE=InnoDB AUTO_INCREMENT=123 DEFAULT CHARSET=utf8mb3;
+) ENGINE=InnoDB DEFAULT CHARSET=utf8mb3;
{% endraw %}
{% endhighlight %}

[`AUTO_INCREMENT=value`](https://dev.mysql.com/doc/refman/8.4/en/example-auto-increment.html) tells the DB the next value for the `AUTO_INCREMENT` column. In the above example, `id` is that column, and the current value is `123`. When we insert a new row in table `foo` (and don't set the `id` explicitly), its `id` will be `123`. After insertion, the `AUTO_INCREMENT` value will increment by one, becoming `AUTO_INCREMENT=124`.

As you can see, this is a data-dependent option, so it's not that useful in our schema. Removing it will implicitly set the value to `1` when you load the schema (e.g., in your test environment), which is great.

### Removing MariaDB's sandbox mode comment

If you've worked with MariaDB before, you may have noticed this line at the top of your `structure.sql`:

{% highlight sql %}
{% raw %}
/*!999999\- enable the sandbox mode */
{% endraw %}
{% endhighlight %}

You won't find this with MySQL. To resolve the difference, the script removes the line:

{% highlight ruby %}
{% raw %}
sql.gsub!(/^.+enable the sandbox mode.+$\R/, '')
{% endraw %}
{% endhighlight %}

The line we removed does just as it says: it enables the [sandbox mode](https://mariadb.com/docs/server/clients-and-utilities/mariadb-client/mariadb-command-line-client#sandbox), which disables ["any command that could do something on the shell"](https://mariadb.org/mariadb-dump-file-compatibility-change/). If you trust your dump file, the line won't do much for you, so there's no harm done in removing it.

### Adjusting conditional executable comments

The last step is to normalize executable comments:

{% highlight ruby %}
{% raw %}
sql.gsub!('/*!40101 SET NAMES utf8mb4 */;', '/*!50503 SET NAMES utf8mb4 */;')
sql.gsub!('/*!40101 SET character_set_client = utf8mb4 */;', '/*!50503 SET character_set_client = utf8mb4 */;')
{% endraw %}
{% endhighlight %}

The exact replacements depend on your MySQL and MariaDB versions, but let me first briefly explain how these comments work.

For example, this comment:

{% highlight sql %}
{% raw %}
/*!40101 SET NAMES utf8mb4 */;
{% endraw %}
{% endhighlight %}

will execute the statement `SET NAMES utf8mb4` only if your MySQL/MariaDB server version is greater than or equal to `4.1.1` (version format after `!` is `Mmmrr`, which stands for **M**ajor version, **m**inor version, and **r**elease number). More on this [here](https://dev.mysql.com/doc/refman/8.4/en/comments.html).

Structure dumps include a bunch of these comments, but the problem is that MariaDB and MySQL differ in the version specified in the comment, and sometimes even in the statement that's executed.

On my project, MySQL will print out `/*!50503 SET NAMES utf8mb4 */;`, while MariaDB prints out `/*!40101 SET NAMES utf8mb4 */;`. To resolve the difference, I've opted to adjust MariaDB's comment so it matches the MySQL one. Instead of executing on version 4.1.1 and above, the statement will now execute on v5.5.3 and above.

This is acceptable for us because our versions of MySQL and MariaDB are both greater than 5.5.3. If you're not sure which server version you're running, you can check this through the Rails console:

{% highlight ruby %}
{% raw %}
ActiveRecord::Base.connection.select_value('SELECT VERSION()')
{% endraw %}
{% endhighlight %}

The `SET character_set_client = utf8mb4` statement is treated in a similar manner. Unfortunately, I can't guarantee that the examples I've shown here will apply to your use case (since these comments change with MySQL/MariaDB versions), but what I've hopefully shown is how you can resolve the differences on your own.

## Outro

With the script in place, run `db:schema:dump` and commit the normalized structure dump. The next time a migration runs, `structure.sql` diff should include only the changes that actually need to be committed.

Make sure to also run `db:schema:dump` when you update MySQL or MariaDB to a newer version, to verify the script still works. If there's an unwanted diff (most likely due to new conditional comments), a small adjustment to the script should hopefully remove it.

As for my project, I can't even imagine going back to the old way of dealing with persistent diff noise. In this case at least, I appreciate the present more.

---

#### Footnotes

[^1]: An individual decided to bake this bug report a cake, which you can witness in [this mildly concerning video](https://www.youtube.com/watch?v=oAiVsbXVP6k).