# Memahami Google Sign-In + Firebase Auth

Dokumen ini menjelaskan langkah demi langkah bagaimana fitur Google Sign-In diimplementasikan, apa yang terjadi di balik layar, dan bagaimana komponen-komponennya saling terhubung.

Tujuan belajar:

- Mengerti arsitektur Google Sign-In modern (AndroidX Credential Manager + Google Identity Services) yang terhubung ke Firebase Authentication.
- Mampu menelusuri alur dari klik tombol hingga pengguna berhasil login.
- Paham token apa yang dipakai (ID Token), siapa yang memvalidasi, dan bagaimana Firebase “mengenal” pengguna.
- Siap melakukan debugging saat terjadi error umum (mis. NoCredentialException, salah Client ID, keystore belum terdaftar, dsb.).

---

## Gambaran arsitektur singkat

Komponen yang dipakai di proyek Anda:

- AndroidX Credential Manager: API standar baru untuk mengambil kredensial (termasuk Google) di Android.
- Google Identity Services (GIS): Library `googleid` untuk meminta Google ID Token (OpenID Connect).
- Firebase Authentication: Memvalidasi ID Token & membuat sesi pengguna Firebase (auth.currentUser).
- google-services plugin + `google-services.json`: Menghubungkan app dengan project Firebase Anda.

Alur sederhana:

1. User tekan tombol “Sign in with Google”.
2. App membangun GetCredentialRequest yang berisi opsi Google.
3. Credential Manager menampilkan UI akun Google (account picker / One Tap).
4. App menerima Google ID Token (bukan access token) dari GIS.
5. App tukarkan ID Token itu ke Firebase Authentication (signInWithCredential).
6. Firebase memverifikasi ke Google dan membuat sesi pengguna; `auth.currentUser` menjadi tidak null.
7. App navigasi ke halaman berikutnya.

---

## Cek dependensi dan konfigurasi (Gradle)

File: `app/build.gradle.kts`

Poin penting yang sudah benar di basecode Anda:

- Plugin Google Services (agar `google-services.json` diproses):
  ```kotlin
  plugins {
      alias(libs.plugins.android.application)
      alias(libs.plugins.kotlin.android)
      alias(libs.plugins.gms) // com.google.gms.google-services
  }
  ```
- Firebase Auth melalui BoM (mengatur versi konsisten):
  ```kotlin
  dependencies {
      implementation(platform(libs.firebase.bom))
      implementation(libs.firebase.auth)
  }
  ```
- Credential Manager + Integrasi Play Services + Google Identity Services:
  ```kotlin
  implementation(libs.androidx.credentials)
  implementation(libs.androidx.credentials.play.services.auth)
  implementation(libs.googleid)
  ```
- `google-services.json` sudah ada di `app/` (bagus). Pastikan file ini berasal dari project Firebase yang benar dan SHA-1/SHA-256 dari keystore debug/release sudah ditambahkan di Firebase Console.

Catatan versi ada di `gradle/libs.versions.toml` untuk keterlacakan.

---

## Melihat kode utama: `GoogleSignIn.kt`

File: `app/src/main/java/com/example/ppb_project/usecases/GoogleSignIn.kt`

Kelas ini membungkus seluruh proses login Google + Firebase menjadi use case yang sederhana dipanggil dari Activity.

### 1) Properti penting

- `auth: FirebaseAuth = Firebase.auth`
  - Objek utama untuk mengelola sesi pengguna di Firebase.
- `credentialManager: CredentialManager = CredentialManager.create(context)`
  - Pintu masuk ke sistem kredensial Android (menampilkan UI akun, mengambil token).
- `scope = CoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)`
  - Untuk menjalankan proses login secara asynchronous di main thread (UI-safe) dengan lifecycle cleanup.

### 2) Membangun request Google Sign-In

```kotlin
fun createInstanceGoogleSignInClient(clientId: String): GetCredentialRequest {
    val googleIdOption = GetGoogleIdOption.Builder()
        .setServerClientId(clientId)
        .setFilterByAuthorizedAccounts(false)
        .build()

    return GetCredentialRequest.Builder()
        .addCredentialOption(googleIdOption)
        .build()
}
```

Penjelasan:

