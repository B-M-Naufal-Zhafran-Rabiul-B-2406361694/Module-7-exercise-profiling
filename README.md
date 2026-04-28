# Module 7 Exercise Profiling

## Hasil Performance Test

Pengujian dilakukan dengan konfigurasi JMeter yang sama untuk setiap endpoint: 10 users, ramp-up period 1 detik, dan loop count 1.

| Endpoint | Rata-rata Sebelum Optimasi | Rata-rata Setelah Optimasi | Peningkatan | Error Rate |
| --- | ---: | ---: | ---: | ---: |
| `/all-student` | 41566 ms | 158 ms | 99.62% | 0.00% |
| `/all-student-name` | 1266 ms | 14 ms | 98.89% | 0.00% |
| `/highest-gpa` | 48 ms | 7 ms | 85.42% | 0.00% |

Berdasarkan hasil JMeter, terdapat peningkatan performa yang jelas setelah proses optimasi. Semua endpoint yang diuji mengalami peningkatan lebih dari 20%, sehingga target optimasi performa telah tercapai.

Peningkatan terbesar terjadi pada endpoint `/all-student`. Sebelum optimasi, service melakukan query student course berulang kali untuk setiap student sehingga terjadi masalah N+1 query. Setelah refactor, data student dan course yang dibutuhkan diambil menggunakan satu query `join fetch`.

Endpoint `/all-student-name` juga meningkat signifikan karena proses penggabungan nama tidak lagi dilakukan dengan konkatenasi string berulang di Java. Proses tersebut dipindahkan ke database menggunakan `string_agg`. Endpoint `/highest-gpa` meningkat karena aplikasi tidak lagi mengambil semua student lalu mencari GPA tertinggi dengan loop di Java. Database langsung mengembalikan student dengan GPA tertinggi.

### `/all-student-name`

![Hasil JMeter CLI untuk all-student-name](screenshots/result_all_student_name_jtl.png)

![Report JMeter untuk all-student-name](screenshots/all-student-name-report.png)

### `/highest-gpa`

![Hasil JMeter CLI untuk highest-gpa](screenshots/result_highest_gpa_jtl.png)

![Report JMeter untuk highest-gpa](screenshots/highest-gpa-report.png)

## Reflection

### 1. Apa perbedaan pendekatan performance testing dengan JMeter dan profiling dengan IntelliJ Profiler dalam konteks optimasi performa aplikasi?

JMeter mengukur performa aplikasi dari sisi luar dengan mengirim request HTTP dan mencatat response time, throughput, serta error rate. Hasilnya menunjukkan apakah aplikasi terasa cepat atau lambat dari sudut pandang pengguna. IntelliJ Profiler mengukur dari sisi dalam aplikasi dengan menunjukkan method mana yang paling banyak memakai CPU time dan bagian kode mana yang paling berat. Jadi, JMeter membantu menemukan gejala performa, sedangkan profiler membantu menemukan penyebabnya di level kode.

### 2. Bagaimana proses profiling membantu mengidentifikasi dan memahami titik lemah aplikasi?

Profiling membantu menunjukkan method yang paling banyak mengonsumsi resource ketika request dijalankan. Pada aplikasi ini, profiler membantu mengarahkan perhatian ke method `getAllStudentsWithCourses`, `joinStudentNames`, dan `findStudentWithHighestGpa`. Dari situ terlihat bahwa beberapa proses masih melakukan loop manual atau akses database yang tidak efisien, sehingga bagian tersebut menjadi target optimasi.

### 3. Apakah IntelliJ Profiler efektif untuk membantu menganalisis dan mengidentifikasi bottleneck pada kode aplikasi?

Ya, IntelliJ Profiler efektif karena dapat menampilkan penggunaan CPU per method, call hierarchy, dan timeline eksekusi. Informasi tersebut membuat bottleneck lebih mudah dilacak, apakah berasal dari controller, service, repository, query database, atau overhead framework.

### 4. Apa tantangan utama saat melakukan performance testing dan profiling, dan bagaimana cara mengatasinya?

Tantangan utamanya adalah mendapatkan hasil pengukuran yang konsisten. Run pertama aplikasi biasanya lebih lambat karena JVM masih melakukan warm-up dan JIT compiler belum optimal. Untuk mengatasinya, endpoint dipanggil beberapa kali terlebih dahulu sebelum pengukuran utama. Tantangan lain adalah membedakan waktu response end-to-end dengan CPU time internal aplikasi. Karena itu, JMeter digunakan untuk melihat performa dari sisi request, sedangkan profiler digunakan untuk menentukan bagian kode yang perlu dioptimasi.

### 5. Apa manfaat utama yang diperoleh dari penggunaan IntelliJ Profiler?

Manfaat utamanya adalah profiler memberikan visibilitas langsung terhadap biaya eksekusi method. Daripada menebak bagian mana yang lambat, profiler menunjukkan method yang benar-benar memakan waktu dan jalur pemanggilannya. Hal ini membuat optimasi lebih terarah dan mengurangi risiko mengubah kode yang tidak berhubungan.

### 6. Bagaimana menangani kondisi ketika hasil profiling dengan IntelliJ Profiler tidak sepenuhnya konsisten dengan hasil performance testing menggunakan JMeter?

Kedua tools tersebut mengukur hal yang berbeda. JMeter mencakup waktu jaringan, serialisasi response, waktu tunggu database, dan waktu proses server secara keseluruhan. Profiler lebih fokus pada eksekusi internal aplikasi dan penggunaan CPU. Jika hasilnya berbeda, pengujian perlu diulang setelah warm-up, query database perlu diperiksa, dan bottleneck perlu diklasifikasikan apakah CPU-bound, I/O-bound, atau disebabkan oleh ukuran response.

### 7. Strategi apa yang diterapkan setelah menganalisis hasil performance testing dan profiling? Bagaimana memastikan perubahan tidak merusak fungsionalitas aplikasi?

Strateginya adalah mengoptimasi jalur kode yang paling mahal terlebih dahulu. Pada aplikasi ini, loop yang tidak efisien dikurangi dan pekerjaan yang lebih cocok dilakukan database dipindahkan ke query repository. Untuk memastikan fungsionalitas tetap benar, automated test dijalankan, endpoint diuji secara manual, lalu JMeter dijalankan kembali untuk memastikan perubahan menghasilkan peningkatan performa yang terukur.
