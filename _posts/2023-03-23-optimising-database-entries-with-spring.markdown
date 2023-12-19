---
layout: post
title: "Optimising Database entries with spring/hibernate"
---
Welcome to my blog post! My name is Bernhard Ballek and I recently finished a 10-month Java programming course at [everyone codes](https://everyonecodes.io/). In order to practice our newly-gained skills in Java and Spring Boot, we had 6 weeks to work on a small project of our choice. I decided to clone [LastFm](https://www.last.fm) which has some features behind a paywall. Futhermore the statistics are predefined and the user can't define them themself.

The biggest problem was to optimise how the relationships got saved in my Database. I know, you may think that optimising something that is still in developement isn't smart since you don't know if it's needed. Nevertheless in this project I think it's one of the most important things to do as early as possible.

## Requirements
In the finished program:
* The user should be able to add, update and delete song reviews.
* The songs the user listened to should be saved in the Database.
* Each song has up to 5 Genre associated with it.
* The user should be able to set a few parameters and then get statistics related to the songs he/she heard.

## The problem

Here are the entities that are related to the problem. I left the constructors out to save some space.

###Song Entity
```java
@Entity
@NoArgsConstructor
@Getter
@Setter
public class Song {

    @Id
    @GeneratedValue
    @JsonIgnore
    private Long songId;

    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    @Column(nullable = false)
    @ManyToMany(mappedBy = "songs", cascade = CascadeType.MERGE)
    private List<Account> accountId;

    @Column(nullable = false, columnDefinition = "TEXT")
    private String title;

    @Column(nullable = false, columnDefinition = "TEXT")
    private String artistName;

    @ManyToMany(cascade = CascadeType.MERGE)
    @Column(nullable = false)
    @JoinTable(joinColumns = {@JoinColumn(referencedColumnName = "songId", name = "song_id")}
            ,inverseJoinColumns = {@JoinColumn(referencedColumnName = "genreId", name = "genre_id")})
    private List<Genre> genres;

    @Column(nullable = false)
    private int durationInMilliseconds;

    @Column(nullable = false)
    private LocalDateTime listenedOn;
}
```

###Account Entity
```java
@Entity
@NoArgsConstructor
@Getter
@Setter
public class Account {

    @Id
    @GeneratedValue
    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    private Long accountId;

    @Column(nullable = false, length = 20, unique = true)
    private String name;

    @JsonIgnore
    @Column(nullable = false, length = 30)
    private String password;

    @JsonIgnore
    @Column(columnDefinition = "TEXT", unique = true)
    private String spotifyAuthenticationToken;

    @JsonIgnore
    @ManyToMany(cascade = CascadeType.MERGE)
    @JoinTable(joinColumns = {@JoinColumn(referencedColumnName = "accountId", name = "account_id")}
            ,inverseJoinColumns = {@JoinColumn(referencedColumnName = "songId", name = "song_id")})
    private List<Song> songs;
}
```

###Genre Entity
```java
@Entity
@NoArgsConstructor
@Getter
@Setter
public class Genre {

    @Id
    @GeneratedValue
    @JsonIgnore
    private Long genreId;

    @JsonIgnore
    @ManyToMany(mappedBy = "genres")
    private List<Song> songs;

    @Column(nullable = false, unique = true)
    private String name;
}
```

Understanding the `@ManyToMany` and `@ManyToOne` relations took some time. It works now and I think I could explain them.
Now in order to save songs I started to write the endpoint in order to achieve that. It looked like this:

```java
public void saveSongs(List<Song> songs) {
    for (Song song : songs) {
        var genres = song.getGenres(); //original List of the song
        Set<String> genreAsStrings = new ArrayList<>(genres).stream()
                .map(Genre::getName)
                .collect(Collectors.toSet());  //Working copy of List as String

        Set<Genre> existingGenres = findGenreIfExists(genreAsStrings);

        if (!existingGenres.isEmpty()) { //Set Genre id for song's genres so db gets the connection right
            for (Genre genre : genres) {
                var oExistingGenre = existingGenres.stream()
                        .filter(e -> genre.getName().equals(e.getName()))
                        .findFirst();
                oExistingGenre.ifPresent(e -> genre.setGenreId(e.getGenreId()));
            }
        }

        var genreToAdd = genres.stream() //Filters all genres that don't exist
                .filter(e -> !existingGenres.stream()
                        .map(Genre::getName)
                        .collect(Collectors.toSet())
                        .contains(e.getName()))
                .toList();

        saveAllGenre(genreToAdd); //Adds all genre which didn't exist till now
        saveOne(song);
        accountService.getAccountReference(song.getAccountId().get(0).getAccountId()).getSongs().add(song);
    }
}
```

You may have noticed that there is logic so the genres are not duplicated in the Database. Now The reason for optimising that and the problem I mentioned at the beginning:

* One `Song` can have 5 `Genres` (I decided to limit it to 5 because any more will be unreliable)
* One `User` can listen to around 15-20 `Songs` per hour
* Ideally there are more than one `User` using my program

This would fill the `Genre` Table with multiple entries in no time. This wouldn't be such a problem for performance, but with simple java logic it's solvable. Now what would happen if you add a `Song` to a `User`?

### The real problem
The big no-go is the line `accountService.getAccountReference(song.getAccountId().get(0).getAccountId()).getSongs().add(song);` getting a reference is a good way to only load what you need. Although in this case we have to get the List of `Songs` the user heard `.getSongs()` in order to add a new one to it. That is a problem since the object would be huge and loaded each time a song is saved. This line alone would slow the whole process down. I researched a lot because I thought Hibernate can do that somehow using an easy insert sql query. I haven't found anything for a while until I stumbeled upon a stackoverflow post which said with hibernate alone it's not possible.

## The solution
Then I researched for a solution that actually works. Which are native sql statements and they look like this:

```java
    public interface AccountRepository extends JpaRepository<Account, Long> {

    @Transactional
    @Modifying
    @Query(value = "INSERT INTO account_songs VALUES (:accountId, :songId)", nativeQuery = true)
    void insertAccountSong(@Param("accountId") Long accountId, @Param("songId") Long songId);

}
```

I felt kind of bad to invest that many hours to get to that solution which are a few lines of code. I also felt really good finally solving it.


Thanks for reading my blog, and I hope it helped you!