- `setServerClientId(clientId)`: Ini adalah “Web Client ID”/“Server Client ID” dari Google Cloud/Firebase, digunakan agar Google menerbitkan ID Token dengan audience sesuai app Anda. ID Token ini yang nantinya ditukar ke Firebase.
- `setFilterByAuthorizedAccounts(false)`: Tampilkan semua akun Google yang ada di perangkat, tidak hanya akun yang pernah otorisasi.
- Hasil akhirnya adalah `GetCredentialRequest` yang akan dipakai Credential Manager untuk memunculkan UI sign-in.

### 3) Memulai proses sign-in

```kotlin
fun signIn(onSuccess: () -> Unit) {
    scope.launch {
        execute(onSuccess = onSuccess)
    }
}
```

- Dipanggil dari UI (mis. tombol). Menggunakan coroutine agar tidak blocking UI.

### 4) Eksekusi: minta kredensial & ambil ID Token

```kotlin
suspend fun execute(onSuccess: () -> Unit) {
    try {
        val request = createInstanceGoogleSignInClient("<SERVER_CLIENT_ID>")
        val result = credentialManager.getCredential(context = context, request = request)

        val credential = result.credential
        val googleIdTokenCredential = GoogleIdTokenCredential.createFrom(credential.data)
        firebaseSignInWithGoogle(googleIdTokenCredential.idToken, onSuccess)
    } catch (e: NoCredentialException) {
        // User batal / tidak ada akun cocok
    } catch (e: Exception) {
        // Error lain
    }
}
```

Apa yang terjadi di balik layar pada baris penting ini:

- `credentialManager.getCredential(...)`:
  - Meminta sistem untuk menampilkan UI akun Google (account picker/One Tap) dari Google Play services.
  - Jika user memilih akun, sistem mengembalikan data kredensial mentah.
- `GoogleIdTokenCredential.createFrom(...)`:
  - Mengekstrak Google "ID Token" (format JWT) dari data mentah tersebut.
  - ID Token ini memuat klaim identitas (sub, email, name) dan harus ditujukan ke `serverClientId` Anda (audience cocok).

### 5) Menukar ID Token ke Firebase Auth

```kotlin
private fun firebaseSignInWithGoogle(idToken: String, onSuccess: () -> Unit) {
    val credential = GoogleAuthProvider.getCredential(idToken, null)
    auth.signInWithCredential(credential)
        .addOnCompleteListener(context) { task ->
            if (task.isSuccessful) {
                onSuccess()
            }
        }
}
```

Di balik layar:

- `GoogleAuthProvider.getCredential(idToken, null)` membungkus ID Token menjadi `AuthCredential`.
- `auth.signInWithCredential(...)` mengirim kredensial ke backend Firebase:
  - Firebase memverifikasi ID Token ke Google (apakah valid, belum kadaluarsa, audience cocok, signature sah).
  - Jika valid, Firebase membuat/menyambungkan akun Firebase untuk user itu (berdasarkan `sub`/email).
  - Firebase menyimpan state login di device (SharedPreferences/Local storage) sehingga `auth.currentUser` tetap ada sampai sign-out.

### 6) Utility dan lifecycle

- `isAuthenticated()` hanya memeriksa `auth.currentUser != null`.
- `onDestroy()` membatalkan coroutine scope agar tidak ada kerja latar berjalan saat Activity dihancurkan.

### 7) Error handling yang ada

- `NoCredentialException`: user batal atau tidak ada akun cocok.
- `Exception`: fallback umum. Saat produksi, Anda bisa membedakan error jaringan, service tidak tersedia, atau Play Services usang.

---

## Integrasi dengan UI (contoh `MainActivity.kt`)

File: `app/src/main/java/com/example/ppb_project/MainActivity.kt`

Poin penting:

- Inisialisasi use case:
  ```kotlin
  private lateinit var googleSignIn: GoogleSignIn
  ...
  googleSignIn = GoogleSignIn(this)
  ```
- Auto-redirect jika sudah login:
  ```kotlin
  override fun onStart() {
      super.onStart()
      if (googleSignIn.isAuthenticated()) {
          toProfilePage()
      }
  }
  ```
