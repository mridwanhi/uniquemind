# Pembuatan REST-API dan Server UniqueMind

## Aktifitas Client dan Server

Client akan mengirimkan **HTTP-POST**  request ke server dengan data berikut :

```json
{
  "client" : 
  {
    "id"  : "string",
    "auth" : "api-token"
  },
  "request" :
  {
    "count" : 123,
    "record" :
    [
      {
        "index" : 1 , 
        "page" : 1 ,
        "ignore" : false ,
        "text" : "bla bla bla"
      },
      {
        "index" : 2 , 
        "page" : 1 ,
        "ignore" : false ,
        "text" : "bla bla bla"
      }
    ]
  },
  "document" :
  {
    "length": 123 ,
    "format" : "application/pdf" ,
    "zipped": false ,
    "binary" : "{base64 string}"
  }
}
```

Data tersebut akan dikirimkan ke server portal (**HTTP LISTENER**), dan server akan memproses dengan langkah berikut :

1. Server akan meminta request ke **ELASTIC** host yang berjalan untuk setiap potongan kalimat di *node* **record**,
contoh dengan **CURL** :
```curl
curl -X POST "localhost:9200/uniquemind/search" -H 'Content-Type: application/json' -d'
{
    "query": 
    {
        "bool" : 
        {
            "must" : 
            { 
              "query_string" :  { "query" : "Potongan kalimat nomor 1" }
            },
            "filter" : 
            {
                "term" : { "year" : "2010" }
            }
        }
    }
}
```

Selanjutnya, data yang diterima oleh server akan berupa `JSON` yang dikirimkan kembali oleh *Elastic*. Berikut adalah contoh *Response* yang dikirimkan oleh *Elastic* search :

```json
{
    "took": 1,
    "timed_out": false,
    "_shards":{
        "total" : 1,
        "successful" : 1,
        "skipped" : 0,
        "failed" : 0
    },
    "hits":{
        "total" : 1,
        "max_score": 1.3862944,
        "hits" : [
            {
                "_index" : "uniquemind",
                "_type" : "_doc",
                "_id" : "0",
                "_score": 1.3862944,
                "_source" : {
                    "dbid" : "database_id" ,
                    "url" : "http://blablabla",
                    "doi" : "10.xxx/abcdef",
                    "title" : "Judul dokumen, bla bla",
                    "authors" : 
                    [
                      "first-name" : "Aqew",
                      "last-name" : "Rizky",
                      "affiliation" : "Unjani"
                    ],
                    "year" : "tahun dokumen ini"
                }
            }
        ]
    }
}
```

Kita bisa menggunakan field `dbid` dari record `_source` dalam respon JSON yang diberikan oleh *Elastic* search. Kita gunakan `dbid` itu untuk mencari metadata dan `full-text` document dari database SQL-SERVER dengan mengirimkan query berikut :

```SQL
SELECT TOP 1 * FROM [dbo].[Documents] d WHERE d.[DocumentId] = @dbid
```

Pencarian menggunakan SQL Server akan cepat karena hanya menggunakan `primary-key` search terhadap field `[DocumentId]` dengan `clustered-index`.

## Catatan pencarian
Setiap hasil pencarian dari *Elastic* search, tidak boleh ada duplikat, misalnya pada pencarian kalimat ke 1 mendapatkan 3 dokumen tersimpan dengan **hits score** diatas *0.75*. Tentunya pada pencarian kalimat ke 2 dan selanjutnya hasil pencarian sebelumny tidak usah diikutkan pada *output* yang akan diberikan oleh client. Filter yang paling memungkinkan adalah berdasarkan **DOI** dan **Title**.

2. Selagi server memproses menggunakan *Elastic* search diatas, server akan menjalankan program utilitas yaitu **web-scrapping**, dengan mengirimkan `command line` ke proses web-scrapper (selama ini digunakan oleh *UniqueBrain*). Berikut adalah argumen-argumen dari `command-line` yang dapat di mengerti oleh proses web-scrapper.

Nama Proses : **UniqueBrain.Scrapper.exe**
## Command Lines
```
-crawl -input:<json/xml> -output:<json/xml> -rows:<1-10> -page:<1-10> -data:"C:\Queries\Session\T-001.<json/xml>"
```
Definisi `command-line` :

