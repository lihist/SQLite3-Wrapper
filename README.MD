# SQLite3 Wrapper

Bu proje, D dilinde SQLite3 için derleme zamanlı header ayrıştırma ve BetterC uyumlu FFI wrapper üretimi örneğidir. Amaç, `c/sqlite3.h` dosyasındaki SQLite C API fonksiyon prototiplerini otomatik okuyup D tarafında `extern(C)` bildirimleri ve güvenli çağrı sarmalayıcıları üretmektir.

Proje ayrıca `db/chinook.db` örnek veritabanı üzerinde sorgular çalıştırarak wrapper'ın pratik kullanımını gösterir.

## Öne Çıkan Özellikler

- `sqlite3.h` dosyası D'nin string import mekanizmasıyla derleme zamanında okunur: `import("sqlite3.h")`.
- SQLite C API fonksiyon prototipleri normal runtime regex ile değil, `std.regex.ctRegex` ile hazırlanmış derleme zamanlı regex üzerinden bulunur.
- Header'dan bulunan fonksiyonlar için D `extern(C)` bildirimleri otomatik üretilir.
- Kullanılan SQLite çağrıları için `@trusted @nogc nothrow` D wrapper fonksiyonları üretilir.
- Güvenli olmayan C çağrıları tek noktada `@trusted` sarmalayıcıların içine alınır.
- Uygulamanın ana hedefi `-betterC` ile derlenir; D runtime ve GC gerektirmez.
- SQLite bağlantısı ve statement nesneleri `struct` + `~this()` ile scope-based kaynak yönetimi kullanır.
- `db/chinook.db` veritabanındaki tablolar, satır sayıları ve örnek müzik/fatura sorguları ile test edilir.

## Proje Yapısı

```text
.
├── c/
│   ├── sqlite3.c
│   ├── sqlite3.h
│   └── sqlite3.obj
├── db/
│   ├── chinook.db
│   └── README.MD
├── source/
│   ├── app.d
│   └── sqlite3_wrapper/
│       ├── db.d
│       ├── ffi.d
│       ├── generated.d
│       └── raw.d
├── tools/
│   └── header_generator.d
├── build-sqlite3.ps1
└── dub.json
```

## Nasıl Çalışır?

### 1. Header derleme zamanında okunur

`tools/header_generator.d`, SQLite header dosyasını şu şekilde derleme zamanında içeri alır:

```d
enum sqliteHeader = import("sqlite3.h");
```

`dub.json` içinde `stringImportPaths` olarak `c` dizini tanımlıdır. Bu sayede `sqlite3.h` dosyası doğrudan import edilebilir.

### 2. Fonksiyonlar `ctRegex` ile bulunur

Generator, `SQLITE_API ... sqlite3_...(...);` biçimindeki fonksiyon prototiplerini `ctRegex` ile yakalar. Regex nesnesi derleme zamanında hazırlanır; generator çalıştığında header içeriği üzerinde bu desenle eşleşmeler alınır.

Bu aşamada regex'in hedefi sadece SQLite API fonksiyon prototipleridir. Değişken exportları ve C makroları wrapper üretimine dahil edilmez.

### 3. D `extern(C)` bildirimleri üretilir

Generator, bulunan fonksiyonların dönüş tiplerini ve argüman tiplerini D karşılıklarına çevirir. Örnek dönüşümler:

```text
const char *      -> const(char)*
sqlite3 *         -> sqlite3*
sqlite3_stmt **   -> sqlite3_stmt**
unsigned char *   -> ubyte*
sqlite3_int64     -> long alias'ı
```

Üretilen `extern(C)` bildirimleri `source/sqlite3_wrapper/generated.d` içine yazılır ve `raw.d` içinde `mixin` ile kullanılır.

### 4. @trusted wrapper fonksiyonları üretilir

Ham `extern(C)` fonksiyonlar `@system` kabul edilir. Uygulamanın kullandığı seçili fonksiyonlar için ayrıca `@trusted @nogc nothrow` wrapper fonksiyonları üretilir:

```d
@trusted @nogc nothrow int sqlite3_open(const(char)* arg0, sqlite3** arg1)
{
    return raw.sqlite3_open(arg0, arg1);
}
```

Bu yaklaşımda C tarafına geçiş noktası izole edilir. Uygulama kodu doğrudan ham C sembollerini çağırmak yerine wrapper fonksiyonları kullanır.

### 5. BetterC ve GC'siz çalışma

Ana uygulama `dub.json` içindeki şu bayrakla derlenir:

```json
"dflags": ["-betterC"]
```

