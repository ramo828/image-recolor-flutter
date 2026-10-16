# Renkli

Renkli, siyah-beyaz fotoğrafları cihaz üzerinde renklendiren bir Flutter
uygulamasıdır. Fotoğraf işleme internet bağlantısı olmadan da cihazda çalışır;
fotoğraflar sunucuya gönderilmez. Kaydetme, ödüllü videoyu sonuna kadar
izlemeyi ve reklamın yüklenebilmesi için internet bağlantısını gerektirir.
Fotoğraf seçildiğinde Flutter arayüzü çalışmaya devam
eder; çıkarım ayrı bir Dart isolate'ında yürütülür ve bekleme sırasında yüzde
bilgili doğrusal ilerleme göstergesi sunulur. Yüzde, işleme aşamaları
tamamlandıkça ilerler; ONNX Runtime model çıkarımının kendi içinde daha ayrıntılı
bir ilerleme callback'i sağlamaması nedeniyle bu aşamada yüzde sabit kalırken
dairesel bekleme göstergesi çalışmaya devam eder.

## Model

- **Model:** DDColor-tiny (ICCV 2023), 512×512 giriş, CIELAB `a/b` renk
  kanallarını tahmin eder.
- **Kaynak:** [DDColor-tiny ONNX model kartı](https://huggingface.co/edgetools/ddcolor)
  ve [upstream DDColor projesi](https://github.com/piddnad/DDColor).
- **Lisans:** Apache-2.0; lisans metni [assets/model/LICENSE](assets/model/LICENSE)
  içinde bulunur.
- **Quantizasyon:** Kaynak FP16 ONNX modelindeki ağırlıklar ONNX Runtime
  quantization aracı ile MatMul/Gemm işlemleri için signed INT8'e dinamik olarak
  quantize edilmiştir. Kaynak ağırlıklar 135,444,402 bayttan uygulamadaki
  quantize edilmiş 112,726,671 bayta düşmüştür (~%16.8 küçülme). Modelin
  geri kalanında CPU uyumluluğu için float işlemleri korunur.
- **Çalıştırma:** Bu model için NNAPI/GPU bazı cihazlarda CPU'dan yavaş
  olabildiğinden Android'de doğrudan ONNX Runtime CPU sağlayıcısı kullanılır.
  Intra-op iş parçacığı sayısı cihazın bildirdiği işlemci çekirdeği sayısına
  göre seçilir ve grafik optimizasyonları etkinleştirilir. Böylece işlem
  yavaş GPU/NPU sürücüsüne devredilmez; fotoğraflar hiçbir sunucuya gönderilmez.
- **Çözünürlük:** Yüksek bellek kullanımını önlemek için fotoğrafın uzun kenarı
  en fazla 2048 pikselde işlenir. Kaynak görüntünün luminance kanalı korunur;
  modelin tahmin ettiği renk kanalları yeniden ölçeklenerek birleştirilir.

Modeli yeniden üretmek için:

```sh
python3 -m pip install -r tool/requirements-model.txt
python3 tool/quantize_ddcolor.py
```

Script, indirdiği kaynak modelin SHA-256 özetini doğrular, quantize edilmiş
modeli üretir ve ONNX Runtime CPU üzerinde gerçek bir çıkarımla çıktı biçimini,
sonlu değerleri ve renk kanalının boş olmadığını kontrol eder.

## Uygulamayı çalıştırma

Güncel Flutter araç zinciri, Android/iOS araç zinciri ve Android için API 24+
gerekir (Google Mobile Ads SDK gereksinimi):

```sh
flutter pub get
flutter run
```

İlk fotoğraf seçildiğinde gömülü model uygulama verilerine kopyalanır; sonraki
çalıştırmalarda bu kopya yeniden kullanılır.

## Dil ve ayarlar

Uygulama Azerbaycanca, Türkçe, Rusça ve İngilizce arayüze sahiptir. İlk açılışta
cihaz dilini algılar; desteklenmeyen dillerde İngilizce kullanılır. Ayarlardan
bir dil seçip kaydetmek seçimi kalıcı hale getirir. "Cihaz dilini kullan"
seçeneği manuel dil tercihini silerek otomatik algılamayı geri yükler.

Ayarlar bölümünde sonuç görselinin en büyük kenarı 1024, 1536 veya 2048 piksel
olarak seçilebilir. Model her durumda 512×512 girişte çalışır; bu seçenek
yalnızca model çıktısının tam çözünürlükte birleştirildiği görsel boyutunu
belirler ve sonraki işler için kalıcı olarak saklanır.

Renklendirilmiş sonuç galerine kaydedilebilir veya cihazın paylaşım menüsüyle
başka uygulamalara gönderilebilir. Galeriye kaydetmeden önce tam ekran AdMob
ödüllü videosu gösterilir; video tamamlanıp ödül callback'i alındıktan sonra
reklam kapandığında görsel kaydı devam eder. Kullanıcı videoyu erken kapatırsa,
reklam yüklenemezse veya gösterim başarısız olursa görsel kaydedilmez ve hata
mesajı gösterilir. Reklam ağına fotoğraf gönderilmez.

## Reklamlar

Google Mobile Ads SDK entegredir. Ana ekranın altındaki sabit alanda banner,
galeriye kaydetme akışında ise tamamlanması gereken tam ekran ödüllü video
kullanılır. Kodda
şimdilik Google'ın Android/iOS **test uygulama ve reklam birimi kimlikleri**
tanımlıdır. Yayınlamadan önce Android uygulama kimliği
`android/app/src/main/AndroidManifest.xml` ve iOS uygulama kimliği
`ios/Runner/Info.plist` içinde; birim kimlikleri `lib/ad_service.dart` içinde
gerçek AdMob kimlikleriyle değiştirilmelidir. Test kimlikleri gerçek reklam
geliri oluşturmaz. Canlı reklam kimliklerine geçmeden önce yayın bölgelerine
uygun gizlilik ve kullanıcı rızası akışı da yapılandırılmalıdır.

Android release build, `RENKLI_KEYSTORE_PATH` (varsayılan:
`/home/ramo828/Downloads/your.jks`), `RENKLI_KEY_ALIAS`,
`RENKLI_STORE_PASSWORD` ve `RENKLI_KEY_PASSWORD` ortam değişkenleriyle verilen
keystore'u kullanır. Şifreleri kaynak koda veya repoya koymayın. Yerel terminal
ortamında bu değişkenleri ayarlayıp `flutter build appbundle --release` veya
`flutter build apk --release` çalıştırın. PEM dosyası sertifikayı taşır; APK
imzalama için gerekli özel anahtar JKS içinde olduğundan build ayarına PEM
yolunu eklemek gerekmez.

Parolaları komut geçmişine yazmadan terminalde girmek için:

```sh
export RENKLI_KEYSTORE_PATH=/home/ramo828/Downloads/your.jks
read -r -p "Keystore alias: " RENKLI_KEY_ALIAS
read -r -s -p "Keystore password: " RENKLI_STORE_PASSWORD; printf '\n'
read -r -s -p "Key password: " RENKLI_KEY_PASSWORD; printf '\n'
export RENKLI_KEY_ALIAS RENKLI_STORE_PASSWORD RENKLI_KEY_PASSWORD
flutter build appbundle --release
unset RENKLI_KEY_ALIAS RENKLI_STORE_PASSWORD RENKLI_KEY_PASSWORD
```

## Testler

```sh
flutter analyze
flutter test
flutter build apk --debug
```

Flutter testleri CIELAB dönüşümlerini, giriş/çıkış görüntü işlemesini ve ana
ekranı kapsar. `tool/quantize_ddcolor.py` quantize edilmiş model üzerinde CPU
çıkarımını ayrıca doğrular.
# image-recolor-flutter