- `-crawl` : Menginstruksikan web-crawling, sifatnya wajib untuk di-inputkan.
- `-input` : Jenis input file, hanya support JSON atau XML (*skema input file dibahas dibawah*)
- `-output` : Jenis output file, hanya support JSON atau XML (*lokasi output ada di folder {PARENT_FOLDER\CRAWL} dengan nama yang sama dengan input file*)
- `-rows` : Jumlah baris (maksimum) yang akan diambil di setiap halaman search engine, harus angka dari 1 - 10. Standarnya adalah 3.
- `-page` : Jumlah halaman (maksumum) yang akan diambil dari setiap pencarian, harus angka dari 1 hingga 10. Standarnya adalah 1.
- `-data` : Input file yang akan diproses, berisikan list kalimat yang akan dicari menggunakan internet, e.g. Google, Bing, dll.

**Additional command-lines** :
- `-gui` : Menampilkan form pencarian (GUI)
- `-dieafter:N` : Proses pencarian akan otomatis berhenti jika hasil pencarian dari internet tidak ada (`NULL`) sebanyak `N`-Kali, contohnya `-dieafter:5` maka proses akan mengasumsikan tidak ada hasil lagi yang dapat dicari setelah 5x pencarian `NULL`.
- `-debug` : Menampilkan error dialog jika terjadi kesalahan, standarnya setiap error tidak akan dimunculkan (*silent-mode*).

## Contoh penggunaan

```
UniqueBrain.Scrapper.exe -crawl -input:json -output:json -rows:3 -page:1 -data:"C:\Input\User1_Query2.json"
```

Secara otomatis, proses **UniqueBrain.Scrapper.exe** akan mencari dari internet untuk kalimat yang diminta dan menyimpan output file di alamat berikut : `C:\Input\CRAWL\User1_Query2.json`.

Berikut adalah *Schema* dari input file yang akan diberikan ke proses web crawler :

```
{
  // default engine yang akan digunakan
  "Engine" : "Google|Bing|DuckDuckGo|Scholar...",
  
  // jumlah kalimat yang akan dicari
  "Total" : 10,
  
  // batasan waktu (timeout) dalam menit, set 0 untuk unlimited.
  "Limit" : 5 ,
  
  // array dari kalimat yang akan dicari
  "Entry" :
  [
    {
      // index kalimat ini dalam sequence-order
      "Index" : 1 ,
      
      // teks kalimat (tampa ada line-terminator atau tab)
      "Terms" : "Bla bla bla" ,
      
      // TRUE untuk cek website yang dirujuk dengan cara dibaca terlebih dahulu,
      // jika ada kalimat yang dicari maka website tersebut dimasukan kedalam list.
      // Ini akan memperlambat pencarian namun hasil yang akurat. Standarnya = false.
      "Check" : false ,
      
      // TRUE untuk download halaman HTML dan metadata dari website yang dirujuk.
      // Akan mempermudah client dan scoring, namun memperlambat pencarian.
      "Scrap" : false ,
      
      // TRUE untuk mendownload PDF full text dari halaman yang dirujuk jika ada 
      // tombol download atau alamat PDF dari metadata tags.
      "Fetch" : false ,
      
      // Nama search engine untuk kalimat ini, jika NULL maka akan sama dengan 
      // search engine default-nya (diatas).
      "Using" : "Google|Bing|DuckDuckGo|..." 
    },
    {
      "Index" : 2 ,
      "Terms" : "Bla bla bla" ,
      "Check" : false ,
      "Scrap" : false ,
      "Fetch" : false ,
      "Using" : "Google|Bing|DuckDuckGo|..." 
    }
  ]
}
```

