# SMS Spam Tespiti

SMS mesajlarını `spam` veya `ham` (normal mesaj) olarak sınıflandıran basit bir makine öğrenmesi çalışması. TF-IDF ve Logistic Regression kullanıldı.

## Veri

- **Kaynak:** UCI SMS Spam Collection (herkese açık, İngilizce SMS mesajları). Tab ile ayrılmış, başlıksız dosya; `latin-1` kodlamasıyla okundu.
- **Boyut:** 5.572 mesaj. Bunların 403'ü tekrar eden satırdı ve silindi (5.169 mesaj kaldı).
- **Sınıf dağılımı:** Veri dengesiz. Tekrarlar silindikten sonra mesajların %87,4'ü ham, %12,6'sı spam.
- **Bölme:** Önce tekrarlar silindi, sonra veri `stratify` ile %70 / %15 / %15 olarak bölündü. Böylece her parçada spam oranı aynı (%12,6) kaldı.

| Parça | Mesaj sayısı | Kullanım |
|---|---|---|
| Train | 3.617 | Modeli eğitmek |
| Validation | 776 | Model karşılaştırma ve ayar |
| Test | 776 | Son ölçüm (yalnızca bir kez bakıldı) |

Tekrarlar bölmeden önce silindi. Aksi halde aynı mesaj hem train'de hem test'te bulunabilir ve test skoru olduğundan iyi görünürdü (veri sızıntısı).

## Yöntem

1. **Metni sayıya çevirme:** `TfidfVectorizer` (küçük harfe çevirme, 1-gram ve 2-gram, en az 2 mesajda geçen kelimeler). Sözlükte 9.128 özellik oluştu.
2. **Sızıntıyı önleme:** Vektörleştirici yalnızca train üzerinde `fit` edildi. Validation ve test'e sadece `transform` uygulandı.
3. **Model:** `LogisticRegression`. Her kelimeye bir ağırlık öğrenir; spam kelimeleri pozitif, normal mesaj kelimeleri negatif ağırlık alır. Ağırlıkların toplamı olasılığa çevrilir, 0,5 üstü spam sayılır.

## Neden accuracy yetmez?

Veri dengesiz olduğu için hiçbir şey öğrenmeden her mesaja "ham" diyen bir model bile yaklaşık %87 accuracy alır. Bu yüzden asıl ölçüt spam sınıfı için **precision**, **recall** ve **F1** oldu.

- **Precision:** "Spam" denen mesajların kaçı gerçekten spam.
- **Recall:** Gerçek spam'lerin kaçı yakalandı.
- **F1:** İkisinin dengesi.

## Model karşılaştırması (validation, spam sınıfı)

| Model | Precision | Recall | F1 |
|---|---|---|---|
| Basit (varsayılan ayarlar) | 1,000 | 0,673 | 0,805 |
| `class_weight="balanced"` | 0,917 | 0,898 | 0,907 |
| `C=10` | 1,000 | 0,847 | 0,917 |

- Basit model çok temkinli davrandı: hiç yanlış alarm vermedi ama 98 spam'in 32'sini kaçırdı.
- `balanced` daha çok spam yakaladı, bedeli yaklaşık 8 normal mesajın yanlışlıkla spam sayılması oldu (precision ile recall arasındaki denge).
- `C=10` (daha az düzenleme) precision'ı koruyarak recall'u artırdı. En yüksek F1 ve sıfır yanlış alarm nedeniyle bu model seçildi.

Validation'daki spam sayısı az (98), bu yüzden modeller arasındaki küçük farklar şansa bağlı olabilir.

## Test sonucu (`C=10`, spam sınıfı)

| Metrik | Değer |
|---|---|
| Precision | 0,98 |
| Recall | 0,85 |
| F1 | 0,91 |
| Accuracy | 0,98 |

Confusion matrix (satır: gerçek, sütun: tahmin):

|  | Tahmin: ham | Tahmin: spam |
|---|---|---|
| **Gerçek: ham** | 676 | 2 |
| **Gerçek: spam** | 15 | 83 |

Test F1 değeri (0,91) validation F1 değerine (0,92) çok yakın. Model validation'a aşırı uyum sağlamamış görünüyor.

## Hata analizi

Test setinde 15 spam kaçtı. İncelenen örneklere göre olası nedenler:

- **Nadir veya hiç görülmemiş kelimeler:** Örneğin `ringtoneking 84484` ve yazım hatalı `cal;l 09064012103 box334sk38ch`. Yalnızca bir mesajda geçen kelimeler `min_df=2` yüzünden sözlüğe girmiyor, bu da modelin bu tür mesajlara "ham" demesine yol açıyor.
- **Normal mesaj gibi yazılmış spam:** Örneğin haber başlığı gibi görünen bir mesaj (spam olasılığı 0,06). Tipik spam kelimeleri içermiyor.
- **Sınırda kalanlar:** Bazı spam'lerin olasılığı 0,34-0,40 civarında, yani 0,5 eşiğinin hemen altında kaldı.

Ayrıca 2 normal mesaj yanlışlıkla spam olarak işaretlendi.

## Sınırlılıklar ve olası iyileştirmeler

- Model kelimelerin anlamını bilmez, yalnızca hangi kelimelerin hangi sınıfla birlikte geçtiğini sayar. Yazım hatalı veya kasıtlı bozulmuş spam'ler kaçabilir.
- Karakter n-gram'ları (`analyzer="char_wb"`) nadir kelimelere ve yazım hatalarına karşı daha dayanıklı olabilir.
- Rakam sayısı, büyük harf oranı, URL varlığı ve mesaj uzunluğu gibi ek özellikler eklenebilir.
- Karar eşiği 0,5 yerine daha düşük seçilerek recall artırılabilir; bunun bedeli precision'daki düşüş olur.
- Model seçimi tek bir validation ayrımına dayandığı için k-fold çapraz doğrulama ile teyit edilmelidir.
- Veri eski ve İngilizce. Güncel ve Türkçe mesajlarda performans farklı olacaktır.

## Dosyalar

- Notebook: tüm adımları içerir (Google Colab'da çalıştırıldı).
- `test_tahminleri.csv`: Test mesajları, gerçek etiketler, tahminler ve spam olasılıkları.
- `requirements.txt`: `pandas`, `scikit-learn`.
