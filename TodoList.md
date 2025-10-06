# Materi: Menampilkan List Data Todo dari Firestore

Dokumen ini menjelaskan bagaimana basecode Anda mengambil dan menampilkan list data Todo dari Firebase Firestore menggunakan Kotlin Coroutines + RecyclerView. Bahasannya dibuat langkah demi langkah dan cocok untuk pemula.

Tujuan belajar:

- Memahami struktur komponen: Activity (UI), UseCase (akses data), Adapter (ikatan data ke view), dan Entity (model data).
- Mengerti alur pengambilan data Firestore secara asynchronous (coroutines) dan menampilkannya di RecyclerView.
- Tahu apa yang terjadi di balik layar ketika data diambil dan diikat ke tampilan.

---

## Peta file yang terlibat

- `res/layout/activity_todo.xml` — Layout Activity berisi RecyclerView + ProgressBar (wadah UI utama list).
- `res/layout/item_todo.xml` — Layout tampilan masing-masing item Todo.
- `entity/Todo.kt` — Data model Todo.
- `adapter/TodoAdapter.kt` — Adapter untuk mengikat data Todo ke `item_todo.xml`.
- `usecases/TodoUseCase.kt` — Mengambil data dari Firestore.
- `TodoActivity.kt` — Mengelola UI list, loading state, dan memicu pengambilan data.

---

## Alur besar (end-to-end)

1. `TodoActivity` diinisialisasi → siapkan RecyclerView dan Adapter.
2. `TodoActivity` memanggil `initializeData()` untuk mulai mengambil data.
3. Di dalam coroutine (`lifecycleScope.launch`), UI menampilkan loading dan menyembunyikan list.
4. `TodoUseCase.getTodo()` mengambil data dari koleksi Firestore `todo` → `await()` hasil network call.
5. Hasil query dipetakan ke list `List<Todo>`.
6. Kembali ke `TodoActivity`, loading disembunyikan, list ditampilkan.
7. Adapter di-update melalui `todoAdapter.updateData(list)` agar RecyclerView merender item.

---

## Langkah 1: Buat UI (XML)

Kita mulai dari tampilan terlebih dahulu, karena UI adalah wadah data yang nanti akan diisi kode Kotlin.

Bagian ini membedah `activity_todo.xml` dan `item_todo.xml` agar kamu paham peran tiap atribut dan bagaimana ViewBinding memetakan ID ke properti Kotlin.

### A) `activity_todo.xml`

Ringkasnya, layout ini menampilkan dua komponen utama: `RecyclerView` sebagai kontainer list dan `ProgressBar` sebagai indikator loading. Keduanya ditumpuk (overlap) karena memakai `RelativeLayout` sehingga kita bisa memperlihatkan salah satu dan menyembunyikan yang lain sesuai state.

Potongan penting:

```xml
<RelativeLayout
    android:id="@+id/main"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/container"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:paddingLeft="20dp"
        android:paddingRight="20dp"
        android:paddingBottom="10dp"
        android:paddingTop="10dp"
        tools:listitem="@layout/item_todo" />

    <ProgressBar
        android:id="@+id/ui_loading"
        android:layout_width="50dp"
        android:layout_height="50dp"
        android:layout_centerInParent="true"
        android:visibility="gone" />

    <com.google.android.material.floatingactionbutton.FloatingActionButton
        android:layout_width="50dp"
        android:layout_height="50dp"
        android:contentDescription="@string/navigate_to_create_todo"
        android:padding="20dp"
        android:layout_margin="20dp"
        android:src="@android:color/background_dark"
        android:layout_alignParentEnd="true"
        android:layout_alignParentBottom="true" />

</RelativeLayout>
```

Penjelasan atribut penting:

- `tools:listitem="@layout/item_todo"`: hanya untuk preview di Android Studio; tidak berpengaruh saat runtime. Membantu melihat bentuk tiap item saat desain.
- Padding pada RecyclerView: memberi jarak di kiri/kanan/atas/bawah agar item tidak menempel tepi layar.
- `ProgressBar` berada di tengah dengan `android:layout_centerInParent="true"` dan default `visibility="gone"` agar tidak terlihat kecuali saat loading.
- `FloatingActionButton` diposisikan di kanan bawah (`alignParentEnd` & `alignParentBottom`). Saat ini belum ada onClick di kode, tapi siap dipakai untuk navigasi ke layar tambah Todo.

Detail tambahan penting:

- Namespaces di root: `xmlns:android` wajib; `xmlns:app` untuk atribut library; `xmlns:tools` untuk atribut editor (tidak memengaruhi runtime). `tools:context=".TodoActivity"` membantu preview.
- ViewBinding: `@+id/container` → `binding.container` (RecyclerView), `@+id/ui_loading` → `binding.uiLoading` (ProgressBar).
- WindowInsets & root id `@id/main`: dipakai di `TodoActivity` untuk menambahkan padding sistem bar saat edge-to-edge.
- Toggling visibility saat runtime: `binding.container.visibility = View.GONE` dan `binding.uiLoading.visibility = View.VISIBLE` ketika loading; dibalik saat data siap.
- Layout params: `match_parent` untuk RecyclerView agar mengisi layar; `layout_centerInParent` untuk memusatkan ProgressBar.

Tips UI:

- Saat konten kosong (list kosong), tampilkan empty state (TextView) di tengah dan sembunyikan RecyclerView.
- Gunakan `@dimen` untuk padding agar konsisten dan mudah diubah.

### B) `item_todo.xml`

Layout item menggunakan `RelativeLayout` dengan dua `TextView`: judul dan deskripsi.

