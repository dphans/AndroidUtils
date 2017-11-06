# MediaStore

Scan for music, playlists, retrieve songs from playlists from external storage.

In basic, we'll have 2 models: `PlaylistModel` and `SongModel`:

- PlaylistModel will have property to store songs as an array.
- SongModel store some audio's metadata, like artist, title, track number, composer,...
- Both models will have `_id`, `date_created`, and `date_modified`.

**So, in best practices, we need to create a `BaseModel`, `SongModel` and `PlaylistModel` will inherit from `BaseModel`. So, `BaseModel` maybe design like below:**

### `BaseModel` class

```kotlin
open class Model: Serializable {

    @SerializedName("_id")
    var _id: Long           = System.currentTimeMillis()

    @SerializedName("_createdAt")
    var _createdAt: Long    = System.currentTimeMillis()

    @SerializedName("_updatedAt")
    var _updatedAt: Long    = System.currentTimeMillis()

}
```
---

I often use `Gson` to convert data class as a Json (string) format, so I add `Gson`'s dependency in `build.gradle` file (app level):

```json
	ext.retrofit_version = '2.3.0'
	implementation "com.squareup.retrofit2:retrofit:$retrofit_version"
	implementation "com.squareup.retrofit2:converter-gson:$retrofit_version"
```
---

In `BaseModel` class, add this line to convert data in class as a Json by override `toString` method:


### `BaseModel` class
```kotlin
	override fun toString(): String {
        return try {
            Gson().toJson(this@Jsonable)
        } catch (exception: Exception) {
            Timber.e(exception)
            "{}"
        }
    }
```
---

As you can see, If converter can't convert class into Json, we will return empty Json object. You can do something for this exception but this will better for my project.


### `PlaylistModel` class
```kotlin
class PlaylistModel: BaseModel() {

    @SerializedName("songs")
    var songs: Array<SongModel> = emptyArray()

    @SerializedName("name")
    var name: String            = String()

}
```
---

### `SongModel` class
```kotlin
class SongModel: BaseModel() {

    @SerializedName("title")
    var title: String   = String()

    @SerializedName("artist")
    var artist: String      = String()

    @SerializedName("album")
    var album: String       = String()

    @SerializedName("composer")
    var composer: String?   = null

    @SerializedName("year")
    var year: String?   = null

    @SerializedName("track")
    var track: Long     = 0

    @SerializedName("duration")
    var duration: Long  = 0

    @SerializedName("size")
    var size: Long      = 0

    @SerializedName("path")
    var path: String?   = null

}
```
---


Now, you can use helper below to scan songs, playlists from external storage. Data will be fill into 2 models we created above:

