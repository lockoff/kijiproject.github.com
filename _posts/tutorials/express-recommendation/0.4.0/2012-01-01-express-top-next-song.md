---
layout: post
title: Top Next Songs
categories: [tutorials, express-recommendation, 0.4.0]
tags: [express-music]
order: 5
description: Find the most popular song played after this song.
---

<div id="accordion-container">
  <h2 class="accordion-header"> TopNextSongs.scala </h2>
  <div class="accordion-content">
    <script src="http://gist-it.appspot.com/github/kijiproject/kiji-express-music/raw/kiji-express-music-0.4.0/src/main/scala/org/kiji/express/music/TopNextSongs.scala"> </script>
  </div>
</div>

<h3 style="margin-top:0px;padding-top:10px;">Top Next Songs</h3>

Now, for each song, we want to compute a list of the songs that are most frequently played by users
after that song. This kind of model can eventually be used to write a song recommender.

The entire pipeline for this computation is a little more complex.  It looks like this:

{% highlight scala %}
KijiInput(args("users-table"))(Map(Column("info:track_plays", all) -> 'playlist))
    .flatMap('playlist -> ('firstSong, 'songId)) { bigrams }
    .groupBy(('firstSong, 'songId)) { _.size('count) }
    .pack[SongCount](('songId, 'count) -> 'songCount)
    .groupBy('firstSong) { sortNextSongs }
    .map('scalaTopSongs -> 'topSongs) { scalaListToJavaList }
    .pack[TopSongs]('topSongs -> 'topNextSongs)
    .map('firstSong -> 'entityId) { firstSong: String =>
        EntityId(args("songs-table"))(firstSong) }
    .write(KijiOutput(args("songs-table"))('topNextSongs -> "info:top_next_songs"))
{% endhighlight %}

There are some more user-defined functions defined at the top of the file that make this cleaner,
but let’s just walk through the pipeline line-by-line and examine those functions as they appear:

#### Read "info:track_plays" from a Kiji table

The first line reads all values in the “info:track_plays” column from the Kiji table “users-table”
into a tuple field called ‘playlist.  Recall that the “info:track_plays” column contains, for each
user row, each track played by that user, at the timestamp it was played.

{% highlight scala %}
    KijiInput(args("users-table"))(Map(Column("info:track_plays", all) -> 'playlist))
{% endhighlight %}

#### Transform songs histories into bigrams of songs

The next line maps ‘playlist (all songs a user has played) to bigrams of songs that were played in
that order.  `bigrams` is our first user-defined function in the pipeline, and it’s an
important one.

{% highlight scala %}
/**
 * Transforms a slice of song ids into a collection of tuples `(s1,
 * s2)` signifying that `s2` appeared after `s1` in the slice, chronologically.
 *
 * @param slice of song ids representing a user's play history.
 * @return a list of song bigrams.
 */
def bigrams(slice: KijiSlice[String]): List[(String, String)] = {
  slice.orderChronologically().cells.sliding(2)
      .map { itr => itr.iterator }
      .map { itr => (itr.next().datum, itr.next().datum) }
      .toList
}
{% endhighlight %}

`bigrams` takes as an argument a slice of the table, and transforms it into a collection of tuples.
Each tuple is a pair of songs where the second song was played directly after the first, that is,
the function outputs a sequence of every two adjacent songs in the input.  Bigrams are common in
many types of statistical analysis, see
[the Wikipedia article on bigrams](http://en.wikipedia.org/wiki/Bigram) for more.  We use a flatMap
since the bigrams method produces a list of song pairs from each playlist.

{% highlight scala %}
    .flatMap('playlist -> ('firstSong, 'songId)) { bigrams }
{% endhighlight %}

#### Count the occurrences of every unique song pair

To perform a count, the `groupBy` method will be used to count the unique occurrences of two songs
played in sequence.  For each group of tuples with the same 'firstSong and 'songId, we put the
size of that group in the 'count field.

{% highlight scala %}
    .groupBy(('firstSong, 'songId)) { _.size('count) }
{% endhighlight %}

#### Pack the pair of ‘songId and ‘count into a SongCount record

Next, a `SongCount` record is constructed containing the song played and the count associated
with it. Scalding's `pack` method is used to perform this operation. `pack` takes a mapping from
tuple fields to a field name to bind the resulting record to. Any fields specified must have
matching setters named `set<Fieldname>`.

{% highlight scala %}
    .pack[SongCount](('songId, 'count) -> 'songCount)
{% endhighlight %}

#### Sort the ‘nextSongs associated with each ‘firstSong

To sort the 'nextSongs associated with each 'firstSong, we groupBy the 'firstSongs and sort those
groups.

{% highlight scala %}
    .groupBy('firstSong) { sortNextSongs }
{% endhighlight %}

You've seen `groupBy` operations before, with functions inline such as `_.size('count)` earlier
in this script. Here, sortNextSongs is a UDF we've defined:

{% highlight scala %}
/**
 * Transforms a group of tuples into a group containing a list of song count records,
 * sorted by count.
 *
 * @param nextSongs is the group of tuples containing song count records.
 * @return a group containing a list of song count records, sorted by count.
 */
def sortNextSongs(nextSongs: GroupBuilder): GroupBuilder = {
  nextSongs.sortBy('count).reverse.toList[SongCount]('songCount -> 'scalaTopSongs)
}
{% endhighlight %}

A Scalding `GroupBuilder` is basically a group of named tuples, which have been grouped by
a field, and that you can operate on inside a grouping operation.  Here, we've grouped by the
'firstSong field, and we sort the group by the number of times they've been played. The result is
a list of songs in the order from most to least played, in the field 'scalaTopSongs.

We’re basically done!  ‘scalaTopSongs contains the data we want: the next top songs, sorted by
popularity, that follow any particular song.  The last few lines are the machinery required to put
that into our songs table.

#### Do some final processing on the tuples

Before we're done, we do a simple conversion of Scala Lists to Java Lists and pack that list into
the TopSongs record in the ‘topNextSongs field.  This uses a straightforward user-defined function
that converts Scala Lists to Java Lists:

{% highlight scala %}
/**
 * Transforms a Scala `List` into a Java `List`.
 *
 * @param ls is the list to transform.
 * @tparam T is the type of data in the list.
 * @return a Java `List` created from the original Scala `List`.
 */
def scalaListToJavaList[T](ls: List[T]): java.util.List[T] = Lists.newArrayList[T](ls.asJava)
{% endhighlight %}

{% highlight scala %}
    .map('scalaTopSongs -> 'topSongs) { scalaListToJavaList }
    .pack[TopSongs]('topSongs -> 'topNextSongs)
{% endhighlight %}

#### Finally, write the ‘topNextSongs field to the “info:top_next_songs” column in our table

We create entity IDs using the 'firstSong field and put it in the 'entityId, then write the
'topNextSongs field to the "info:top_next_songs" column in our table.

{% highlight scala %}
    .map('firstSong -> 'entityId) { firstSong: String =>
        EntityId(args("songs-table"))(firstSong) }
    .write(KijiOutput(args("songs-table"))('topNextSongs -> "info:top_next_songs"))
{% endhighlight %}

### Running TopNextSongs ###

<div class="userinput">
{% highlight bash %}
express job --libjars "${MUSIC_EXPRESS_HOME}/lib/*" \
    ${MUSIC_EXPRESS_HOME}/lib/kiji-express-music-0.4.0.jar \
    org.kiji.express.music.TopNextSongs --users-table ${KIJI}/users \
    --songs-table ${KIJI}/songs --hdfs
{% endhighlight %}
</div>

### Alternative: Running as a script ###

Or run it as a script:

<div class="userinput">
{% highlight bash %}
express script --libjars "${MUSIC_EXPRESS_HOME}/lib/*" \
    ${MUSIC_EXPRESS_HOME}/scripts/TopNextSongs.express --hdfs
{% endhighlight %}
</div>

### Verify the Output ###

<div class="userinput">
{% highlight bash %}
kiji scan ${KIJI}/songs --max-rows=2
{% endhighlight %}
</div>

You should see:

    Scanning kiji table: kiji://localhost:2181/kiji_express_music/songs/
    entity-id=['song-32'] [1365549351598] info:metadata
                                     {"song_name": "song name-32", "artist_name": "artist-2", "album_name": "album-0", "genre": "genre1.0", "tempo": 120, "duration": 180}
    entity-id=['song-32'] [1365550614616] info:top_next_songs
                                     {"topSongs": [{"song_id": "song-31", "count": 8}, {"song_id": "song-30", "count": 8}, {"song_id": "song-33", "count": 6}, {"song_id": "song-32", "count": 6}, {"song_id": "song-38", "count": 4}, {"song_id": "song-37", "count": 4}, {"song_id": "song-39", "count": 2}, {"song_id": "song-8", "count": 2}, {"song_id": "song-6", "count": 1}, {"song_id": "song-34", "count": 1}, {"song_id": "song-29", "count": 1}, {"song_id": "song-24", "count": 1}, {"song_id": "song-23", "count": 1}, {"song_id": "song-0", "count": 1}]}

    entity-id=['song-49'] [1365549353027] info:metadata
                                     {"song_name": "song name-49", "artist_name": "artist-3", "album_name": "album-1", "genre": "genre4.0", "tempo": 150, "duration": 180}
    entity-id=['song-49'] [1365550613461] info:top_next_songs
                                     {"topSongs": [{"song_id": "song-38", "count": 1}, {"song_id": "song-8", "count": 1}, {"song_id": "song-31", "count": 1}, {"song_id": "song-10", "count": 1}, {"song_id": "song-6", "count": 1}]}

Notice that for each of these songs, there is now a info:top_next_songs column that contains a record “topSongs”, the list of top songs played after each of these, in order of popularity.  We now have the data necessary for a song recommendation model based on the most popular next songs.

This concludes the tutorial of KijiExpress.  If you were using a bento cluster for this tutorial and are done with it, now is a good time shut it down:

<div class="userinput">
{% highlight bash %}
    bento stop
{% endhighlight %}
</div>
