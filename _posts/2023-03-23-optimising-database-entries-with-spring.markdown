---
layout: post
title: "Optimising Database entries with spring/hibernate"
---
My name is Bernhard Ballek and I decided to clone [LastFm](https://www.last.fm) which has some features behind a paywall. Futhermore the statistics are predefined and the user can't define them themself.

The biggest problem was to optimise how the relationships got saved in my Database. I know, you may think that optimising something that is still in developement isn't smart since you don't know if it's needed. Nevertheless in this project I think it's one of the most important things to do as early as possible.

## Requirements
In the finished program:
* The user should be able to add, update and delete song reviews.
* The songs the user listened to should be saved in the Database.
* Each song has up to 5 Genre associated with it.
* The user should be able to set a few parameters and then get statistics related to the songs he/she heard.

Here are the entities that are related to the problem. I simplified them to save some space.

### Song Entity
```java
public class Song {
    @Id
    @GeneratedValue
    @JsonIgnore
    private Long songId;

    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    @Column(nullable = false)
    @ManyToMany(mappedBy = "songs", cascade = CascadeType.MERGE)
    private List<Account> accountId;

    @ManyToMany(cascade = CascadeType.MERGE)
    @Column(nullable = false)
    @JoinTable(joinColumns = {@JoinColumn(referencedColumnName = "songId", name = "song_id")}
            ,inverseJoinColumns = {@JoinColumn(referencedColumnName = "genreId", name = "genre_id")})
    private List<Genre> genres;
}
```

### Account Entity
```java
public class Account {
    @Id
    @GeneratedValue
    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    private Long accountId;

    @JsonIgnore
    @ManyToMany(cascade = CascadeType.MERGE)
    @JoinTable(joinColumns = {@JoinColumn(referencedColumnName = "accountId", name = "account_id")}
            ,inverseJoinColumns = {@JoinColumn(referencedColumnName = "songId", name = "song_id")})
    private List<Song> songs;
}
```

### Genre Entity
```java
public class Genre {
    @Id
    @GeneratedValue
    @JsonIgnore
    private Long genreId;

    @JsonIgnore
    @ManyToMany(mappedBy = "genres")
    private List<Song> songs;
}
```

In order to save songs I started to write the endpoint in order to achieve that. I also took care that the `Genre` aren't duplicated in the Database via java logic to avoid filling it with unneccessary data.

```java
public void saveSongs(List<Song> songs) {
    var genres = song.getGenres(); //Original List of the song
    Set<Genre> existingGenres = findGenreIfExists(new ArrayList<>(genres).stream()
            .map(Genre::getName)
            .collect(Collectors.toSet())); //Passing a working copy of List as Strings

    if (!existingGenres.isEmpty()) { //Set Genre id for song's genres so db gets the connection right
        for (Genre genre : genres) {
            existingGenres.stream()
            .filter(e -> genre.getName().equals(e.getName()))
            .findFirst()
                    .ifPresent(e -> genre.setGenreId(e.getGenreId()));
        }
    }
    var genreToAdd = genres.stream() //Filters all genres that don't exist
            .filter(e -> !existingGenres.stream()
                    .map(Genre::getName)
                    .collect(Collectors.toSet())
                    .contains(e.getName()))
            .toList();

    saveAllGenre(genreToAdd); //Adds all genre which didn't exist till now
}
```

The lines to save the song in the Database and get the connection to the account right looked like this (again simplified):

```java
public void saveSongs(List<Song> songs) {
    for (Song song : songs) {
        saveOne(song);
        accountService.getAccountReference(song.getAccountId().get(0).getAccountId()).getSongs().add(song);
    }
}
```

## The problem
The big no-go is the line `accountService.getAccountReference(song.getAccountId().get(0).getAccountId()).getSongs().add(song);` getting a reference is a good way to only load what you need. Although in this case we have to get the List of `Songs` the user heard `.getSongs()` in order to add a new one to it. 
That is a problem since the object would be huge and loaded each time a song is saved. This line alone would slow the whole process down. 

I researched a lot because I thought Hibernate can do that somehow using an easy insert sql query. I haven't found anything for a while until I stumbeled upon a stackoverflow post which said with hibernate alone it's not possible.

## The solution
Then I researched for something that actually works. Which are native sql statements and they look like this:

```java
    public interface AccountRepository extends JpaRepository<Account, Long> {

    @Transactional
    @Modifying
    @Query(value = "INSERT INTO account_songs VALUES (:accountId, :songId)", nativeQuery = true)
    void insertAccountSong(@Param("accountId") Long accountId, @Param("songId") Long songId);

}
```

So the line `accountService.getAccountReference(song.getAccountId().get(0).getAccountId()).getSongs().add(song);` got changed into `accountService.insertAccountSong(accountId, song.getSongId());`. With that the whole `Song` list doesn't get queried everytime we want to save a `Song`.

Native query refers to plain sql syntax, while queries are written in JPQL.

I felt kind of bad investing many hours to get to that solution which are a few lines of code, though it also felt really good finally solving it!


Thanks for reading my blog!