### Jenis-Jenis Search Engine yang disupport :
- [Google] (https://www.google.com)
- [Scholar] (https://scholar.google.com)
- [Bing] (https://www.bing.com)
- [Microsoft Academic] (https://academic.microsoft.com/)
- [Duck Duck Go] (https://duckduckgo.com)
- [Start Page] (https://startpage.com)
- [QWant] (https://qwant.com)
- [QWant-Lite] (https://qwant-lite.com)
- [Web Crawler] (https://www.webcrawler.com)
- [SearX] (https://searx.me)
- [Swisscows] (https://swisscows.ch)
- [HotBot] (https://www.hotbot.com)
- [GigaBlast] (https://www.gigablast.com)
- [Yippy] (https://yippy.com)
- [DogPile] (https://dogpile.com)

**Catatan** : Tidak semua search engine dalam kondisi Fit terutama yang bertipe private search engine, seperti **GigaBlast** dan lainnya, standarnya web-crawler akan menggunakan **Duck Duck Go** sebagai mesin pencarian default, namun sebaiknya menggunakan kombinasi search engine di setiap kalimatnya.

### Format Output File

```json
{
   "Status" : "Success|Error|Timeout|Crash",
   "Message" :"Detail informasi lebih lanjut" ,
   "Results" : 1 ,
   "Elapsed" : "00:10:00.00000",
   "Starting" : "1 Januari 2019 12:40:12",
   "Finished" : "1 Januari 2019 12:50:12",
   "Records" :
   [
      {
        "EntryId" : 0 ,
        "PageNo" : 1, 
        "Position" : 8,
        "Scoring" : 0.719 ,
        "Engine" : "Google|Bing|.." ,
        "Address" : "https://www.blablabla.com" ,
        "Caption" : "Bla bla bla" ,
        "Snippet" : "Text text text" ,
        "FileURL" : "https://blabla.(pdf|.com)",
        "Crossref" : "https://api.crossref.org/works/(DOI)",
        "Metadata": 
        {
          "Title" : "Judul (kalau ada)",
          "DOI" : "DOI (kalau ada)",
          "Authors" : "Nama authors (kalau ada)" ,
          "PDF" : "Link download PDF (kalau ada)" ,
          "Year" : 2010 ,
          "Abstract" : "Text abstrak (kalau ada)"
        },
        "Matches" :
        [
           {
             "Index" : 55,
             "Score" : 0.862,
             "Found" : "Bla bla bla.",
             "Source" : "Bla bla bla.",
             "Offset" : 13,
           },
           {
             "Index" : 74,
             "Score" : 0.862,
             "Found" : "Bla bla bla.",
             "Source" : "Bla bla bla.",
             "Offset" : 34
           }
        ],
        "Contents" :
        {
          "HTML" : "HTML dokumen jika opsi 'scrap' digunakan dan jika ada.",
          "DOCS" : "Base-64 string dari dokumen PDF atau WORD jika opsi 'fetch' digunakan dan jika ada."
        }
      }
   ]
}
```
**Definisi global fields** :

- `Status` : Status dari pencarian, tidak pernah `NULL`, akan berupa `Success` jika proses sukses, `Error` jika selesai dengan error, `Timeout` jika selesai karena waktu habis, atau `Crash` jika proses error tanpa hasil (_fatal error_).

- `Message` : Deskripsi lebih lanjut tentang `Status` diatas, jika `Status` adalah **Success** biasanya `Message` akan `Success` juga.

- `Results` : **Integer**, total dari hasil yang ditemukan dalam pencarian internet. Angka akan NOL (0) jika tidak ada hasil apapun.

- `Elapsed` : Total waktu yang dibutuhkan dalam pencarian internet.

- `Starting` : Tanggal dan waktu dimana proses pencarian internet dimulai.

- `Finished` : Tanggal dan waktu dimana proses pencarian internet berakhir.

- `Records` : Array dari semua hasil pencarian internet. Array ini akan kosong jika tidak ada hasil apapun.

**Definisi fields dalam setiap Records**

- `EntryId` : **INTEGER**, Index dari entry, mulai dari nol hingga N (`Total - 1`).

- `PageNo` : **INTEGER**, Nomor halaman pencarian, mulai dari angka 1.

- `Position` :**INTEGER**, Baris dalam halaman, mulai dari angka 1 (teratas).

- `Scoring` : **FLOAT**, Jumlah plagiarism (secara kasar) untuk pencarian ke halaman yang dirujuk.

- `Address` : **URL**, Alamat *URL* dari halaman yang dirujuk oleh mesin pencarian.

- `Caption` : **STRING**, Title yang ditunjukan oleh mesin pencarian, seperti mencari di Google, maka akan ada title disetiap hasil pencarian.

- `Snippet` : **STRING**, Deskripsi pendek yang diberikan oleh mesin pencarian untuk halaman ini.

- `FileURL` : **URL**, Jika halaman yang dirujuk adalah halaman situs akademik dan **ADA** PDF dokumen yang dapat di-download secara bebas, maka `FileURL` akan berisikan alamat `URL` file tersebut. Jika tidak ada maka akan `NULL`.

- `Crossref` : **URL**, Jika halaman yang dirujuk terdapat informasi **DOI** (*Digital Identifier of an Object*), maka field ini akan berisikan alamat query metadata dari **CROSSREF** untuk DOI tersebut. Jika tidak ada maka akan `NULL`.

- `Metadata` : **OBJECT**, berikut adalah definisi dari object `Metadata` ini :

1. `Metadata->Title` : **STRING**, Jika ada, maka akan berisi judul asli dari dokumen yang ditemukan, jika tidak maka akan `NULL`.
2. `Metadata->DOI` : **STRING**, Jika ada, maka akan berisi **DOI** dari dokumen yang ditemukan, jika tidak maka akan `NULL`.
3. `Metadata->Authors` : **STRING**, Jika ada, maka akan berisi list **AUTHORS** (nama pembuat) dari dokumen yang ditemukan, setiap nama dipisahkan oleh koma (comma-delimited). Jika tidak ada maka akan `NULL`.
4. `Metadata->PDF`: **URL**, Alamat PDF berdasarkan DOI, contoh: "https://dx.doi.org/10.1423.xxxx" (sample). Jika tidak ada **DOI** maka akan `NULL`.
5. `Metadata->Year` : **INTEGER**, jika ada, maka akan berisi tahun publikasi dokumen yang ditemukan, jika tidak maka akan `NULL`.
6. `Metadata->Abstract` : **STRING**, jika ada maka akan berisi potongan teks abstrak dari dokumen yang ditemukan, jika tidak maka `NULL`.

- `Matches` : **OBJECT**, adalah array yang menunjukan potongan kalimat dari halaman ini yang sama dengan kalimat yang ada pada input file. Berikut adalah definisi dari setiap fields yang ada dalam object `Matches` :

1. `Matches->Index` : **INTEGER**, Index dari hasil extrak kalimat di dokumen atau halaman yang ditemukan, mulai dari angka nol.
2. `Matches->Score` : **FLOAT**, Nilai kesamaan (*Damerau-Levensthein Distance*) untuk kalimat ini.
3. `Matches->Actual` : **STRING**, Text yang ada pada halaman atau dokumen yang ditemukan yang diduga memiliki kesamaan.
4. `Matches->Source` : **STRING**, Text yang ada pada list kalimat di input file yang diduga mirip.
5. `Matches->Offset` : **INTEGER**, Index pada list kalimat yang ada di input file yang diduga mirip.

- `Contents` : **OBJECT**, berisikan raw-data dari hasil scrapping. **WARNING**, jika ada maka fields dalam object akan berisikan text yang panjang atau berukuran besar. Berikut adalah definisi dari fields yang ada pada object `Contents` :

1. `Contents->HTML` : Jika opsi `scrap` digunakan, maka field ini akan berisikan text dari HTML dokumen yang diambil jika ada. Jika tidak maka akan `NULL`.
2. `Contents->PDF` : Jika opsi `fetch` digunakan dan ditemukan file PDF yang dapat di-download. Maka field ini berisikan `BASE-64` string dari binary dokumen PDF yang sudah di-download.


=================================================================================

Setelah kedua proses diatas selesai, maka hasil output akhir akan dikembalikan ke client lagi sebagai response dari `HTTP-POST` yang diawal dikirim oleh client.

**Catatan** :

- Format komunikasi data antara `client-server` nanti akan menyusul.
- Penjelasan diatas adalah penjelasan teknis dari langkah-langkah yang akan dijalan-kan oleh `PORTAL` server.

Silahkan dikoreksi jika ada kesalahan.


**Rizky**