- Klik tombol untuk memulai login:
  ```kotlin
  binding.btnGoogleSignIn.setOnClickListener {
      googleSignIn.signIn { toProfilePage() }
  }
  ```
- Navigasi setelah sukses:
  ```kotlin
  private fun toProfilePage() {
      val intent = Intent(this, TodoActivity::class.java)
      startActivity(intent)
      finish()
  }
  ```

Catatan: Ada `ProfileActivity` juga di project yang menampilkan `displayName` dan `email` dari `auth.currentUser`. Di kode saat ini, setelah login diarahkan ke `TodoActivity`. Anda bisa menyesuaikan target activity sesuai kebutuhan materi.

---

## Apa itu Google ID Token, dan mengapa “Server Client ID” penting?

- ID Token adalah JWT (JSON Web Token) yang berisi identitas user (mis. `sub`, `email`, `name`) dan ditandatangani Google.
- `aud` (audience) di ID Token harus cocok dengan Client ID yang Anda minta (`setServerClientId(...)`). Jika tidak cocok, Firebase akan menolak.
- Di proyek Firebase, Client ID yang dipakai biasanya adalah “Web client (Auto-created by Google Service)” yang muncul saat mengaktifkan Sign-in method Google.

---

## Ringkasan alur end-to-end (tekan tombol → masuk)

1. User tekan tombol → `signIn { ... }` dipanggil.
2. `execute()` membuat `GetCredentialRequest` berisi opsi Google dengan `serverClientId`.
3. Credential Manager menampilkan UI akun; user memilih akun.
4. App menerima `credential.data`, diekstrak jadi `GoogleIdTokenCredential` → ambil `idToken`.
5. Buat `AuthCredential` (GoogleAuthProvider) lalu panggil `FirebaseAuth.signInWithCredential`.
6. Firebase verifikasi ke Google; jika sukses → `auth.currentUser` terisi.
7. Callback `onSuccess()` dipanggil → Activity navigasi ke halaman berikutnya.

---

## Praktik baik & keamanan

- Jangan hardcode Client ID di kode produksi. Simpan di:
  - `local.properties` → inject ke `BuildConfig`, atau
  - `res/values/strings.xml`, atau
  - Remote Config / secrets manager.
- Pastikan `google-services.json` sesuai environment (debug vs release). Setel juga SHA-1/SHA-256 dari keystore debug dan release di Firebase Console agar sign-in berjalan di device nyata.
- Tangani error dengan pesan yang ramah pengguna, dan log detail untuk developer (jangan tampilkan stack trace ke user).
- Sediakan fitur sign-out dan (opsional) revoke access jika diperlukan.

Contoh sign-out sederhana:

```kotlin
Firebase.auth.signOut()
```

---

## Edge cases dan troubleshooting

- NoCredentialException: user batal, tidak ada akun di perangkat, atau belum ada kredensial cocok.
- Play Services / Google Account Picker tidak muncul: pastikan perangkat punya Google Play services yang up-to-date.
- Error kredensial/"12500"/audience mismatch: periksa Client ID yang dipakai di `setServerClientId`, harus yang dari Firebase/Google Cloud yang benar.
- Gagal di perangkat rilis tapi sukses di debug: kemungkinan SHA-1/SHA-256 untuk keystore rilis belum ditambahkan di Firebase Console.
- Jaringan lambat/putus: tampilkan indikator loading dan retry; Firebase akan menangani refresh token otomatis setelah berhasil login pertama kali.

---

## Tugas praktik untuk mahasiswa (opsional)

- Tambahkan tombol “Logout” di halaman setelah login, lalu kembalikan ke `MainActivity`.
- Tampilkan foto profil (`photoUrl`) dan verifikasi email di `ProfileActivity`.
- Tampilkan error spesifik saat gagal login (mis. jaringan, akun dibatalkan) agar UX lebih jelas.
- Pindahkan `serverClientId` ke `BuildConfig` atau `strings.xml` dan akses dari sana.

---

## Referensi

- Android Credential Manager: https://developer.android.com/training/sign-in/credential-manager
- Google Identity Services for Android: https://developers.google.com/identity/gsi/android
- Firebase Authentication (Google): https://firebase.google.com/docs/auth/android/google-signin

---
