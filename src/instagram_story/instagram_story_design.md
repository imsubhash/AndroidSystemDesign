# 📱 Instagram Story Feed – Android System Design

---

## 1. Business Idea
Design an Instagram-like Story Feed where users can:
- Tap through image/video stories.
- Auto-play with progress indicators.
- Swipe between users’ story sets.
- Stories auto-delete in 24h.

---


## 2. Clarifying Questions
### 🧩 Feature Scope
- Are we building just the **story viewing experience**, or also **creation**, **uploading**, or **analytics**?
- Should the design support **image and video stories**, or only one type?
- Is this similar to **Instagram Stories**, or closer to **WhatsApp Status** / **Snapchat Discover**?
- Should we support **interactive elements** like polls, swipe-ups, or music?

### 💾 Offline & Caching
- Should users be able to **view stories offline**?
- Is **disk-based caching** (e.g. Room/MediaStore) expected?
- What should be the **cache eviction policy**?

### 🚀 Prefetching & Prioritization
- Should we prefetch stories of **all followed users**, or only a few?
- Should stories be fetched **in order of recency, engagement**, or **network quality**?
- Is there a **priority for certain users** (e.g., close friends)?

## 3. Requirements Clarification

### Functional
- Show stories in full screen with autoplay.
- Swipe between users, tap to advance.
- Progress indicator per story.
- Stories contain images/videos.
- Save story state (resume after interruption).
- Handle network, retry, and caching.

### Non-Functional
- Smooth 60fps experience.
- Prefetching and intelligent caching.
- Disk and memory optimization.
- Offline story replay.
- Minimal startup delay.

---

## 4. Optional Mathematical Model
- Story size ≈ 1–3 MB.
- Avg user watches 10 users × 4 stories = 40 stories = ~80–100MB.
- Target memory cache: 20–30MB.
- Target disk cache: 100–200MB.

---

## 5. Client–Server Split

### Server
- APIs:
    - `GET /stories`: fetches user's and friends’ stories.
    - `HEAD /media/{id}`: returns content type, size, etc.
- TTL-based expiry (24h window).
- CDN for media (images/videos).

### Client
- Story UI rendering (images, videos).
- Smart caching (memory/disk).
- Prefetch & media preload.
- Resume state from local.

---

## 6. API Design

```kotlin
interface StoryApi {
    @GET("/stories")
    suspend fun getStories(@Query("userId") userId: String): StoryFeedResponse
}

sealed class StoryMediaType { IMAGE, VIDEO }

data class Story(
    val id: String,
    val userId: String,
    val mediaUrl: String,
    val type: StoryMediaType,
    val postedAt: Long // for TTL
)
```

---

## 7. High-Level Android Architecture

```plaintext
StoryFeedFragment
   |-- ViewPager2 (Users)
         |-- RecyclerView (Stories per user)
   |-- StoryView (ImageView / PlayerView)
   |-- StoryViewModel (state)
   |-- PrefetchManager
   |-- StoryRepository
         |-- MemoryCache (LRU)
         |-- DiskCache (ExoPlayer / Coil)
         |-- StoryApi
```

---

## 8. Detailed Module: Prefetch + Cache

### 📦 Cache Strategy

#### Memory Cache
- LRU of currently visible + next 2 users.
- Evict on memory warning.

#### Disk Cache
- Image: Coil default disk cache.
- Video: ExoPlayer `SimpleCache`
- TTL: 24h expiry

```kotlin
val cache = SimpleCache(
    File(context.cacheDir, "video-cache"),
    LeastRecentlyUsedCacheEvictor(100 * 1024 * 1024),
    StandaloneDatabaseProvider(context)
)
```

---

### ⚡ Prefetching Logic

```kotlin
fun prefetchStories(userIds: List<String>) {
    viewModelScope.launch(Dispatchers.IO) {
        userIds.forEach { userId ->
            val stories = repository.getStories(userId)
            stories.forEach { story ->
                preloadMedia(story.mediaUrl, story.type)
            }
        }
    }
}

suspend fun preloadMedia(url: String, type: StoryMediaType) {
    when (type) {
        IMAGE -> imageLoader.enqueue(ImageRequest.Builder(context).data(url).build())
        VIDEO -> {
            val mediaItem = MediaItem.fromUri(url)
            val player = ExoPlayer.Builder(context).build()
            player.setMediaItem(mediaItem)
            player.prepare()
            player.release()
        }
    }
}
```

---

### 🧹 Eviction Policy
| Layer        | Policy               | Tool Used              |
|--------------|----------------------|------------------------|
| Memory Cache | LRU (30MB)           | Coil / ExoPlayer       |
| Disk Cache   | LRU + TTL (24h)      | Coil / SimpleCache     |
| State Cache  | SavedStateHandle     | Jetpack ViewModel      |

---

## 9. Tricky Case: Network + Resume State

### Problem
- User sees 2 stories → app killed → reopens → must resume at same place.

### Solution
- Persist currentUserId + currentStoryIndex in `SavedStateHandle` or local DB.
- On `onCreate()`, check saved state and scroll ViewPager2/RecyclerView accordingly.
- Replay stories using cache.

```kotlin
class StoryStateSaver(val savedStateHandle: SavedStateHandle) {
    fun save(userId: String, index: Int) {
        savedStateHandle["uid"] = userId
        savedStateHandle["index"] = index
    }

    fun restore(): Pair<String, Int>? {
        val uid = savedStateHandle.get<String>("uid") ?: return null
        val index = savedStateHandle.get<Int>("index") ?: 0
        return uid to index
    }
}
```

---