### `MediaHelper` class
```kotlin
@Suppress("unused", "MemberVisibilityCanPrivate")
class MediaHelper(private val context: Context) {

    private val contentResolver     = this@MediaHelper.context.contentResolver

    companion object {
        fun with(context: Context)  = MediaHelper(context.applicationContext)
    }

    fun scanForSongs(): List<SongModel> {
        val musicUri    = MediaStore.Audio.Media.EXTERNAL_CONTENT_URI
        val cursor      = this@MediaHelper.contentResolver.query(
                musicUri,
                null,
                "${MediaStore.Audio.Media.IS_MUSIC} != 0",
                null,
                "${AudioColumns.TITLE} ASC, ${AudioColumns.ARTIST} ASC"
        )
        return this@MediaHelper.scanSongWithCursor(cursor)
    }

    fun scanForPlaylists(): List<PlaylistModel> {
        val playlists   = mutableListOf<PlaylistModel>()
        val uri         = MediaStore.Audio.Playlists.EXTERNAL_CONTENT_URI
        val cursor      = this@MediaHelper.contentResolver.query(
                uri,
                null,
                null,
                null,
                null
        )
        if (cursor != null) {
            if (cursor.moveToFirst()) {
                val columnIdxId     = cursor.getColumnIndex(MediaStore.Audio.Playlists._ID)
                val columnIdxName   = cursor.getColumnIndex(MediaStore.Audio.Playlists.NAME)
                val columnIdxCreatedAt  = cursor.getColumnIndex(MediaStore.Audio.Playlists.DATE_ADDED)
                val columnIdxUpdatedAt  = cursor.getColumnIndex(MediaStore.Audio.Playlists.DATE_MODIFIED)
                do {
                    val playlistItem    = PlaylistModel()
                    playlistItem._id        = cursor.getLong(columnIdxId)
                    playlistItem.name       = cursor.getString(columnIdxName) ?: ""
                    playlistItem.songs      = this@MediaHelper.getSongsFromPlaylistById(playlistItem._id).toTypedArray()
                    playlistItem._createdAt = cursor.getLong(columnIdxCreatedAt)
                    playlistItem._updatedAt = cursor.getLong(columnIdxUpdatedAt)
                    playlists.add(playlistItem)
                } while (cursor.moveToNext())
            }
            cursor.close()
        }
        return playlists
    }

    fun getSongsFromPlaylistById(playlistId: Long): List<SongModel> {
        val songUri = MediaStore.Audio.Playlists.Members.getContentUri("external", playlistId)
        val cursor  = this@MediaHelper.contentResolver.query(
                songUri,
                null,
                "${MediaStore.Audio.Media.IS_MUSIC} != 0",
                null,
                null
        )
        return this@MediaHelper.scanSongWithCursor(cursor, true)
    }

    private fun scanSongWithCursor(cursor: Cursor?, isFromPlaylist: Boolean = false): List<SongModel> {
        val songs   = mutableListOf<SongModel>()
        if (cursor != null) {
            if (cursor.moveToFirst()) {
                val columnIdxId         = cursor.getColumnIndex(if (isFromPlaylist) MediaStore.Audio.Playlists.Members._ID else MediaStore.Audio.Media._ID)
                val columnIdxTitle      = cursor.getColumnIndex(if (isFromPlaylist) MediaStore.Audio.Playlists.Members.TITLE else MediaStore.Audio.Media.TITLE)
                val columnIdxArtist     = cursor.getColumnIndex(if (isFromPlaylist) MediaStore.Audio.Playlists.Members.ARTIST else MediaStore.Audio.Media.ARTIST)
                val columnIdxAlbum      = cursor.getColumnIndex(if (isFromPlaylist) MediaStore.Audio.Playlists.Members.ALBUM else MediaStore.Audio.Media.ALBUM)
                val columnIdxYear       = cursor.getColumnIndex(if (isFromPlaylist) MediaStore.Audio.Playlists.Members.YEAR else MediaStore.Audio.Media.YEAR)
                val columnIdxTrack      = cursor.getColumnIndex(if (isFromPlaylist) MediaStore.Audio.Playlists.Members.TRACK else MediaStore.Audio.Media.TRACK)
                val columnIdxComposer   = cursor.getColumnIndex(if (isFromPlaylist) MediaStore.Audio.Playlists.Members.COMPOSER else MediaStore.Audio.Media.COMPOSER)
                val columnIdxDuration   = cursor.getColumnIndex(if (isFromPlaylist) MediaStore.Audio.Playlists.Members.DURATION else MediaStore.Audio.Media.DURATION)
                val columnIdxSize       = cursor.getColumnIndex(if (isFromPlaylist) MediaStore.Audio.Playlists.Members.SIZE else MediaStore.Audio.Media.SIZE)
                val columnIdxPath       = cursor.getColumnIndex(if (isFromPlaylist) MediaStore.Audio.Playlists.Members.DATA else MediaStore.Audio.Media.DATA)
                val columnIdxCreatedAt  = cursor.getColumnIndex(if (isFromPlaylist) MediaStore.Audio.Playlists.Members.DATE_ADDED else MediaStore.Audio.Media.DATE_ADDED)
                val columnIdxUpdatedAt  = cursor.getColumnIndex(if (isFromPlaylist) MediaStore.Audio.Playlists.Members.DATE_MODIFIED else MediaStore.Audio.Media.DATE_MODIFIED)
                do {
                    val songItem        = SongModel()
                    songItem._id        = cursor.getLong(columnIdxId)
                    songItem.title      = cursor.getString(columnIdxTitle) ?: ""
                    songItem.artist     = cursor.getString(columnIdxArtist) ?: ""
                    songItem.album      = cursor.getString(columnIdxAlbum) ?: ""
                    songItem.year       = cursor.getString(columnIdxYear) ?: ""
                    songItem.track      = cursor.getLong(columnIdxTrack)
                    songItem.composer   = cursor.getString(columnIdxComposer) ?: ""
                    songItem.duration   = cursor.getLong(columnIdxDuration)
                    songItem.size       = cursor.getLong(columnIdxSize)
                    songItem.path       = cursor.getString(columnIdxPath) ?: ""
                    songItem._createdAt = cursor.getLong(columnIdxCreatedAt)
                    songItem._updatedAt = cursor.getLong(columnIdxUpdatedAt)
                    songs.add(songItem)
                } while (cursor.moveToNext())
            }
            cursor.close()
        }
        return songs
    }

}
```
---

- You can get songs by call: `MediaHelper.with(<context>).scanForSongs(): List<SongModel>`
- Get playlists: `MediaHelper.with(<context>).scanForPlaylists(): List<PlaylistModel>`

This class missing somethings... I will add later like: edit playlist, edit song information, add/remove songs into/from playlist item,...