Bu nedenle uygulama:

- D runtime başlatmaz.
- GC kullanmaz.
- `main` fonksiyonu `extern(C) int main() @nogc nothrow` olarak yazılır.
- Çıktı için Phobos yerine `core.stdc.stdio.printf` kullanılır.
- Kaynak yönetimi sınıf/GC yerine stack üzerindeki `struct` destructorlarıyla yapılır.

Not: `std.regex.ctRegex` Phobos'a bağlı olduğu için BetterC hedefinin içine doğrudan alınmaz. Bu yüzden header ayrıştırma ayrı bir pre-build generator aracıyla yapılır. Generator normal D ile derlenir, ürettiği kod ise BetterC uygulamasında kullanılır.

### 6. Scope-based kaynak yönetimi

`source/sqlite3_wrapper/db.d` içinde iki temel struct vardır:

```d
struct Database
{
    sqlite3* handle;

    ~this() @trusted @nogc nothrow
    {
        if (handle !is null)
            sqlite3_close(handle);
    }
}
```

`Database` scope dışına çıktığında `sqlite3_close` otomatik çağrılır. `Statement` için de aynı model kullanılır ve destructor içinde `sqlite3_finalize` çalışır.

Bu model C kaynaklarını manuel `close/finalize` çağrılarını unutmaya karşı daha güvenli hale getirir.

## Build ve Çalıştırma

Gereksinimler:

- DMD
- DUB
- Windows üzerinde MSVC `cl.exe`
- SQLite amalgamation dosyaları: `c/sqlite3.c`, `c/sqlite3.h`

Derlemek için:

```powershell
dub build --compiler=dmd
```

Çalıştırmak için:

```powershell
dub run --compiler=dmd
```

Pre-build aşaması şunları yapar:

- `tools/header_generator.d` dosyasını derler.
- `c/sqlite3.h` içinden wrapper kodunu üretir.
- `source/sqlite3_wrapper/generated.d` dosyasını günceller.
- `c/sqlite3.c` dosyasını MSVC ile `c/sqlite3.obj` nesne dosyasına çevirir.
- BetterC D uygulamasını bu nesne dosyasıyla linkler.

## Chinook Veritabanı Testleri

Program `db/README.MD` dosyasında anlatılan Chinook tablolarını temel alır ve `db/chinook.db` üzerinde şu sorguları çalıştırır:

- Veritabanındaki tablo listesini okur.
- `artists`, `albums`, `tracks`, `customers` tablolarının satır sayılarını gösterir.
- Track sayısına göre en popüler genre listesini çıkarır.
- Fatura toplamına göre en yüksek harcama yapan müşterileri listeler.

Örnek çıktı:

```text
SQLite runtime: 3.25.3
sqlite3.h compile-time parse: 315 functions, 315 extern(C) declarations generated

Catalog counts
table_name=artists | rows=275
table_name=albums | rows=347
table_name=tracks | rows=3503
table_name=customers | rows=59

Top genres by track count
genre=Rock | tracks=1297
genre=Latin | tracks=579
genre=Metal | tracks=374
```

## Tasarım Notları

Bu wrapper bir eğitim ve sistem programlama örneğidir. Tüm SQLite API yüzeyi için `extern(C)` bildirimleri üretir, fakat yüksek seviyeli güvenli D wrapper katmanı şu anda örnek programın kullandığı fonksiyonlara odaklanır.

Kapsanan güvenli wrapper fonksiyonları arasında şunlar vardır:

- `sqlite3_libversion`
- `sqlite3_open`
- `sqlite3_close`
- `sqlite3_errmsg`
- `sqlite3_prepare_v2`
- `sqlite3_step`
- `sqlite3_finalize`
- `sqlite3_column_count`
- `sqlite3_column_name`
- `sqlite3_column_text`
- `sqlite3_column_int`
- `sqlite3_column_double`

İstenirse generator'daki `selected` listesi genişletilerek daha fazla SQLite fonksiyonu için `@trusted` wrapper üretilebilir.

## Sınırlamalar

- C makroları ve sabitleri otomatik çevrilmez; gerekli SQLite sabitleri D tarafında elle tanımlanır.
- Karmaşık C callback tipleri için temel destek vardır, fakat yüksek seviyeli D callback adaptörü bu örneğin kapsamına alınmamıştır.
- Generator normal D/Phobos kullanır; BetterC hedefi generator çıktısını tüketir.
- `generated.d` otomatik üretilen dosyadır; kalıcı değişiklikler generator üzerinde yapılmalıdır.

