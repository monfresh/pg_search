# pg_search

http://github.com/Casecommons/pg_search/

[<img src="https://secure.travis-ci.org/Casecommons/pg_search.png?branch=master"
alt="Build Status" />](http://travis-ci.org/Casecommons/pg_search)
[<img src="https://gemnasium.com/Casecommons/pg_search.png" alt="Dependency Status"
/>](https://gemnasium.com/Casecommons/pg_search)
[<img src="https://codeclimate.com/github/Casecommons/pg_search.png"
/>](https://codeclimate.com/github/Casecommons/pg_search) 
[<img src="https://coveralls.io/repos/Casecommons/pg_search/badge.png?branch=master"
alt="Coverage Status" />](https://coveralls.io/r/Casecommons/pg_search) 
[<img src="https://badge.fury.io/rb/pg_search.png" alt="Gem Version"
/>](http://badge.fury.io/rb/pg_search)

## DESCRIPTION

PgSearch builds named scopes that take advantage of PostgreSQL's full text
search.

Read the blog post introducing PgSearch at http://pivotallabs.com/pg-search/

## REQUIREMENTS

*   Ruby 1.9.2 or later
*   Active Record 3.1 or later
*   PostgreSQL
*   [PostgreSQL contrib packages for certain
    features](https://github.com/Casecommons/pg_search/wiki/Installing-Postgres-Contrib-Modules)


## INSTALL

    gem install pg_search

### Rails 3.1 or later, Ruby 1.9.2 or later

In Gemfile

    gem 'pg_search'

### Rails 3.0

The newest versions of PgSearch no longer support Rails 3.0. However, the 0.5
series still works. It's not actively maintained, but submissions are welcome
for backports and bugfixes.

    gem 'pg_search', "~> 0.5.7"

The 0.5 branch lives at
https://github.com/Casecommons/pg_search/tree/0.5-stable

### Rails 2

The newest versions of PgSearch no longer support Rails 2. However, the 0.2
series still works. It's not actively maintained, but submissions are welcome
for backports and bugfixes.

    gem 'pg_search', "~> 0.2.0"

The 0.2 branch lives at
https://github.com/Casecommons/pg_search/tree/0.2-stable

### Other Active Record projects

In addition to installing and requiring the gem, you may want to include the
PgSearch rake tasks in your Rakefile. This isn't necessary for Rails projects,
which gain the Rake tasks via a Railtie.

    load "pg_search/tasks.rb"

### Ruby 1.8.7 or earlier

The newest versions of PgSearch no longer support Ruby 1.8.7. However, the 0.6
series still works. It's not actively maintained, but submissions are welcome
for backports and bugfixes.

    gem 'pg_search', "~> 0.6.4"

The 0.6 branch lives at
https://github.com/Casecommons/pg_search/tree/0.6-stable

## USAGE

To add PgSearch to an Active Record model, simply include the PgSearch module.

    class Shape < ActiveRecord::Base
      include PgSearch
    end

### Multi-search vs. search scopes

pg_search supports two different techniques for searching, multi-search and
search scopes.

The first technique is multi-search, in which records of many different Active
Record classes can be mixed together into one global search index across your
entire application. Most sites that want to support a generic search page will
want to use this feature.

The other technique is search scopes, which allow you to do more advanced
searching against only one Active Record class. This is more useful for
building things like autocompleters or filtering a list of items in a faceted
search.

### Multi-search

#### Setup

Before using multi-search, you must generate and run a migration to create the
pg_search_documents database table.

    $ rails g pg_search:migration:multisearch
    $ rake db:migrate

#### multisearchable

To add a model to the global search index for your application, call
multisearchable in its class definition.

    class EpicPoem < ActiveRecord::Base
      include PgSearch
      multisearchable :against => [:title, :author]
    end

    class Flower < ActiveRecord::Base
      include PgSearch
      multisearchable :against => :color
    end

If this model already has existing records, you will need to reindex this
model to get existing records into the pg_search_documents table. See the
rebuild task below.

Whenever a record is created, updated, or destroyed, an Active Record callback
will fire, leading to the creation of a corresponding PgSearch::Document
record in the pg_search_documents table. The :against option can be one or
several methods which will be called on the record to generate its search
text.

You can also pass a Proc or method name to call to determine whether or not a
particular record should be included.

    class Convertible < ActiveRecord::Base
      include PgSearch
      multisearchable :against => [:make, :model],
                      :if => :available_in_red?
    end

    class Jalopy < ActiveRecord::Base
      include PgSearch
      multisearchable :against => [:make, :model],
                      :if => lambda { |record| record.model_year > 1970 }
    end

Note that the Proc or method name is called in an after_save hook. This means
that you should be careful when using Time or other objects. In the following
example, if the record was last saved before the published_at timestamp, it
won't get listed in global search at all until it is touched again after the
timestamp.

    class AntipatternExample
      include PgSearch
      multisearchable :against => [:contents],
                      :if => :published?

      def published?
        published_at < Time.now
      end
    end

    problematic_record = AntipatternExample.create!(
      :contents => "Using :if with a timestamp",
      :published_at => 10.minutes.from_now
    )

    problematic_record.published?     # => false
    PgSearch.multisearch("timestamp") # => No results

    sleep 20.minutes

    problematic_record.published?     # => true
    PgSearch.multisearch("timestamp") # => No results

    problematic_record.save!

    problematic_record.published?     # => true
    PgSearch.multisearch("timestamp") # => Includes problematic_record

#### Multi-search associations

Two associations are built automatically. On the original record, there is a
has_one :pg_search_document association pointing to the PgSearch::Document
record, and on the PgSearch::Document record there is a belongs_to :searchable
polymorphic association pointing back to the original record.

    odyssey = EpicPoem.create!(:title => "Odyssey", :author => "Homer")
    search_document = odyssey.pg_search_document #=> PgSearch::Document instance
    search_document.searchable #=> #<EpicPoem id: 1, title: "Odyssey", author: "Homer">

#### Searching in the global search index

To fetch the PgSearch::Document entries for all of the records that match a
given query, use PgSearch.multisearch.

    odyssey = EpicPoem.create!(:title => "Odyssey", :author => "Homer")
    rose = Flower.create!(:color => "Red")
    PgSearch.multisearch("Homer") #=> [#<PgSearch::Document searchable: odyssey>]
    PgSearch.multisearch("Red") #=> [#<PgSearch::Document searchable: rose>]

#### Chaining method calls onto the results

PgSearch.multisearch returns an ActiveRecord::Relation, just like scopes do,
so you can chain scope calls to the end. This works with gems like Kaminari
that add scope methods. Just like with regular scopes, the database will only
receive SQL requests when necessary.

    PgSearch.multisearch("Bertha").limit(10)
    PgSearch.multisearch("Juggler").where(:searchable_type => "Occupation")
    PgSearch.multisearch("Alamo").page(3).per_page(30)
    PgSearch.multisearch("Diagonal").find_each do |document|
      puts document.searchable.updated_at
    end

#### Configuring multi-search

PgSearch.multisearch can be configured using the same options as
`pg_search_scope` (explained in more detail below). Just set the
PgSearch.multisearch_options in an initializer:

    PgSearch.multisearch_options = {
      :using => [:tsearch, :trigram],
      :ignoring => :accents
    }

#### Rebuilding search documents for a given class

If you change the :against option on a class, add multisearchable to a class
that already has records in the database, or remove multisearchable from a
class in order to remove it from the index, you will find that the
pg_search_documents table could become out-of-sync with the actual records in
your other tables.

The index can also become out-of-sync if you ever modify records in a way that
does not trigger Active Record callbacks. For instance, the #update_attribute
instance method and the .update_all class method both skip callbacks and
directly modify the database.

To remove all of the documents for a given class, you can simply delete all of
the PgSearch::Document records.

    PgSearch::Document.delete_all(:searchable_type => "Animal")

To regenerate the documents for a given class, run:

    PgSearch::Multisearch.rebuild(Product)

This is also available as a Rake task, for convenience.

    $ rake pg_search:multisearch:rebuild[BlogPost]

A second optional argument can be passed to specify the PostgreSQL schema
search path to use, for multi-tenant databases that have multiple
pg_search_documents tables. The following will set the schema search path to
"my_schema" before reindexing.

    $ rake pg_search:multisearch:rebuild[BlogPost,my_schema]

For models that are multisearchable :against methods that directly map to
Active Record attributes, an efficient single SQL statement is run to update
the pg_search_documents table all at once. However, if you call any dynamic
methods in :against, the following strategy will be used:

    PgSearch::Document.delete_all(:searchable_type => "Ingredient")
    Ingredient.find_each { |record| record.update_pg_search_document }

You can also provide a custom implementation for rebuilding the documents by
adding a class method called `rebuild_pg_search_documents` to your model.

    class Movie < ActiveRecord::Base
      belongs_to :director

      def director_name
        director.name
      end

      multisearchable against: [:name, :director_name]

      # Naive approach
      def self.rebuild_pg_search_documents
        find_each { |record| record.update_pg_search_document }
      end

      # More sophisticated approach
      def self.rebuild_pg_search_documents
        connection.execute <<-SQL
         INSERT INTO pg_search_documents (searchable_type, searchable_id, content, created_at, updated_at)
           SELECT 'Movie' AS searchable_type,
                  movies.id AS searchable_id,
                  (movies.name || ' ' || directors.name) AS content,
                  now() AS created_at,
                  now() AS updated_at
           FROM movies
           LEFT JOIN directors
             ON directors.id = movies.director_id
        SQL
      end
    end

#### Disabling multi-search indexing temporarily

If you have a large bulk operation to perform, such as importing a lot of
records from an external source, you might want to speed things up by turning
off indexing temporarily. You could then use one of the techniques above to
rebuild the search documents off-line.

    PgSearch.disable_multisearch do
      Movie.import_from_xml_file(File.open("movies.xml"))
    end

### pg_search_scope

You can use pg_search_scope to build a search scope. The first parameter is a
scope name, and the second parameter is an options hash. The only required
option is :against, which tells pg_search_scope which column or columns to
search against.

#### Searching against one column

To search against a column, pass a symbol as the :against option.

    class BlogPost < ActiveRecord::Base
      include PgSearch
      pg_search_scope :search_by_title, :against => :title
    end

We now have an ActiveRecord scope named search_by_title on our BlogPost model.
It takes one parameter, a search query string.

    BlogPost.create!(:title => "Recent Developments in the World of Pastrami")
    BlogPost.create!(:title => "Prosciutto and You: A Retrospective")
    BlogPost.search_by_title("pastrami") # => [#<BlogPost id: 2, title: "Recent Developments in the World of Pastrami">]

#### Searching against multiple columns

Just pass an Array if you'd like to search more than one column.

    class Person < ActiveRecord::Base
      include PgSearch
      pg_search_scope :search_by_full_name, :against => [:first_name, :last_name]
    end

Now our search query can match either or both of the columns.

    person_1 = Person.create!(:first_name => "Grant", :last_name => "Hill")
    person_2 = Person.create!(:first_name => "Hugh", :last_name => "Grant")

    Person.search_by_full_name("Grant") # => [person_1, person_2]
    Person.search_by_full_name("Grant Hill") # => [person_1]

#### Dynamic search scopes

Just like with Active Record named scopes, you can pass in a Proc object that
returns a hash of options. For instance, the following scope takes a parameter
that dynamically chooses which column to search against.

Important: The returned hash must include a :query key. Its value does not
necessary have to be dynamic. You could choose to hard-code it to a specific
value if you wanted.

    class Person < ActiveRecord::Base
      include PgSearch
      pg_search_scope :search_by_name, lambda do |name_part, query|
        raise ArgumentError unless [:first, :last].include?(name_part)
        {
          :against => name_part,
          :query => query
        }
      end
    end

    person_1 = Person.create!(:first_name => "Grant", :last_name => "Hill")
    person_2 = Person.create!(:first_name => "Hugh", :last_name => "Grant")

    Person.search_by_name :first, "Grant" # => [person_1]
    Person.search_by_name :last, "Grant" # => [person_2]

#### Searching through associations

It is possible to search columns on associated models. Note that if you do
this, it will be impossible to speed up searches with database indexes.
However, it is supported as a quick way to try out cross-model searching.

In PostgreSQL 8.3 and earlier, you must install a utility function into your
database. To generate and run a migration for this, run:

    $ rails g pg_search:migration:associated_against
    $ rake db:migrate

This migration is safe to run against newer versions of PostgreSQL as well. It
will essentially do nothing.

You can pass a Hash into the :associated_against option to set up searching
through associations. The keys are the names of the associations and the value
works just like an :against option for the other model. Right now, searching
deeper than one association away is not supported. You can work around this by
setting up a series of :through associations to point all the way through.

    class Cracker < ActiveRecord::Base
      has_many :cheeses
    end

    class Cheese < ActiveRecord::Base
    end

    class Salami < ActiveRecord::Base
      include PgSearch

      belongs_to :cracker
      has_many :cheeses, :through => :cracker

      pg_search_scope :tasty_search, :associated_against => {
        :cheeses => [:kind, :brand],
        :cracker => :kind
      }
    end

    salami_1 = Salami.create!
    salami_2 = Salami.create!
    salami_3 = Salami.create!

    limburger = Cheese.create!(:kind => "Limburger")
    brie = Cheese.create!(:kind => "Brie")
    pepper_jack = Cheese.create!(:kind => "Pepper Jack")

    Cracker.create!(:kind => "Black Pepper", :cheeses => [brie], :salami => salami_1)
    Cracker.create!(:kind => "Ritz", :cheeses => [limburger, pepper_jack], :salami => salami_2)
    Cracker.create!(:kind => "Graham", :cheeses => [limburger], :salami => salami_3)

    Salami.tasty_search("pepper") # => [salami_1, salami_2]

### Searching using different search features

By default, pg_search_scope uses the built-in [PostgreSQL text
search](http://www.postgresql.org/docs/current/static/textsearch-intro.html).
If you pass the :using option to pg_search_scope, you can choose alternative
search techniques.

    class Beer < ActiveRecord::Base
      include PgSearch
      pg_search_scope :search_name, :against => :name, :using => [:tsearch, :trigram, :dmetaphone]
    end

The currently implemented features are

*   :tsearch - [Full text search](http://www.postgresql.org/docs/current/static/textsearch-intro.html) 
    (built-in with 8.3 and later, available as a contrib package for some earlier versions)
*   :trigram - [Trigram search](http://www.postgresql.org/docs/current/static/pgtrgm.html), which
    requires the trigram contrib package
*   :dmetaphone - [Double Metaphone search](http://www.postgresql.org/docs/9.0/static/fuzzystrmatch.html#AEN124771), which requires the fuzzystrmatch contrib package


#### :tsearch (Full Text Search)

PostgreSQL's built-in full text search supports weighting, prefix searches,
and stemming in multiple languages.

##### Weighting
Each searchable column can be given a weight of "A", "B", "C", or "D". Columns
with earlier letters are weighted higher than those with later letters. So, in
the following example, the title is the most important, followed by the
subtitle, and finally the content.

    class NewsArticle < ActiveRecord::Base
      include PgSearch
      pg_search_scope :search_full_text, :against => {
        :title => 'A',
        :subtitle => 'B',
        :content => 'C'
      }
    end

You can also pass the weights in as an array of arrays, or any other structure
that responds to #each and yields either a single symbol or a symbol and a
weight. If you omit the weight, a default will be used.

    class NewsArticle < ActiveRecord::Base
      include PgSearch
      pg_search_scope :search_full_text, :against => [
        [:title, 'A'],
        [:subtitle, 'B'],
        [:content, 'C']
      ]
    end

    class NewsArticle < ActiveRecord::Base
      include PgSearch
      pg_search_scope :search_full_text, :against => [
        [:title, 'A'],
        {:subtitle => 'B'},
        :content
      ]
    end

##### :prefix (PostgreSQL 8.4 and newer only)

PostgreSQL's full text search matches on whole words by default. If you want
to search for partial words, however, you can set :prefix to true. Since this
is a :tsearch-specific option, you should pass it to :tsearch directly, as
shown in the following example.

    class Superhero < ActiveRecord::Base
      include PgSearch
      pg_search_scope :whose_name_starts_with,
                      :against => :name,
                      :using => {
                        :tsearch => {:prefix => true}
                      }
    end

    batman = Superhero.create :name => 'Batman'
    batgirl = Superhero.create :name => 'Batgirl'
    robin = Superhero.create :name => 'Robin'

    Superhero.whose_name_starts_with("Bat") # => [batman, batgirl]

##### :dictionary

PostgreSQL full text search also support multiple dictionaries for stemming.
You can learn more about how dictionaries work by reading the [PostgreSQL
documention](http://www.postgresql.org/docs/current/static/textsearch-dictionaries.html). 
If you use one of the language dictionaries, such as "english",
then variants of words (e.g. "jumping" and "jumped") will match each other. If
you don't want stemming, you should pick the "simple" dictionary which does
not do any stemming. If you don't specify a dictionary, the "simple"
dictionary will be used.

    class BoringTweet < ActiveRecord::Base
      include PgSearch
      pg_search_scope :kinda_matching,
                      :against => :text,
                      :using => {
                        :tsearch => {:dictionary => "english"}
                      }
      pg_search_scope :literally_matching,
                      :against => :text,
                      :using => {
                        :tsearch => {:dictionary => "simple"}
                      }
    end

    sleepy = BoringTweet.create! :text => "I snoozed my alarm for fourteen hours today. I bet I can beat that tomorrow! #sleepy"
    sleeping = BoringTweet.create! :text => "You know what I like? Sleeping. That's what. #enjoyment"
    sleeper = BoringTweet.create! :text => "Have you seen Woody Allen's movie entitled Sleeper? Me neither. #boycott"

    BoringTweet.kinda_matching("sleeping") # => [sleepy, sleeping, sleeper]
    BoringTweet.literally_matching("sleeping") # => [sleeping]

##### :normalization

PostgreSQL supports multiple algorithms for ranking results against queries.
For instance, you might want to consider overall document size or the distance
between multiple search terms in the original text. This option takes an
integer, which is passed directly to PostgreSQL. According to the latest
[PostgreSQL documentation](http://www.postgresql.org/docs/current/static/textsearch-controls.html),
the supported algorithms are:

    0 (the default) ignores the document length
    1 divides the rank by 1 + the logarithm of the document length
    2 divides the rank by the document length
    4 divides the rank by the mean harmonic distance between extents
    8 divides the rank by the number of unique words in document
    16 divides the rank by 1 + the logarithm of the number of unique words in document
    32 divides the rank by itself + 1

This integer is a bitmask, so if you want to combine algorithms, you can add
their numbers together. (e.g. to use algorithms 1, 8, and 32, you would pass 1
+ 8 + 32 = 41)

    class BigLongDocument < ActiveRecord::Base
      include PgSearch
      pg_search_scope :regular_search,
                      :against => :text

      pg_search_scope :short_search,
                      :against => :text,
                      :using => {
                        :tsearch => {:normalization => 2}
                      }

    long = BigLongDocument.create!(:text => "Four score and twenty years ago")
    short = BigLongDocument.create!(:text => "Four score")

    BigLongDocument.regular_search("four score") #=> [long, short]
    BigLongDocument.short_search("four score") #=> [short, long]

##### :any_word

Setting this attribute to true will perform a search which will return all
models containing any word in the search terms.

    class Number < ActiveRecord::Base
      include PgSearch
      pg_search_scope :search_any_word,
                      :against => :text,
                      :using => {
                        :tsearch => {:any_word => true}
                      }

      pg_search_scope :search_all_words,
                      :against => :text
    end

    one = Number.create! :text => 'one'
    two = Number.create! :text => 'two'
    three = Number.create! :text => 'three'

    Number.search_any_word('one two three') # => [one, two, three]
    Number.search_all_words('one two three') # => []

#### :dmetaphone (Double Metaphone soundalike search)

[Double Metaphone](http://en.wikipedia.org/wiki/Double_Metaphone) is an
algorithm for matching words that sound alike even if they are spelled very
differently. For example, "Geoff" and "Jeff" sound identical and thus match.
Currently, this is not a true double-metaphone, as only the first metaphone is
used for searching.

Double Metaphone support is currently available as part of the [fuzzystrmatch
contrib package](http://www.postgresql.org/docs/current/static/fuzzystrmatch.html)
that must be installed before this feature can be used. In addition to the
contrib package, you must install a utility function into your database. To
generate and run a migration for this, run:

    $ rails g pg_search:migration:dmetaphone
    $ rake db:migrate

The following example shows how to use :dmetaphone.

    class Word < ActiveRecord::Base
      include PgSearch
      pg_search_scope :that_sounds_like,
                      :against => :spelling,
                      :using => :dmetaphone
    end

    four = Word.create! :spelling => 'four'
    far = Word.create! :spelling => 'far'
    fur = Word.create! :spelling => 'fur'
    five = Word.create! :spelling => 'five'

    Word.that_sounds_like("fir") # => [four, far, fur]

#### :trigram (Trigram search)

Trigram search works by counting how many three-letter substrings (or
"trigrams") match between the query and the text. For example, the string
"Lorem ipsum" can be split into the following trigrams:

    [" Lo", "Lor", "ore", "rem", "em ", "m i", " ip", "ips", "psu", "sum", "um ", "m  "]

Trigram search has some ability to work even with typos and misspellings in
the query or text.

Trigram support is currently available as part of the [pg_trgm contrib
package](http://www.postgresql.org/docs/current/static/pgtrgm.html) that must
be installed before this feature can be used.

    class Website < ActiveRecord::Base
      include PgSearch
      pg_search_scope :kinda_spelled_like,
                      :against => :name,
                      :using => :trigram
    end

    yahooo = Website.create! :name => "Yahooo!"
    yohoo = Website.create! :name => "Yohoo!"
    gogle = Website.create! :name => "Gogle"
    facebook = Website.create! :name => "Facebook"

    Website.kinda_spelled_like("Yahoo!") # => [yahooo, yohoo]

### Ignoring accent marks (PostgreSQL 9.0 and newer only)

Most of the time you will want to ignore accent marks when searching. This
makes it possible to find words like "piñata" when searching with the query
"pinata". If you set a pg_search_scope to ignore accents, it will ignore
accents in both the searchable text and the query terms.

Ignoring accents uses the [unaccent contrib
package](http://www.postgresql.org/docs/current/static/unaccent.html) that
must be installed before this feature can be used.

    class SpanishQuestion < ActiveRecord::Base
      include PgSearch
      pg_search_scope :gringo_search,
                      :against => :word,
                      :ignoring => :accents
    end

    what = SpanishQuestion.create(:word => "Qué")
    how_many = SpanishQuestion.create(:word => "Cuánto")
    how = SpanishQuestion.create(:word => "Cómo")

    SpanishQuestion.gringo_search("Que") # => [what]
    SpanishQuestion.gringo_search("Cüåñtô") # => [how_many]

Advanced users may wish to add indexes for the expressions that pg_search
generates. Unfortunately, the unaccent function supplied by this contrib
package is not indexable (as of PostgreSQL 9.1). Thus, you may want to write
your own wrapper function and use it instead. This can be configured by
calling the following code, perhaps in an initializer.

    PgSearch.unaccent_function = "my_unaccent"

### Using tsvector columns

PostgreSQL allows you the ability to search against a column with type
tsvector instead of using an expression; this speeds up searching dramatically
as it offloads creation of the tsvector that the tsquery is evaluated against.

To use this functionality you'll need to do a few things:

*   Create a column of type tsvector that you'd like to search against. If you
    want to search using multiple search methods, for example tsearch and
    dmetaphone, you'll need a column for each.
*   Create a trigger function that will update the column(s) using the
    expression appropriate for that type of search. See:
    [the PostgreSQL documentation for text search triggers](http://www.postgresql.org/docs/current/static/textsearch-features.html#TEXTSEARCH-UPDATE-TRIGGERS)
*   Should you have any pre-existing data in the table, update the
    newly-created tsvector columns with the expression that your trigger
    function uses.
*   Add the option to pg_search_scope, e.g:

        pg_search_scope :fast_content_search,
                        :against => :content,
                        :using => {
                          dmetaphone: {
                            tsvector_column: 'tsvector_content_dmetaphone'
                          },
                          tsearch: {
                            dictionary: 'english',
                            tsvector_column: 'tsvector_content_tsearch'
                          }
                          trigram: {} # trigram does not use tsvectors
                        }


Please note that the :against column is only used when the tsvector_column is
not present for the search type.

### Configuring ranking and ordering

#### :ranked_by (Choosing a ranking algorithm)

By default, pg_search ranks results based on the :tsearch similarity between
the searchable text and the query. To use a different ranking algorithm, you
can pass a :ranked_by option to pg_search_scope.

    pg_search_scope :search_by_tsearch_but_rank_by_trigram,
                    :against => :title,
                    :using => [:tsearch],
                    :ranked_by => ":trigram"

Note that :ranked_by using a String to represent the ranking expression. This
allows for more complex possibilities. Strings like ":tsearch", ":trigram",
and ":dmetaphone" are automatically expanded into the appropriate SQL
expressions.

    # Weighted ranking to balance multiple approaches
    :ranked_by => ":dmetaphone + (0.25 * :trigram)"

    # A more complex example, where books.num_pages is an integer column in the table itself
    :ranked_by => "(books.num_pages * :trigram) + (:tsearch / 2.0)"

#### :order_within_rank (Breaking ties)

PostgreSQL does not guarantee a consistent order when multiple records have
the same value in the ORDER BY clause. This can cause trouble with pagination.
Imagine a case where 12 records all have the same ranking value. If you use a
pagination library such as [kaminari](https://github.com/amatsuda/kaminari) or
[will_paginate](https://github.com/mislav/will_paginate) to return results in
pages of 10, then you would expect to see 10 of the records on page 1, and the
remaining 2 records at the top of the next page, ahead of lower-ranked
results.

But since there is no consistent ordering, PostgreSQL might choose to
rearrange the order of those 12 records between different SQL statements. You
might end up getting some of the same records from page 1 on page 2 as well,
and likewise there may be records that don't show up at all.

pg_search fixes this problem by adding a second expression to the ORDER BY
clause, after the :ranked_by expression explained above. By default, the
tiebreaker order is ascending by id.

    ORDER BY [complicated :ranked_by expression...], id ASC

This might not be desirable for your application, especially if you do not
want old records to outrank new records. By passing an :order_within_rank, you
can specify an alternate tiebreaker expression. A common example would be
descending by updated_at, to rank the most recently updated records first.

    pg_search_scope :search_and_break_ties_by_latest_update,
                    :against => [:title, :content],
                    :order_within_rank => "blog_posts.updated_at DESC"

#### PgSearch#pg_search_rank (Reading a record's rank as a Float)

It may be useful or interesting to see the rank of a particular record. This
can be helpful for debugging why one record outranks another. You could also
use it to show some sort of relevancy value to end users of an application.
Just call .pg_search_rank on a record returned by a pg_search_scope.

    shirt_brands = ShirtBrand.search_by_name("Penguin")
    shirt_brands[0].pg_search_rank #=> 0.0759909
    shirt_brands[1].pg_search_rank #=> 0.0607927

## ATTRIBUTIONS

PgSearch would not have been possible without inspiration from texticle (now renamed
[textacular](https://github.com/textacular/textacular)). Thanks to [Aaron
Patterson](http://tenderlovemaking.com/) for the original version!

## CONTRIBUTIONS AND FEEDBACK

Welcomed! Feel free to join and contribute to our [public Pivotal Tracker
project](https://www.pivotaltracker.com/projects/228645) where we manage new
feature ideas and bugs.

We also have a [Google Group](http://groups.google.com/group/casecommons-dev)
for discussing pg_search and other Case Commons open source projects.

Please read our [CONTRIBUTING guide](https://github.com/Casecommons/pg_search/blob/master/CONTRIBUTING.md).

## LICENSE

MIT