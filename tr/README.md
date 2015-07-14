# HTTP API Tasarım Kılavuzu

## Giriş

Bu kılavuz bir takım HTTP+JSON API tasarımı uygulamalarını açıklar ve 
[Heroku Platform API](https://devcenter.heroku.com/articles/platform-api-reference) 
çalışmalarından ortaya çıkmıştır.

Bu kılavuz Heroku'nun yeni iç API'lerini yönlendirirken API hakkında da bilgi 
verir. Umarız Heroku dışındaki API tasarımcılarına da yarar sağlar. 

Bizim amaçlarımız oyalanma tasarımından kaçarak tutarlılık ve iş mantığına 
odaklanmak. API tasarımı için _sadece ve en ideal olan yolu_ değil _iyi, 
tutarlı, güzel dokümante edilmiş bir yol_ aramaktayız.

Basit HTTP+JSON API arayüzü hakkında bilgi sahibi olduğunuzu varsayıyoruz ve 
bu kılavuz API'nin temel konularından bir çok şeyi kapsamayacak.

Bu kılavuza [katkıları](CONTRIBUTING.md) kabul ediyoruz. 

## İçerik

* [Temeller](#temeller)
  *  [Kaygılarınızı Parçalayın](#kaygılarınızı-parçalayın)
  *  [Güvenli Bir Bağlantı Gereklidir](#güvenli-bir-bağlantı-gereklidir)
  *  [Sürüm Bilgisi için "Accepts" Başlığı Gereklidir](#sürüm-bilgisi-için-accepts-başlığı-gereklidir)
  *  [Önbellekleme için ETags Desteği](#Önbellekleme-için-etags-desteği)
  *  [Gözlemlemek için `Request-Ids` Oluşturun](#gözlemlemek-için-request-ids-oluşturun)
  *  [Range'ler ile Yüksek Boyuttaki Cevapları Parçalara Bölün](#rangeler-ile-yüksek-boyuttaki-cevapları-parçalara-bölün)
* [İstekler](#İstekler)
  *  [Uygun durum kodları(status codes) döndürün](#uygun-durum-kodlarıstatus-codes-döndürün)
  *  [Uygun Olan Yerlerde Tüm Kaynakları Sunun](#uygun-olan-yerlerde-tüm-kaynakları-sunun)
  *  [İstek gövdesinde JSON Kabul Edin](#İstek-gövdesinde-json-kabul-edin)
  *  [Kararlı URL Yolu Formatı Kullanın](#kararlı-url-yolu-formatı-kullanın)
    *  [Kaynak İsimleri](#kaynak-İsimleri)
    *  [Olaylar](#olaylar)
    *  [URL ve Niteliklerde Küçük Harf Kullanın](#downcase-paths-and-attributes)
    *  [ID Baz Alınmadan Veri Çekmeyi Destekleyin](#id-baz-alınmadan-veri-Çekmeyi-destekleyin)
    *  [Çok Dallanan URL Yollarını (nested path) Azaltın](#Çok-dallanan-url-yollarını-nested-path-azaltın)
* [Cevaplar](#cevaplar)
  *  [Kaynaklar için (UU)ID Oluşturun](#kaynaklar-için-uuid-oluşturun)
  *  [Standart Zaman Damgaları(timestamp) Oluşturun](#standart-zaman-damgalarıtimestamp-oluşturun)
  *  [ISO8601'deki UTC zamanını kullanın](#iso8601deki-utc-zaman-formatını-kullanın)
  *  [Dallanan Nesnelerin (Nested Object) İlişkilendirimesi](#dallanan-nesnelerin-nested-object-İlişkilendirimesi)
  *  [Belirli Yapıda Hatalar Oluşturun](#belirli-yapıda-hatalar-oluşturun)
  *  [`Rate Limit` Durumunu Gösterin](#rate-limit-durumunu-gösterin)
  *  [Bütün Cevaplarda JSON Veriniz Sıkıştırılmış(minified) Olsun](#bütün-cevaplarda-json-veriniz-sıkıştırılmışminified-olsun)
* [Artifacts](#artifacts)
  *  [Programların Okuyabileceği(Machine-readable) JSON Şeması Oluşturun](#programların-okuyabileceğimachine-readable-json-Şeması-oluşturun)
  *  [İnsanların Okuyabileceği Dökümanlar Oluşturun](#İnsanların-okuyabileceği-dokümanlar-oluşturun)
  *  [Çalıştırılabilir Örnekler Oluşturun](#Çalıştırılabilir-Örnekler-oluşturun)
  *  [İstikrarı Açıklayın](#İstikrarı-açıklayın)
* [Çeviriler](#ceviriler)

### Temeller

#### Kaygılarınızı Parçalayın

Tasarım sırasında nesneleri(kaygılarınızı) istek(request) ve cevap(response) 
döngüsündeki farklı bölümler arasında parçalayarak basite indirgeyin. Kuralların 
basit tutulması büyük ve zor sorunlar için daha fazla odaklanma sağlar. 

İstek ve cevaplar belirli bir kaynak veya koleksiyon adresine yapılacaktır. 
Kimliği belirlemek için adres yolunu, içeriği iletmek için gövdeyi ve meta veri 
iletişimi için başlıkları kullanın. Sorgu parametreleri farklı durumlarda başlık(header) 
verilerini iletmek için kullanılabilir, ama başlıklar ile iletmek daha esnek ve 
daha çeşitli bilgi göndermek için tercih edilir.

#### Güvenli Bir Bağlantı Gereklidir

API'ye erişim için istisnasız TLS(Transport Layer Security) ile güvenli bir bağlantı kullanın.

Güvensiz veri alışverişini önlemek için http veya 80 portu üzerinden gelen 
isteklere yanıt vermeyerek, TLS olmayan istekleri basitçe reddet. Mümkün olmayan 
ortamlarda `403 Forbidden` ile yanıt ver. 

Herhangi bir net kazancı olmadan özensiz/kötü istemci(client) davranışlarına 
izin verdiği için yönlendirmelerden kaçınılmalıdır. İstemciler yönlendirmelere
güvenerek sunucu trafiğini ikiye katlarlar ve hassas veriler ilk sorguda 
korunmasız kaldığından TLS kullanmanın bir anlamı olmaz.

#### Sürüm Bilgisi için "Accepts" Başlığı Gereklidir

Sürüm ve sürümler arası geçişler API'nin işletilmesi ve tasarlanması aşamasının 
en zor yanlarından birisidir. Bunun gibi mekanizmalar ile başlamak, en baştan 
bu sıkıntıları azaltmak için en iyisidir.

Süprizleri önlemek, kullanıcıların değişikliklerini kırmak için en iyi yöntem 
bütün isteklerde bir sürüm gerekliliği en iyisidir. Bu anlamda en iyisi, daha 
sonra bir değişiklik yapmanın zor olmasını engellemek için varsayılan versiyonlar
kullanılmamalıdır.

Sürüm özelliklerini başlıklar içerisinde diğer meta veriler ile birlikte sunmak 
en iyisidir. `Accept` başlığını kullanma örneği:

```
Accept: application/vnd.heroku+json; version=3
```

#### Önbellekleme için ETags Desteği

Bütün cevaplar(responses), dönen cevabın içeriğine özel tanımlayıcılar olan, 
`ETag` içermelidir. Bu kullanıcılara kaynakları önbelleklemek için ve kendi 
belleğindeki(cache) verinin güncellenip güncellemeyeceğini karşılaştırmak 
için isteklerde kullanmalarına izin verir. 
[`If-None-Match`](https://tools.ietf.org/html/rfc2616#section-14.26) 

#### Gözlemlemek için `Request-Ids` Oluşturun

Her API cevabında UUID değeri ile doldurulmuş, `Request-Id` başlığı olmalıdır.
Bu değeri, istekleri `trace`, `diagnose` ve `debug` edebilmek için, sunucu, 
istemci(client) ve diğer her servis tarafında günlükleme (logging) için 
kullanabilirsiniz. 

#### Range'ler ile Yüksek Boyuttaki Cevapları Parçalara Bölün

Yüksek boyutlardaki cevapları(responses) [`Range`](https://tools.ietf.org/html/rfc2616#section-14.35) 
başlığını kullanarak bir kaç parçaya bölebilirsiniz. İstek ve cevapların 
başlıklar, durum kodları, limitler, sıralama ve tekrarlama(iteration) hakkında 
daha fazla bilgi için [Heroku Platform API discussion of Ranges](https://devcenter.heroku.com/articles/platform-api-reference#ranges)
bağlantısını takip edebilirsiniz. 

### İstekler

#### Uygun Durum Kodları(status codes) Döndürün

Her cevapla birlikte cevaba uygun HTTP durum kodları döndürün. Başarılı cevaplar
bu kılavuza uygun olmalı:

* `200`: `GET` metodu ile yapılan istek başarılıdır. `DELETE` veya `PATCH` metodu ile yapılan istekler(requests) senkronlu bir şekilde tamamlanmıştır, veya `PUT` metodu ile yapılan istek(request) senkron bir şekilde var olan içeriği güncellemiştir.
* `201`: Senkron bir şekilde yapılan `POST` veya senkron bir şekilde yeni bir içerik yaratan`PUT` isteği(request) başarılı olarak tamamlanmıştır.
* `202`: `POST`, `PUT`, `DELETE`, veya `PATCH` metodu ile asenkron bir şekilde yapılan istek(request) kabul edilmiştir.
* `206`: `GET` ile yapılan istek(request) başarılıdır, fakat sadece cevabın(response) bir kısmı dönmüştür: bakınız [Range'ler ile Yüksek Boyuttaki Cevapları Parçalara Bölün](#divide-large-responses-across-requests-with-ranges)

Doğrulama(authentication) ve yetki(authorization) kodlarını kullanmaya dikkat edin:

* `401 Unauthorized`: Kullanıcı yetkilendirilmediği için istek(request) başarısız olmuştur.
* `403 Forbidden`: Kullanıcının belli bir bölüme ulaşma yetkisi olmadığı için istek başarısız olmuştur.

Ek bilgilerin eklenmesi konusunda bir hata yapıldığında uygun kodları dönün:

* `422 Unprocessable Entity`: Yapılan istek anlamlandırılmış, fakat gönderilen parametler doğru değildir.
* `429 Too Many Requests`: Yapılan istek sayısı izin verilen limite ulaşmıştır, tekrar deneyin.
* `500 Internal Server Error`: Sunucu üzerinde bir şeyler ters gitti, kontrol edin veya yetkili kişiye rapor edin.

Sunucu ve kullanıcı hata durum kodları hakkında daha fazla bilgi için 
[HTTP response code spec](https://tools.ietf.org/html/rfc7231#section-6) 
adresini kontrol edebilirsiniz.

#### Uygun Olan Yerlerde Tüm Kaynakları Sunun

İmkan olduğu sürece tüm [kaynakları](https://tools.ietf.org/html/rfc7231#section-2) 
(örneğin: bir nesnenin tüm niteliklerini) cevap olarak dönün. `PUT`/`PATCH` 
ve `DELETE` istekleri dahil, [200](https://tools.ietf.org/html/rfc7231#section-6.3.1) 
ve [201](https://tools.ietf.org/html/rfc7231#section-6.3.2) 
cevaplarında daima tüm kaynakları geri döndürün, örneğin:

```bash
$ curl -X DELETE \  
  https://service.com/apps/1f9b/domains/0fd4

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
...
{
  "created_at": "2012-01-01T12:00:00Z",
  "hostname": "subdomain.example.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "updated_at": "2012-01-01T12:00:00Z"
}
```
[202](https://tools.ietf.org/html/rfc7231#section-6.3.3) cevapları 
kaynakları(resources) içermeyecektir.  
Örneğin:

```bash
$ curl -X DELETE \  
  https://service.com/apps/1f9b/dynos/05bd

HTTP/1.1 202 Accepted
Content-Type: application/json;charset=utf-8
...
{}
```

#### İstek gövdesinde JSON Kabul Edin

`form-encoded` veriyi kullanmak ya da buna ek olarak kullanmak yerine 
`PUT`/`PATCH`/`POST` istek gövdelerinde JSON verilerini kabul edin. Bu bir 
JSON-serialized cevap gövdesi ile bir simetri oluşturacaktır. 

```bash
$ curl -X POST https://service.com/apps \
    -H "Content-Type: application/json" \
    -d '{"name": "demoapp"}'

{
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "name": "demoapp",
  "owner": {
    "email": "username@example.com",
    "id": "01234567-89ab-cdef-0123-456789abcdef"
  },
  ...
}
```

#### Kararlı URL Yolu Formatı Kullanın

##### Kaynak İsimleri

Eğer kaynak sistemde tekil değilse kaynakların çoğul isimlerini kullanın 
(Örneğin, bir çok sistemde kullanıcıların sadece bir kullanıcı hesabı vardır).
Bu özel kaynaklara tutarlılık katar.

##### Olaylar

Özel bir aksiyon gerektiren durumlarda olayları standart bir `actions` öneki ile 
tanımlayın:

```
/resources/:resource/actions/:action
```

Örneğin:

```
/runs/{run_id}/actions/stop
```

#### URL ve Niteliklerde Küçük Harf Kullanın

URL yolu için küçük harf ve `-` ile ayrılmış (dash-seperated) isimler kullanın.
Örneğin:

```
service-api.com/users
service-api.com/app-setups
```

Nitelikleri(attributes) küçük harfe çevirin, kelimeleri `_` kullanarak ayırın.
Böylelikle Javascript'te tırnak işareti kullanmadan yazılabilir. Örneğin:

```
service_class: "first"
```

#### ID Baz Alınmadan Veri Çekmeyi Destekleyin

Bazı durumlarda son kullanıcı için kaynağı ID ile ilişkilendirmek kolay 
olmayabilir. Örneğin, kullanıcılar Heroku uygulama isimleri üzerinden 
düşünüyorlar, ama uygulama bir UUID tarafından ile ilişkilendirilmiş olabilir.
Bu durumda id ve name terimlerinin ikisi ile de ulaşılabilir olabilir. Örneğin:

```bash
$ curl https://service.com/apps/{app_id_or_name}
$ curl https://service.com/apps/97addcf0-c182
$ curl https://service.com/apps/www-prod
```
Bu ID'yi hariç tutup sadece isim ile veri çekilebilir yapmak değildir.

#### Çok Dallanan URL Yollarını (nested path) Azaltın

Veri modelinizdeki iç içe ilişkili kaynaklar için çok fazla alt dalları oluşan 
URL yollarından kaçının. Aşağıdaki örnek bu karmaşık yapıya örnektir.

```
/orgs/{org_id}/apps/{app_id}/dynos/{dyno_id}
```

Derin ilişkilerin dallanan yollarını ana yollarda kalacak şekilde kısıtlayın. 
Örneğin, aşağıdaki örnekte `dyno` bir organizasyonun uygulamasıdır:

```
/orgs/{org_id}
/orgs/{org_id}/apps
/apps/{app_id}
/apps/{app_id}/dynos
/dynos/{dyno_id}
```

### Cevaplar

#### Kaynaklar için (UU)ID Oluşturun

Her kaynak için varsayılan olarak bir `id` parametresi oluşturun. Kullanmamak 
için iyi bir sebebiniz olana kadar UUID'leri kullanın. Tüm heryerde tekil 
olmayan ID'leri kullanmayın. ID'ler genelde otomatik artan (autoincrement) 
olduğu için diğer servislerde yine karşılaşılabilirdir. 

UUID'lerin formatı küçük harflerle ve `8-4-4-4-12` şeklindedir. Örneğin:

```
"id": "01234567-89ab-cdef-0123-456789abcdef"
```

#### Standart Zaman Damgaları(timestamp) Oluşturun

`created_at` ve `updated_at` zaman damgalarını varsayılan olarak oluşturun. 
Örneğin:

```javascript
{
  // ...
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T13:00:00Z",
  // ...
}
```

Bu zaman damgaları bazı kaynaklar için mantıklı olmayabilir, bu durumlarda 
kullanmayabilirsiniz.

#### ISO8601'deki UTC Zaman Formatını Kullanın

ISO8601 formatında, UTC zamanını kullanın. Örneğin: 

```
"finished_at": "2012-01-01T12:00:00Z"
```

#### Dallanan Nesnelerin (Nested Object) İlişkilendirimesi

`Foreign Key` referanslarını dallanan nesneler(nested object) ile gösterin, 
Örneğin:

```javascript
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0..."
  },
  // ...
}
```

Aşağıdaki gibi yapmamaya çalışın:

```javascript
{
  "name": "service-production",
  "owner_id": "5d8201b0...",
  // ...
}
```

Bu yaklaşım gerektiğinde size bu dallanan nesneler hakında, cevabınızın yapısını 
değiştirmeden ya da bozmadan  daha fazla bilgi verebilme imkanı sağlar. Örneğin:

```javascript
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0...",
    "name": "Alice",
    "email": "alice@heroku.com"
  },
  // ...
}
```

#### Belirli Yapıda Hatalar Oluşturun

Tutarlı, belirli yapıda hatalar oluşturun. Bir programın okuyabileceği hata
`id`'leri ve insanların anlayabileceği hata `message`'ları, daha fazla bilgi 
alabilecekleri ve çözüm bulabilecekleri hata `url`'leri belirleyin. Örneğin:

```
HTTP/1.1 429 Too Many Requests
```

```json
{
  "id":      "rate_limit",
  "message": "Account reached its API rate limit.",
  "url":     "https://docs.service.com/rate-limits"
}
```

Kullanıcılarınız karşılaşabileceği hata formatlarınızı dokümante edin.

#### `Rate Limit` Durumunu Gösterin

`Rate Limit` diğer kullanıcıların servisi sağlıklı kullanabilmeleri için 
istemcinin isteklerinini sınırlandırılmasıdır. Bunun için 
[token bucket algorithm](http://en.wikipedia.org/wiki/Token_bucket)
algoritmasını kullanabilirsiniz. 

Kalan istek sayılarını `RateLimit-Remaining` başlığı içerisinde sunun.

#### Bütün Cevaplarda JSON Veriniz Sıkıştırılmış(minified) Olsun

Ekstra boşluklar isteklerin cevaplarını gereksiz yere büyütür, ve bir çok 
istemci zaten otomatik olarak bu verileri güzel görünür hale getirmektedir. 
En iyisi JSON verinizi sıkıştırılmış halde tutmaktır. Örneğin:

```json
{"beta":false,"email":"alice@heroku.com","id":"01234567-89ab-cdef-0123-456789abcdef","last_login":"2012-01-01T12:00:00Z","created_at":"2012-01-01T12:00:00Z","updated_at":"2012-01-01T12:00:00Z"}
```

Aşağıdakinin yerine:

```json
{
  "beta": false,
  "email": "alice@heroku.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "last_login": "2012-01-01T12:00:00Z",
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

Sorgu paramatreleri ile (Örneğin: `?pretty=true`) veya başlık parametresi
(Örneğin: `Accept: application/vnd.heroku+json; version=3; indent=4;`) olarak 
kullanıcılarınıza güzel görünümlü hali için bir seçenek sunabilirsiniz.

### Artifacts

#### Programların Okuyabileceği(Machine-readable) JSON Şeması Oluşturun

API arayüzünüzü ayrıntılarıyla belirtmek için programların anlayabileceği 
şemalar oluşturun. Şemanızı yönetmek için [prmd](https://github.com/interagent/prmd) 
kullanabilirsiniz, ve `prmd verify` ile geçerliliğini doğrulayın. 

#### İnsanların Okuyabileceği Dökümanlar Oluşturun

API'nizi kullanmak isteyecek istemci developerların anlayabileceği dokümanlar
oluşturun. 

Eğer daha önce bahsedildiği gibi prmd ile bir şema oluşturursanız, `prmd doc` 
komutu ile son kullanıcılarınıza kolayca Markdown dokümanı oluşturabilirsiniz

Son kullanıcılara ek olarak, aşağıdaki konuları içeren bir API'ye genel bakış 
bölümün oluşturun:

* Authentication(Kimlik DOğrulama), bilgilerini edinme ve kullanımı.
* API kararlılığı ve versiyonlama, API versiyonlarını seçimi
* Genel istek ve cevap başlıkları
* Hata formatları
* API arayüzünü farklı dillerdeki kullanıcılar ile kullanım örnekleri

#### Çalıştırılabilir Örnekler Oluşturun

Kullanıcılarınızın terminal arayüzünden direk çağırabileceği çalıştırılabilir 
örnekler hazırlayın. Bu örneklerin, kullanıcının API arayüzü ile çalışırken 
minimum enerjiyi harcaması için, mümkün olduğunca kelimesi kelimesine 
kullanılabilir olmasını sağlayın. Örneğin: 

```bash
$ export TOKEN=... # acquire from dashboard
$ curl -is https://$TOKEN@service.com/users
```

Eğer siz Markdown dokümanı oluştururken [prmd](https://github.com/interagent/prmd)
kullanırsanız, ücretsiz, her son kullanıcı örnekleriniz olur. 

#### İstikrarı Açıklayın

API'nizin istikrarlılığını açıklayın veya kararlılığına ve olgunluğuna göre 
birden fazla bağlantı seçeneği sunun, örneğin: prototype/development/production
seçenekleri ile. 

[Heroku API compatibility policy](https://devcenter.heroku.com/articles/api-compatibility-policy)
dokümanını, yönetim yaklaşımlarınının değişikliği ve olası kararlılıkları için,
örnek alabilirsiniz.

API arayüzünüz kararlı ve production'a bir kez çıktıktan sonra geriye dönük 
versiyon uyumsuzlukları çıkarmamaya çalışın. Eğer böyle bir uyumsuzluk olması 
gerekiyor ise yeni bir version numarası ile API çıkarın. 


### Çeviriler
 * [Korean version](https://github.com/yoondo/http-api-design) (based on [f38dba6](https://github.com/interagent/http-api-design/commit/f38dba6fd8e2b229ab3f09cd84a8828987188863)), by [@yoondo](https://github.com/yoondo/)
 * [Simplified Chinese version](https://github.com/ZhangBohan/http-api-design-ZH_CN) (based on [337c4a0](https://github.com/interagent/http-api-design/commit/337c4a05ad08f25c5e232a72638f063925f3228a)), by [@ZhangBohan](https://github.com/ZhangBohan/)
 * [Traditional Chinese version](https://github.com/kcyeu/http-api-design) (based on [232f8dc](https://github.com/interagent/http-api-design/commit/232f8dc6a941d0b25136bf64998242dae5575f66)), by [@kcyeu](https://github.com/kcyeu/)

