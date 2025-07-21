
# 📁 Android File Uploader Library Design

## 📌 Use Case

Reusable Android library to upload files (images, videos, docs) with:

- Progress tracking
- Retry on failure
- Parallel/queued uploads
- Background support
- Multipart/form-data support

---

## ✅ 1. Clarifying Questions

- Only `multipart/form-data` or raw uploads too?
- Upload to own backend or S3/GCS?
- File size limits?
- Background uploads (WorkManager/ForegroundService)?
- Auth headers needed?
- Queue vs parallel uploads?
- Resumable/chunked uploads?

---

## 📐 2. High-Level Architecture

```mermaid
flowchart TD
    A[App Calls FileUploader.enqueue()] --> B[UploadManager]
    B --> C{Is Running?}
    C -- No --> D[Start Foreground UploadWorker]
    D --> E[UploadTask]
    E --> F[ProgressListener + RetryHandler]
    F --> G[NetworkManager -> Server]
    G --> H[Callback to App]
```

---

## 📦 3. Components

### 3.1 Public API

```kotlin
interface FileUploader {
    fun enqueue(request: UploadRequest, callback: UploadCallback)
    fun cancel(uploadId: String)
    fun getStatus(uploadId: String): UploadStatus
}
```

---

### 3.2 UploadRequest

```kotlin
data class UploadRequest(
    val file: File,
    val uploadUrl: String,
    val headers: Map<String, String> = emptyMap(),
    val metadata: Map<String, String> = emptyMap(),
    val uploadId: String = UUID.randomUUID().toString()
)
```

---

### 3.3 UploadCallback

```kotlin
interface UploadCallback {
    fun onProgress(uploadId: String, progress: Int)
    fun onSuccess(uploadId: String, response: String)
    fun onFailure(uploadId: String, error: Throwable)
}
```

---

### 3.4 UploadManager

```kotlin
class UploadManager : FileUploader {

    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    private val queue = LinkedBlockingQueue<UploadRequest>()

    override fun enqueue(request: UploadRequest, callback: UploadCallback) {
        queue.add(request)
        scope.launch { startWorkerIfNeeded(callback) }
    }

    private suspend fun startWorkerIfNeeded(callback: UploadCallback) {
        while (queue.isNotEmpty()) {
            val request = queue.poll()
            UploadTask().upload(request, callback)
        }
    }

    override fun cancel(uploadId: String) { /* ... */ }

    override fun getStatus(uploadId: String): UploadStatus = UploadStatus.PENDING
}
```

---

### 3.5 UploadTask

```kotlin
class UploadTask {

    suspend fun upload(request: UploadRequest, callback: UploadCallback) {
        val client = OkHttpClient.Builder().build()

        val requestBody = MultipartBody.Builder()
            .setType(MultipartBody.FORM)
            .addFormDataPart("file", request.file.name,
                request.file.asRequestBody("application/octet-stream".toMediaTypeOrNull()))
            .build()

        val req = Request.Builder()
            .url(request.uploadUrl)
            .headers(Headers.of(request.headers))
            .post(ProgressRequestBody(requestBody) { progress ->
                callback.onProgress(request.uploadId, progress)
            })
            .build()

        try {
            val response = client.newCall(req).execute()
            if (response.isSuccessful) {
                callback.onSuccess(request.uploadId, response.body?.string().orEmpty())
            } else {
                callback.onFailure(request.uploadId, IOException("Upload failed with code ${response.code}"))
            }
        } catch (e: Exception) {
            callback.onFailure(request.uploadId, e)
        }
    }
}
```

---

### 3.6 ProgressRequestBody

```kotlin
class ProgressRequestBody(
    private val body: RequestBody,
    private val progressListener: (Int) -> Unit
) : RequestBody() {

    override fun contentType(): MediaType? = body.contentType()
    override fun contentLength(): Long = body.contentLength()

    override fun writeTo(sink: BufferedSink) {
        val countingSink = object : ForwardingSink(sink) {
            var totalBytesWritten = 0L
            override fun write(source: Buffer, byteCount: Long) {
                super.write(source, byteCount)
                totalBytesWritten += byteCount
                val percent = (100 * totalBytesWritten / contentLength()).toInt()
                progressListener(percent)
            }
        }
        val bufferedSink = countingSink.buffer()
        body.writeTo(bufferedSink)
        bufferedSink.flush()
    }
}
```

---

## 📊 Upload Status Enum

```kotlin
enum class UploadStatus {
    PENDING, IN_PROGRESS, SUCCESS, FAILED, CANCELLED
}
```

---

## 🚀 Optional Extensions

- ✅ RetryHandler with backoff
- 📦 WorkManager background support
- 🔄 Resumable chunked uploads
- 🔒 Encrypted file staging
- 🔔 Foreground notification
- 🧠 Room/DB for persistent queue

---

## 💡 Sample Usage

```kotlin
val uploader: FileUploader = UploadManager()

val request = UploadRequest(
    file = File("/sdcard/myimage.png"),
    uploadUrl = "https://myserver.com/upload",
    headers = mapOf("Authorization" to "Bearer token")
)

uploader.enqueue(request, object : UploadCallback {
    override fun onProgress(uploadId: String, progress: Int) {
        println("Progress: $progress%")
    }

    override fun onSuccess(uploadId: String, response: String) {
        println("Success: $response")
    }

    override fun onFailure(uploadId: String, error: Throwable) {
        println("Error: ${error.message}")
    }
})
```