```xml
<RelativeLayout
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:padding="10dp"
    android:background="@drawable/rounded"
    android:layout_marginTop="10dp"
    android:layout_marginBottom="10dp">

    <TextView
        android:id="@+id/title"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Memancing di dermaga" />

    <TextView
        android:id="@+id/description"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="10dp"
        android:layout_below="@id/title"
        android:text="Contoh deskripsi"/>

</RelativeLayout>
```

Penjelasan atribut penting:

- `@drawable/rounded`: memberi latar berbentuk rounded card (cek file drawable terkait). Ini meningkatkan keterbacaan antara item.
- `android:layout_below="@id/title"`: memastikan deskripsi muncul di bawah judul.
- Margin atas/bawah pada item: memberi jarak antar item agar list terasa lega.

Saran aksesibilitas & i18n:

- Hindari teks contoh hardcoded untuk produksi; pindahkan ke `@string/...` bila perlu.
- Atur `android:textAppearance` (headline/body) agar konsisten lewat tema.

---

### Ringkasan alur setelah UI siap

Setelah layout XML siap, kita beralih ke kode Kotlin yang akan:

- mendefinisikan model `Todo`,
- mengambil data dari Firestore (`TodoUseCase`),
- menampilkan data di RecyclerView (`TodoAdapter`), dan
- mengontrol flow UI di `TodoActivity`.

## Detail per komponen

### 1) Entity: `Todo`

File: `entity/Todo.kt`

```kotlin
data class Todo(
    val id: String,
    val title: String,
    val description: String,
)
```

- Model data sederhana; field `id`, `title`, `description` dipakai untuk binding ke UI.

### 2) Use case akses data: `TodoUseCase`

File: `usecases/TodoUseCase.kt`

```kotlin
class TodoUseCase {
    val db = Firebase.firestore

    suspend fun getTodo(): List<Todo> {
        val data = db.collection("todo")
            .get()
            .await()

        if (!data.isEmpty) {
            return data.documents.map {
                Todo(
                    id = it.id,
                    title = it.getString("title").toString(),
                    description = it.getString("description").toString()
                )
            }
        }
        return arrayListOf()
    }
}
```

Yang terjadi di balik layar:

- `Firebase.firestore` adalah klien Firestore dari SDK Firebase.
- `db.collection("todo").get()` menjalankan query read (satu kali) untuk mengambil semua dokumen pada koleksi `todo`.
- `await()` adalah ekstensi coroutine dari `kotlinx-coroutines-play-services` yang menunggu hasil Task secara non-blocking.
- Dokumen yang diterima dikonversi ke list `Todo`.

Catatan:

- Error apapun akan dilempar ke pemanggil (Activity) sehingga bisa ditangani dan ditampilkan ke user.
- Ada juga method `getTodo(id: String)` untuk mengambil satu item berdasarkan `document(id)`.

### 3) Adapter RecyclerView: `TodoAdapter`

File: `adapter/TodoAdapter.kt`

```kotlin
class TodoAdapter(
    private val dataset: MutableList<Todo>
) : RecyclerView.Adapter<TodoAdapter.CustomViewHolder>() {
    // onCreateViewHolder -> inflate layout item_todo
    // onBindViewHolder -> bind data ke view holder
    // updateData -> ganti isi dataset dan notify
}
```

Poin penting:

- `ItemTodoBinding` dipakai (ViewBinding) untuk akses view tanpa `findViewById`.
- `bindData(item)` mengisi `title` dan `description` ke `TextView`.
- `updateData(newData)` membersihkan dataset, menambah data baru, lalu `notifyDataSetChanged()` agar RecyclerView merender ulang.

### 4) Activity UI: `TodoActivity`

File: `TodoActivity.kt`

- Set up RecyclerView:
  ```kotlin
  private fun setupRecyclerView() {
      todoAdapter = TodoAdapter(mutableListOf())
      binding.container.apply {
          adapter = todoAdapter
          layoutManager = LinearLayoutManager(this@TodoActivity)
      }
  }
  ```
- Ambil data dengan coroutine dan tampilkan loading:

  ```kotlin
  private fun initializeData() {
      lifecycleScope.launch {
          binding.container.visibility = View.GONE
          binding.uiLoading.visibility = View.VISIBLE

          try {
              val todoList = todoUseCase.getTodo()
              binding.uiLoading.visibility = View.GONE
              binding.container.visibility = View.VISIBLE
              todoAdapter.updateData(todoList)
          } catch (e: Exception) {
              Toast.makeText(this@TodoActivity, e.message, Toast.LENGTH_SHORT).show()
          }
      }
  }
  ```

  Yang terjadi di balik layar:

- `lifecycleScope.launch`: menjalankan coroutine yang otomatis mengikuti lifecycle Activity.
- Saat menunggu Firestore, UI menampilkan `ProgressBar` dan menyembunyikan list untuk pengalaman pengguna yang jelas.
- Setelah data datang, Adapter di-update sehingga RecyclerView merender item satu per satu sesuai `getItemCount()` dan `onBindViewHolder()`.

### 5) Catatan singkat tentang layout (pengingat)

- `activity_todo.xml`: berisi `RecyclerView` (id: `container`) dan `ProgressBar` (id: `ui_loading`).
- `item_todo.xml`: tampilan tiap item dengan dua `TextView`: `title` dan `description`.

---

## Troubleshooting umum

Catatan keamanan Firestore:

- Untuk kelas/praktikum, aturan default mungkin longgar (read/write: true). Untuk app nyata, batasi akses berdasarkan user (Firebase Auth) dan gunakan aturan berbasis dokumen/field.
- Contoh rule sederhana (jangan pakai di produksi tanpa penyesuaian):
  ```
  rules_version = '2';
  service cloud.firestore {
      match /databases/{database}/documents {
          match /todo/{id} {
              allow read: if true;
              allow write: if request.auth != null; // hanya user login boleh menulis
          }
      }
  }
  ```

---
