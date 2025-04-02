
# Ödeme Entegrasyonu

Bu döküman baştan sona bir react-native projesinin ödeme sisteminin entegrasyonunu içerir.




## Adımlar

- react-native-iap entegrasyonu ve mobil ödeme entegrasyonu
- Google ile yapılan ödemelerin server side kontrolü ve işlenmesi
- Google server notifications
- Apple ile yapılan ödemelerin server side kontrolü ve işlenmesi
- Apple server notifications


## react-native iap entegrasyonu

Kütüphanenin indirilmesi

```bash
  npm install react-native-iap
```

### ios

```bash
  cd ios
  pod install
```

### android

android/build.gradle içinde buildscript.ext alanına ekleme yapılmalı
 

    
```java
buildscript {
  ext {
    ...
+   supportLibVersion = "28.0.0"
  }
}
```



androidX ile kullanılacaksa android/build.gradle da eklemeler yapılmalı

```java
buildscript {
  ext {
    ...
+    androidXAnnotation = "1.1.0"
+    androidXBrowser = "1.0.0"
+    minSdkVersion = 24
+    kotlinVersion = "1.8.0"
  }
  dependencies {
    ...
+   classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlinVersion"
  }
}
```

#### Ödeme providerının belirlenmesi (android)

Play store ve amazon ile ödeme aynı anda desteklenebilir ancak şuanki durumda yalnızca google play ekliyoruz.

android/app/build.gradle pathine gidip gerekli eklemeler yapılmalı

```java
android {
    ...
    flavorDimensions "store"
    productFlavors {
        play {
            dimension "store"
        }
    }
    defaultConfig {
    ...
    + missingDimensionStrategy "store", "play"
    }
}

dependencies{
    // Debug ortamda kullanmak için aşağıdaki satırı kullan
    implementation project(path: ':react-native-iap', configuration: 'playDebugRuntimeElements')
    // Build alırken aşaıdaki satırı kullan
    // implementation project(path: ':react-native-iap')
    ...
}

...

//en aşağıya bu satırı ekle
task installDebug(dependsOn: 'installPlayDebug')

```

#### Kullanım (ios & android)
App.tsx'e gidip appi withIAPContext ile sarmallamak gerekli.

```javascript
import { setup, withIAPContext } from 'react-native-iap';
import { isIos } from './src/data/static';
...

isIos ? setup({ storekitMode: 'STOREKIT2_MODE' }) : setup();

function App() {
    ...
}

export default withIAPContext(App);

```

#### Ödeme custom hooku

```javascript
import {
    initConnection,
    purchaseErrorListener,
    purchaseUpdatedListener,
    finishTransaction,
    requestSubscription,
    flushFailedPurchasesCachedAsPendingAndroid,
    type SubscriptionPurchase,
    type ProductPurchase,
    type PurchaseError
} from 'react-native-iap';

export const usePurchaseController = () => {
    const purchaseUpdateSubscription = useRef<any>(null);
    const purchaseErrorSubscription = useRef<any>(null);
    const [loading, setLoading] = useState<boolean>(false);
    useEffect(() => {
        const initialize = async () => {
            try {
                await initConnection();
                await flushFailedPurchasesCachedAsPendingAndroid().catch(() => {
                    // Pending error olabilir, görmezden geliyoruz.
                });

                purchaseUpdateSubscription.current = purchaseUpdatedListener(
                    async (purchase: SubscriptionPurchase | ProductPurchase) => {
                        console.log('purchaseUpdatedListener', purchase);
                        if (purchase) {
                            try {
                                const deliveryResult = await yourAPI.deliverOrDownloadFancyInAppPurchase(purchase);

                                if (isSuccess(deliveryResult)) {
                                    // İşlem başarılı → finishTransaction
                                    await finishTransaction({ purchase, isConsumable: false });
                                } else {
                                    console.warn('Teslimat başarısız, sahte işlem olabilir.');
                                }
                            } catch (err) {
                                console.error('Teslimat hatası:', err);
                            }
                        }
                    }
                );

                purchaseErrorSubscription.current = purchaseErrorListener(
                    (error: PurchaseError) => {
                        console.warn('purchaseErrorListener', error);
                    }
                );
            } catch (error) {
                console.error('Bağlantı hatası:', error);
            }
        };

        initialize();

        return () => {
            if (purchaseUpdateSubscription.current) {
                purchaseUpdateSubscription.current.remove();
                purchaseUpdateSubscription.current = null;
            }
            if (purchaseErrorSubscription.current) {
                purchaseErrorSubscription.current.remove();
                purchaseErrorSubscription.current = null;
            }
        };
    }, []);


    const handlePurchase = async (sku: string, subscriptionOffers: any) => {
        try {
            setLoading(true);
            await requestSubscription(
                !isEmpty(subscriptionOffers)
                    ? {
                        sku,
                        subscriptionOffers
                    }
                    : { sku }
            );
        } catch (err) {
            console.error('Satın alma başlatılırken hata:', err);
        } finally {
            setLoading(false);
        }
    };

    return { loading, handlePurchase }
}
```

## Google ile yapılan ödemelerin server side kontrolü ve işlenmesi

Goole ile yapılan ödemelerden sonra gelen örnek purchase datası şu şekildedir

```json
{
  "purchase": {
    "autoRenewingAndroid": true,
    "dataAndroid": {
      "orderId": "GPA.3360-0857-7484-28488",
      "packageName": "com.armakom.qring.net",
      "productId": "qring_gold",
      "purchaseTime": 1743150758779,
      "purchaseState": 0,
      "purchaseToken": "cdkokcfcohmjblgcjkidgbok.AO-J1OxRa_DBhojWf2A0ydvNH-Xqir7pmCwnStkxlfJ8sV5ggmEBtx4aWY9wftAN9QbtK0z4XLASaXQRdLM3eYcTGDz1EYZlByAx6miXxNl6YrHi71tWXg0",
      "quantity": 1,
      "autoRenewing": true,
      "acknowledged": false
    },
    "developerPayloadAndroid": "",
    "isAcknowledgedAndroid": false,
    "obfuscatedAccountIdAndroid": "",
    "obfuscatedProfileIdAndroid": "",
    "packageNameAndroid": "com.armakom.qring.net",
    "productId": "qring_gold",
    "productIds": [
      "qring_gold"
    ],
    "purchaseStateAndroid": 1,
    "purchaseToken": "cdkokcfcohmjblgcjkidgbok.AO-J1OxRa_DBhojWf2A0ydvNH-Xqir7pmCwnStkxlfJ8sV5ggmEBtx4aWY9wftAN9QbtK0z4XLASaXQRdLM3eYcTGDz1EYZlByAx6miXxNl6YrHi71tWXg0",
    "signatureAndroid": "OtPu4wg9wnNK7EUlXihrdENkMuDbzSVacMUkD5e89sYA4ZpMiCY3oHYlRUG42dfpxEyXxrCNE+CR5cu9U42pBAMGW/RdxDmmWnB8/e3YilYsDAJMZaU3OkZgg7259A+rALnlT4Oxx5eiHgxrZ4X2k7K/x8vvOgpWYkwpOuUnTKniiwlo1Jf9kv4WqDuWkMqOlwak6vMzEeod9sWWBfYUkqA785qIIqb4+bEjRT6WFaePWsKpAdBrIbdGM2V75e1sk/0RD8fI92ITzigoyVqYomx2v5j/VufsXwIyypRBSD48W26YLkF4HWAT/jEgbPVf8iSxP1to+W82MzFcglINrQ==",
    "transactionDate": 1743150758779,
    "transactionId": "GPA.3360-0857-7484-28488",
    "transactionReceipt": {
      "orderId": "GPA.3360-0857-7484-28488",
      "packageName": "com.armakom.qring.net",
      "productId": "qring_gold",
      "purchaseTime": 1743150758779,
      "purchaseState": 0,
      "purchaseToken": "cdkokcfcohmjblgcjkidgbok.AO-J1OxRa_DBhojWf2A0ydvNH-Xqir7pmCwnStkxlfJ8sV5ggmEBtx4aWY9wftAN9QbtK0z4XLASaXQRdLM3eYcTGDz1EYZlByAx6miXxNl6YrHi71tWXg0",
      "quantity": 1,
      "autoRenewing": true,
      "acknowledged": false
    }
  }
}
```

buradaki  transactionId, purchaseToken bilgileri mutlaka apiye gönderilmelidir. Currency bilgisi ios ve android için bu kısımda gelmez. Server sideda alınması gerek.
Api tarafında bu ödemenin kontrolünü yapabilmek için [Google Cloud Console](https://console.cloud.google.com)'a giderek appinize bağlamak üzere bir proje oluşturun.

Google Play Android Developer API etkinleştirdikten sonra

IAM & Aadmin sekmesine gidin

Sol Menüden Service Accounts seçin ve bir servis account oluşturun

Sol Menüden IAM seçin ve oluşturduğunuz service accountu seçtikten sonra Grant access ile yetkileri düzenleyin şu yetkiler verilmeli:

- Application Admin
- Owner
- Pub/Sub Admin
- Financial Services Admin
- Editor
- Android Management User

Sol Menüden Service Accounts seçin ve oluşturduğunuz service accounts'un menüsünden "Manage keys" seçilerek Add key> Create new key seçin. İnen json dosyası google apiye istek atarken kullanılacak olan jwt tokenın claimlerini içermektedir.

Bir sonraki adım olarak 
- [Google Play Console](https://play.google.com/console)'a giriş yapın
- Sol menüden Users and permissions seçin
- Invite users seçin
- Cloud Console'dan oluşturduğunuz servisin adresini ekleyin ve yetkilerini verin

Servisiniz'e ait indirdiğiniz key dosyası ile ödemenin kontorlünün sağlandığı örnek python kodu aşağıdaki gibidir.

```python
import json
import requests
from google.oauth2 import service_account
from google.auth.transport.requests import Request  # ✅ Doğru import

# 📌 Service Account JSON dosyanızın yolu
SERVICE_ACCOUNT_FILE = "google_play_api.json"  # Buraya dosyanın adını yazın

# 📌 Doğrulama için gerekli Google API scope
SCOPES = ["https://www.googleapis.com/auth/androidpublisher"]

# 📌 Google Play API bilgileri
PACKAGE_NAME = "com.paketadi"  # Uygulamanızın paket adı
PRODUCT_ID = "productId"  # Ürününüzün ID'si
PURCHASE_TOKEN = "bgklsdhfıeıfhsndmnfmnm...."  # Doğrulamak istediğiniz satın alma token'ı

def get_access_token():
    """Google Service Account ile access token oluşturur"""
    credentials = service_account.Credentials.from_service_account_file(
        SERVICE_ACCOUNT_FILE, scopes=SCOPES
    )

    # Eğer token süresi dolmuşsa yenile
    if not credentials.valid:
        credentials.refresh(Request())

    return credentials.token

def verify_purchase(access_token):
    """Google Play API'ye istek atarak satın almayı doğrular"""
    url = f"https://androidpublisher.googleapis.com/androidpublisher/v3/applications/{PACKAGE_NAME}/purchases/subscriptions/{PRODUCT_ID}/tokens/{PURCHASE_TOKEN}"
    
    headers = {"Authorization": f"Bearer {access_token}"}
    
    response = requests.get(url, headers=headers)
    
    if response.status_code == 200:
        data = response.json()
        print("✅ Satın Alma Başarılı:", json.dumps(data, indent=4))
    else:
        print("❌ Hata:", response.status_code, response.text)

if __name__ == "__main__":
    token = get_access_token()
    verify_purchase(token)
```

Gelen başarılı response datası aşağıdaki gibidir

```json
{
    "startTimeMillis": "1742985723596",
    "expiryTimeMillis": "1742998325049",
    "autoRenewing": false,
    "priceCurrencyCode": "TRY",
    "priceAmountMicros": "199990000",
    "countryCode": "TR",
    "developerPayload": "",
    "cancelReason": 1,
    "orderId": "GPA.3385-3910-4138-23422..5",
    "purchaseType": 0,
    "acknowledgementState": 1,
    "kind": "androidpublisher#subscriptionPurchase"
}
```

Gelecek datanın en kapsamlı modeli ise aşağıdaki gibidir

```typescript
{
  "kind": string,
  "startTimeMillis": string,
  "expiryTimeMillis": string,
  "autoResumeTimeMillis": string,
  "autoRenewing": boolean,
  "priceCurrencyCode": string,
  "priceAmountMicros": string,
  "introductoryPriceInfo": {
    "introductoryPriceCurrencyCode": string,
    "introductoryPriceAmountMicros": string,
    "introductoryPricePeriod": string,
    "introductoryPriceCycles": number
  },
  "countryCode": string,
  "developerPayload": string,
  "paymentState": number,
  "cancelReason": number,
  "userCancellationTimeMillis": string,
  "cancelSurveyResult": {
    "cancelSurveyReason": number,
    "userInputCancelReason": string
  },
  "orderId": string,
  "linkedPurchaseToken": string,
  "purchaseType": number,
  "priceChange": {
    "newPrice": {
        "priceMicros": string,
        "currency": string
    },
    "state": number
  },
  "profileName": string,
  "emailAddress": string,
  "givenName": string,
  "familyName": string,
  "profileId": string,
  "acknowledgementState": number,
  "externalAccountId": string,
  "promotionType": number,
  "promotionCode": string,
  "obfuscatedExternalAccountId": string,
  "obfuscatedExternalProfileId": string
}
```

İlgili alanların açıklamaları linkte verilmiştir

[Resource: SubscriptionPurchase](https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.subscriptions#resource:-subscriptionpurchase)

## Google server notifications

Bir abonelik süresi bitiminde ödeme tekrarlandığında google'ın otomatik olarak sizin belirlediğiniz bir endpointe istek atıyor olması gerekir. Bu yüzden bir typeı Post olan bir endpoint hazırlayınız.

endpointe gelecek örnek json datası şu şekildedir:

```json
{
  "Message": {
    "Data": "eyJ2ZXJzaW9uIjoiMS4wIiwicGFja2FnZU5hbWUiOiJjb20uYXJtYWtvbS5xcmluZy5uZXQiLCJldmVudFRpbWVNaWxsaXMiOiIxNzQyOTc3NjQ2NjIxIiwic3Vic2NyaXB0aW9uTm90aWZpY2F0aW9uIjp7InZlcnNpb24iOiIxLjAiLCJub3RpZmljYXRpb25UeXBlIjo0LCJwdXJjaGFzZVRva2VuIjoicHBvZGlsb2Rva25jZGduZW1lbmJwbGdlLkFPLUoxT3pDWTh2c1FGa0FVYW83VEtJSGlnVzhPRVZWZjd2SXpvLUF3bkQyRHJ0MG1iSXdWMktDUG5ZM3BKcWlnbVFEaGcwRHRmZEhjZFJGSmVwYnBqUk5Uei1yRlQxX2dKbU9zMklqWGV4ZG9yaGd6cVRZNHk4Iiwic3Vic2NyaXB0aW9uSWQiOiJxcmluZ19nb2xkIn19",
    "MessageId": "14346610825415280",
    "Message_Id": "14346610825415280",
    "PublishTime": "2025-03-26T08:27:26.795Z",
    "Publish_Time": "2025-03-26T08:27:26.795Z"
  },
  "Subscription": "projects/qringmobileapp/subscriptions/qring_gold"
}
```

Buradaki data propertysi ile gelen token decode edilir. Decode edildikten sonra gelen elde edilen data:

```json
{
  "version": "1.0",
  "packageName": "com.armakom.qring.net",
  "eventTimeMillis": "1742977646621",
  "subscriptionNotification": {
    "version": "1.0",
    "notificationType": 4,
    "purchaseToken": "ppodilodokncdgnemenbplge.AO-J1OzCY8vsQFkAUao7TKIHigW8OEVVf7vIzo-AwnD2Drt0mbIwV2KCPnY3pJqigmQDhg0DtfdHcdRFJepbpjRNTz-rFT1_gJmOs2IjXexdorhgzqTY4y8",
    "subscriptionId": "qring_gold"
  }
}
```

Elde edilen yukarıdaki datadaki purchaseToken ve subscriptionId kullanılarak google apiye gidilir ve ödeme detaylarına ulaşılır. Örnek python kodu (ödeme validasyonu ile aynı) Aşağıdaki gibidir: 

```python
import json
import requests
from google.oauth2 import service_account
from google.auth.transport.requests import Request  # ✅ Doğru import

# 📌 Service Account JSON dosyanızın yolu
SERVICE_ACCOUNT_FILE = "google_play_api.json"  # Buraya dosyanın adını yazın

# 📌 Doğrulama için gerekli Google API scope
SCOPES = ["https://www.googleapis.com/auth/androidpublisher"]

# 📌 Google Play API bilgileri
PACKAGE_NAME = "com.paketadi"  # Uygulamanızın paket adı
PRODUCT_ID = "productId"  # Ürününüzün ID'si
PURCHASE_TOKEN = "bgklsdhfıeıfhsndmnfmnm...."  # Doğrulamak istediğiniz satın alma token'ı

def get_access_token():
    """Google Service Account ile access token oluşturur"""
    credentials = service_account.Credentials.from_service_account_file(
        SERVICE_ACCOUNT_FILE, scopes=SCOPES
    )

    # Eğer token süresi dolmuşsa yenile
    if not credentials.valid:
        credentials.refresh(Request())

    return credentials.token

def verify_purchase(access_token):
    """Google Play API'ye istek atarak satın almayı doğrular"""
    url = f"https://androidpublisher.googleapis.com/androidpublisher/v3/applications/{PACKAGE_NAME}/purchases/subscriptions/{PRODUCT_ID}/tokens/{PURCHASE_TOKEN}"
    
    headers = {"Authorization": f"Bearer {access_token}"}
    
    response = requests.get(url, headers=headers)
    
    if response.status_code == 200:
        data = response.json()
        print("✅ Satın Alma Başarılı:", json.dumps(data, indent=4))
    else:
        print("❌ Hata:", response.status_code, response.text)

if __name__ == "__main__":
    token = get_access_token()
    verify_purchase(token)
```



Gelecek datanın en kapsamlı modeli ise aşağıdaki gibidir

```typescript
{
  "kind": string,
  "startTimeMillis": string,
  "expiryTimeMillis": string,
  "autoResumeTimeMillis": string,
  "autoRenewing": boolean,
  "priceCurrencyCode": string,
  "priceAmountMicros": string,
  "introductoryPriceInfo": {
    "introductoryPriceCurrencyCode": string,
    "introductoryPriceAmountMicros": string,
    "introductoryPricePeriod": string,
    "introductoryPriceCycles": number
  },
  "countryCode": string,
  "developerPayload": string,
  "paymentState": number,
  "cancelReason": number,
  "userCancellationTimeMillis": string,
  "cancelSurveyResult": {
    "cancelSurveyReason": number,
    "userInputCancelReason": string
  },
  "orderId": string,
  "linkedPurchaseToken": string,
  "purchaseType": number,
  "priceChange": {
    "newPrice": {
        "priceMicros": string,
        "currency": string
    },
    "state": number
  },
  "profileName": string,
  "emailAddress": string,
  "givenName": string,
  "familyName": string,
  "profileId": string,
  "acknowledgementState": number,
  "externalAccountId": string,
  "promotionType": number,
  "promotionCode": string,
  "obfuscatedExternalAccountId": string,
  "obfuscatedExternalProfileId": string
}
```

İlgili alanların açıklamaları linkte verilmiştir

[Resource: SubscriptionPurchase](https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.subscriptions#resource:-subscriptionpurchase)

Buradaki verilere göre action alınır.

## Apple ile yapılan ödemelerin server side kontrolü ve işlenmesi

Apple ile yapılan ödemelerin valid olup olmadığını kontrol etmek için herhangi bir apiye istek atmaya ihtiyaç duyulmaz. Ödemeden sonra elde edilen token apple tarafından imzalanmış bir tokendır ve taklit edilemez. Bu dökümanda serverside kontrolü server notifications ile gerçekleştirilmiştir. 

## Apple server notifications

Server notifications'ı aktif etmek için [Appstore Connect](https://appstoreconnect.apple.com)'e giriş yapın. İlgili uygulamanızı seçtikten sonra Distribution sekmesinden, sol menüde buluna App Information'ı seçin. App Store Server Notifications section'ında bulunan alanlara ilgili endpointleri yazın. 

Ödemelerden sonra apple tarafından endpointe gelen örnek datada yalnızca bir token bulunur.

örnek json:

```json
{
  "signedPayload": "eyJhbGciOiJFUzI1NiIsIng1YyI6WyJNSUlFTURDQ0E3YWdBd0lCQWdJUWZUbGZkMGZOdkZXdnpDMVlJQU5zWGpBS0JnZ3Foa2pPUFFRREF6QjFNVVF3UWdZRFZRUURERHRCY0hCc1pTQlhiM0pzWkhkcFpHVWdSR1YyWld4dmNHVnlJRkpsYkdGMGFXOXVjeUJEWlhKMGFXWnBZMkYwYVc5dUlFRjFkR2h2Y21sMGVURUxNQWtHQTFVRUN3d0NSell4RXpBUkJnTlZCQW9NQ2tGd2NHeGxJRWx1WXk0eEN6QUpCZ05WQkFZVEFsVlRNQjRYRFRJek1Ea3hNakU1TlRFMU0xb1hEVEkxTVRBeE1URTVOVEUxTWxvd2daSXhRREErQmdOVkJBTU1OMUJ5YjJRZ1JVTkRJRTFoWXlCQmNIQWdVM1J2Y21VZ1lXNWtJR2xVZFc1bGN5QlRkRzl5WlNCU1pXTmxhWEIwSUZOcFoyNXBibWN4TERBcUJnTlZCQXNNSTBGd2NHeGxJRmR2Y214a2QybGtaU0JFWlhabGJHOXdaWElnVW1Wc1lYUnBiMjV6TVJNd0VRWURWUVFLREFwQmNIQnNaU0JKYm1NdU1Rc3dDUVlEVlFRR0V3SlZVekJaTUJNR0J5cUdTTTQ5QWdFR0NDcUdTTTQ5QXdFSEEwSUFCRUZFWWUvSnFUcXlRdi9kdFhrYXVESENTY1YxMjlGWVJWLzB4aUIyNG5DUWt6UWYzYXNISk9OUjVyMFJBMGFMdko0MzJoeTFTWk1vdXZ5ZnBtMjZqWFNqZ2dJSU1JSUNCREFNQmdOVkhSTUJBZjhFQWpBQU1COEdBMVVkSXdRWU1CYUFGRDh2bENOUjAxREptaWc5N2JCODVjK2xrR0taTUhBR0NDc0dBUVVGQndFQkJHUXdZakF0QmdnckJnRUZCUWN3QW9ZaGFIUjBjRG92TDJObGNuUnpMbUZ3Y0d4bExtTnZiUzkzZDJSeVp6WXVaR1Z5TURFR0NDc0dBUVVGQnpBQmhpVm9kSFJ3T2k4dmIyTnpjQzVoY0hCc1pTNWpiMjB2YjJOemNEQXpMWGQzWkhKbk5qQXlNSUlCSGdZRFZSMGdCSUlCRlRDQ0FSRXdnZ0VOQmdvcWhraUc5Mk5rQlFZQk1JSCtNSUhEQmdnckJnRUZCUWNDQWpDQnRneUJzMUpsYkdsaGJtTmxJRzl1SUhSb2FYTWdZMlZ5ZEdsbWFXTmhkR1VnWW5rZ1lXNTVJSEJoY25SNUlHRnpjM1Z0WlhNZ1lXTmpaWEIwWVc1alpTQnZaaUIwYUdVZ2RHaGxiaUJoY0hCc2FXTmhZbXhsSUhOMFlXNWtZWEprSUhSbGNtMXpJR0Z1WkNCamIyNWthWFJwYjI1eklHOW1JSFZ6WlN3Z1kyVnlkR2xtYVdOaGRHVWdjRzlzYVdONUlHRnVaQ0JqWlhKMGFXWnBZMkYwYVc5dUlIQnlZV04wYVdObElITjBZWFJsYldWdWRITXVNRFlHQ0NzR0FRVUZCd0lCRmlwb2RIUndPaTh2ZDNkM0xtRndjR3hsTG1OdmJTOWpaWEowYVdacFkyRjBaV0YxZEdodmNtbDBlUzh3SFFZRFZSME9CQllFRkFNczhQanM2VmhXR1FsekUyWk9FK0dYNE9vL01BNEdBMVVkRHdFQi93UUVBd0lIZ0RBUUJnb3Foa2lHOTJOa0Jnc0JCQUlGQURBS0JnZ3Foa2pPUFFRREF3Tm9BREJsQWpFQTh5Uk5kc2twNTA2REZkUExnaExMSndBdjVKOGhCR0xhSThERXhkY1BYK2FCS2pqTzhlVW85S3BmcGNOWVVZNVlBakFQWG1NWEVaTCtRMDJhZHJtbXNoTnh6M05uS20rb3VRd1U3dkJUbjBMdmxNN3ZwczJZc2xWVGFtUllMNGFTczVrPSIsIk1JSURGakNDQXB5Z0F3SUJBZ0lVSXNHaFJ3cDBjMm52VTRZU3ljYWZQVGp6Yk5jd0NnWUlLb1pJemowRUF3TXdaekViTUJrR0ExVUVBd3dTUVhCd2JHVWdVbTl2ZENCRFFTQXRJRWN6TVNZd0pBWURWUVFMREIxQmNIQnNaU0JEWlhKMGFXWnBZMkYwYVc5dUlFRjFkR2h2Y21sMGVURVRNQkVHQTFVRUNnd0tRWEJ3YkdVZ1NXNWpMakVMTUFrR0ExVUVCaE1DVlZNd0hoY05NakV3TXpFM01qQXpOekV3V2hjTk16WXdNekU1TURBd01EQXdXakIxTVVRd1FnWURWUVFERER0QmNIQnNaU0JYYjNKc1pIZHBaR1VnUkdWMlpXeHZjR1Z5SUZKbGJHRjBhVzl1Y3lCRFpYSjBhV1pwWTJGMGFXOXVJRUYxZEdodmNtbDBlVEVMTUFrR0ExVUVDd3dDUnpZeEV6QVJCZ05WQkFvTUNrRndjR3hsSUVsdVl5NHhDekFKQmdOVkJBWVRBbFZUTUhZd0VBWUhLb1pJemowQ0FRWUZLNEVFQUNJRFlnQUVic1FLQzk0UHJsV21aWG5YZ3R4emRWSkw4VDBTR1luZ0RSR3BuZ24zTjZQVDhKTUViN0ZEaTRiQm1QaENuWjMvc3E2UEYvY0djS1hXc0w1dk90ZVJoeUo0NXgzQVNQN2NPQithYW85MGZjcHhTdi9FWkZibmlBYk5nWkdoSWhwSW80SDZNSUgzTUJJR0ExVWRFd0VCL3dRSU1BWUJBZjhDQVFBd0h3WURWUjBqQkJnd0ZvQVV1N0Rlb1ZnemlKcWtpcG5ldnIzcnI5ckxKS3N3UmdZSUt3WUJCUVVIQVFFRU9qQTRNRFlHQ0NzR0FRVUZCekFCaGlwb2RIUndPaTh2YjJOemNDNWhjSEJzWlM1amIyMHZiMk56Y0RBekxXRndjR3hsY205dmRHTmhaek13TndZRFZSMGZCREF3TGpBc29DcWdLSVltYUhSMGNEb3ZMMk55YkM1aGNIQnNaUzVqYjIwdllYQndiR1Z5YjI5MFkyRm5NeTVqY213d0hRWURWUjBPQkJZRUZEOHZsQ05SMDFESm1pZzk3YkI4NWMrbGtHS1pNQTRHQTFVZER3RUIvd1FFQXdJQkJqQVFCZ29xaGtpRzkyTmtCZ0lCQkFJRkFEQUtCZ2dxaGtqT1BRUURBd05vQURCbEFqQkFYaFNxNUl5S29nTUNQdHc0OTBCYUI2NzdDYUVHSlh1ZlFCL0VxWkdkNkNTamlDdE9udU1UYlhWWG14eGN4ZmtDTVFEVFNQeGFyWlh2TnJreFUzVGtVTUkzM3l6dkZWVlJUNHd4V0pDOTk0T3NkY1o0K1JHTnNZRHlSNWdtZHIwbkRHZz0iLCJNSUlDUXpDQ0FjbWdBd0lCQWdJSUxjWDhpTkxGUzVVd0NnWUlLb1pJemowRUF3TXdaekViTUJrR0ExVUVBd3dTUVhCd2JHVWdVbTl2ZENCRFFTQXRJRWN6TVNZd0pBWURWUVFMREIxQmNIQnNaU0JEWlhKMGFXWnBZMkYwYVc5dUlFRjFkR2h2Y21sMGVURVRNQkVHQTFVRUNnd0tRWEJ3YkdVZ1NXNWpMakVMTUFrR0ExVUVCaE1DVlZNd0hoY05NVFF3TkRNd01UZ3hPVEEyV2hjTk16a3dORE13TVRneE9UQTJXakJuTVJzd0dRWURWUVFEREJKQmNIQnNaU0JTYjI5MElFTkJJQzBnUnpNeEpqQWtCZ05WQkFzTUhVRndjR3hsSUVObGNuUnBabWxqWVhScGIyNGdRWFYwYUc5eWFYUjVNUk13RVFZRFZRUUtEQXBCY0hCc1pTQkpibU11TVFzd0NRWURWUVFHRXdKVlV6QjJNQkFHQnlxR1NNNDlBZ0VHQlN1QkJBQWlBMklBQkpqcEx6MUFjcVR0a3lKeWdSTWMzUkNWOGNXalRuSGNGQmJaRHVXbUJTcDNaSHRmVGpqVHV4eEV0WC8xSDdZeVlsM0o2WVJiVHpCUEVWb0EvVmhZREtYMUR5eE5CMGNUZGRxWGw1ZHZNVnp0SzUxN0lEdll1VlRaWHBta09sRUtNYU5DTUVBd0hRWURWUjBPQkJZRUZMdXczcUZZTTRpYXBJcVozcjY5NjYvYXl5U3JNQThHQTFVZEV3RUIvd1FGTUFNQkFmOHdEZ1lEVlIwUEFRSC9CQVFEQWdFR01Bb0dDQ3FHU000OUJBTURBMmdBTUdVQ01RQ0Q2Y0hFRmw0YVhUUVkyZTN2OUd3T0FFWkx1Tit5UmhIRkQvM21lb3locG12T3dnUFVuUFdUeG5TNGF0K3FJeFVDTUcxbWloREsxQTNVVDgyTlF6NjBpbU9sTTI3amJkb1h0MlFmeUZNbStZaGlkRGtMRjF2TFVhZ002QmdENTZLeUtBPT0iXX0.eyJub3RpZmljYXRpb25UeXBlIjoiRElEX0NIQU5HRV9SRU5FV0FMX1BSRUYiLCJzdWJ0eXBlIjoiRE9XTkdSQURFIiwibm90aWZpY2F0aW9uVVVJRCI6ImYxOTg1MmViLWE4ZGQtNDAyMy1iNDljLTk0OWNhZGJlZTg0YyIsImRhdGEiOnsiYXBwQXBwbGVJZCI6NjczODkyNDQyMSwiYnVuZGxlSWQiOiJjb20uYXJtYWtvbS5xcmluZy5uZXQiLCJidW5kbGVWZXJzaW9uIjoiNiIsImVudmlyb25tZW50IjoiU2FuZGJveCIsInNpZ25lZFRyYW5zYWN0aW9uSW5mbyI6ImV5SmhiR2NpT2lKRlV6STFOaUlzSW5nMVl5STZXeUpOU1VsRlRVUkRRMEUzWVdkQmQwbENRV2RKVVdaVWJHWmtNR1pPZGtaWGRucERNVmxKUVU1eldHcEJTMEpuWjNGb2EycFBVRkZSUkVGNlFqRk5WVkYzVVdkWlJGWlJVVVJFUkhSQ1kwaENjMXBUUWxoaU0wcHpXa2hrY0ZwSFZXZFNSMVl5V2xkNGRtTkhWbmxKUmtwc1lrZEdNR0ZYT1hWamVVSkVXbGhLTUdGWFduQlpNa1l3WVZjNWRVbEZSakZrUjJoMlkyMXNNR1ZVUlV4TlFXdEhRVEZWUlVOM2QwTlNlbGw0UlhwQlVrSm5UbFpDUVc5TlEydEdkMk5IZUd4SlJXeDFXWGswZUVONlFVcENaMDVXUWtGWlZFRnNWbFJOUWpSWVJGUkplazFFYTNoTmFrVTFUbFJGTVUweGIxaEVWRWt4VFZSQmVFMVVSVFZPVkVVeFRXeHZkMmRhU1hoUlJFRXJRbWRPVmtKQlRVMU9NVUo1WWpKUloxSlZUa1JKUlRGb1dYbENRbU5JUVdkVk0xSjJZMjFWWjFsWE5XdEpSMnhWWkZjMWJHTjVRbFJrUnpsNVdsTkNVMXBYVG14aFdFSXdTVVpPY0ZveU5YQmliV040VEVSQmNVSm5UbFpDUVhOTlNUQkdkMk5IZUd4SlJtUjJZMjE0YTJReWJHdGFVMEpGV2xoYWJHSkhPWGRhV0VsblZXMVdjMWxZVW5CaU1qVjZUVkpOZDBWUldVUldVVkZMUkVGd1FtTklRbk5hVTBKS1ltMU5kVTFSYzNkRFVWbEVWbEZSUjBWM1NsWlZla0phVFVKTlIwSjVjVWRUVFRRNVFXZEZSME5EY1VkVFRUUTVRWGRGU0VFd1NVRkNSVVpGV1dVdlNuRlVjWGxSZGk5a2RGaHJZWFZFU0VOVFkxWXhNamxHV1ZKV0x6QjRhVUl5Tkc1RFVXdDZVV1l6WVhOSVNrOU9ValZ5TUZKQk1HRk1ka28wTXpKb2VURlRXazF2ZFhaNVpuQnRNalpxV0ZOcVoyZEpTVTFKU1VOQ1JFRk5RbWRPVmtoU1RVSkJaamhGUVdwQlFVMUNPRWRCTVZWa1NYZFJXVTFDWVVGR1JEaDJiRU5PVWpBeFJFcHRhV2M1TjJKQ09EVmpLMnhyUjB0YVRVaEJSME5EYzBkQlVWVkdRbmRGUWtKSFVYZFpha0YwUW1kbmNrSm5SVVpDVVdOM1FXOVphR0ZJVWpCalJHOTJUREpPYkdOdVVucE1iVVozWTBkNGJFeHRUblppVXprelpESlNlVnA2V1hWYVIxWjVUVVJGUjBORGMwZEJVVlZHUW5wQlFtaHBWbTlrU0ZKM1QyazRkbUl5VG5walF6Vm9ZMGhDYzFwVE5XcGlNakIyWWpKT2VtTkVRWHBNV0dReldraEtiazVxUVhsTlNVbENTR2RaUkZaU01HZENTVWxDUmxSRFEwRlNSWGRuWjBWT1FtZHZjV2hyYVVjNU1rNXJRbEZaUWsxSlNDdE5TVWhFUW1kbmNrSm5SVVpDVVdORFFXcERRblJuZVVKek1VcHNZa2RzYUdKdFRteEpSemwxU1VoU2IyRllUV2RaTWxaNVpFZHNiV0ZYVG1oa1IxVm5XVzVyWjFsWE5UVkpTRUpvWTI1U05VbEhSbnBqTTFaMFdsaE5aMWxYVG1wYVdFSXdXVmMxYWxwVFFuWmFhVUl3WVVkVloyUkhhR3hpYVVKb1kwaENjMkZYVG1oWmJYaHNTVWhPTUZsWE5XdFpXRXByU1VoU2JHTnRNWHBKUjBaMVdrTkNhbUl5Tld0aFdGSndZakkxZWtsSE9XMUpTRlo2V2xOM1oxa3lWbmxrUjJ4dFlWZE9hR1JIVldkalJ6bHpZVmRPTlVsSFJuVmFRMEpxV2xoS01HRlhXbkJaTWtZd1lWYzVkVWxJUW5sWlYwNHdZVmRPYkVsSVRqQlpXRkpzWWxkV2RXUklUWFZOUkZsSFEwTnpSMEZSVlVaQ2QwbENSbWx3YjJSSVVuZFBhVGgyWkROa00weHRSbmRqUjNoc1RHMU9kbUpUT1dwYVdFb3dZVmRhY0ZreVJqQmFWMFl4WkVkb2RtTnRiREJsVXpoM1NGRlpSRlpTTUU5Q1FsbEZSa0ZOY3poUWFuTTJWbWhYUjFGc2VrVXlXazlGSzBkWU5FOXZMMDFCTkVkQk1WVmtSSGRGUWk5M1VVVkJkMGxJWjBSQlVVSm5iM0ZvYTJsSE9USk9hMEpuYzBKQ1FVbEdRVVJCUzBKblozRm9hMnBQVUZGUlJFRjNUbTlCUkVKc1FXcEZRVGg1VWs1a2MydHdOVEEyUkVaa1VFeG5hRXhNU25kQmRqVktPR2hDUjB4aFNUaEVSWGhrWTFCWUsyRkNTMnBxVHpobFZXODVTM0JtY0dOT1dWVlpOVmxCYWtGUVdHMU5XRVZhVEN0Uk1ESmhaSEp0YlhOb1RuaDZNMDV1UzIwcmIzVlJkMVUzZGtKVWJqQk1kbXhOTjNad2N6SlpjMnhXVkdGdFVsbE1OR0ZUY3pWclBTSXNJazFKU1VSR2FrTkRRWEI1WjBGM1NVSkJaMGxWU1hOSGFGSjNjREJqTW01MlZUUlpVM2xqWVdaUVZHcDZZazVqZDBObldVbExiMXBKZW1vd1JVRjNUWGRhZWtWaVRVSnJSMEV4VlVWQmQzZFRVVmhDZDJKSFZXZFZiVGwyWkVOQ1JGRlRRWFJKUldONlRWTlpkMHBCV1VSV1VWRk1SRUl4UW1OSVFuTmFVMEpFV2xoS01HRlhXbkJaTWtZd1lWYzVkVWxGUmpGa1IyaDJZMjFzTUdWVVJWUk5Ra1ZIUVRGVlJVTm5kMHRSV0VKM1lrZFZaMU5YTldwTWFrVk1UVUZyUjBFeFZVVkNhRTFEVmxaTmQwaG9ZMDVOYWtWM1RYcEZNMDFxUVhwT2VrVjNWMmhqVGsxNldYZE5la1UxVFVSQmQwMUVRWGRYYWtJeFRWVlJkMUZuV1VSV1VWRkVSRVIwUW1OSVFuTmFVMEpZWWpOS2MxcElaSEJhUjFWblVrZFdNbHBYZUhaalIxWjVTVVpLYkdKSFJqQmhWemwxWTNsQ1JGcFlTakJoVjFwd1dUSkdNR0ZYT1hWSlJVWXhaRWRvZG1OdGJEQmxWRVZNVFVGclIwRXhWVVZEZDNkRFVucFplRVY2UVZKQ1owNVdRa0Z2VFVOclJuZGpSM2hzU1VWc2RWbDVOSGhEZWtGS1FtZE9Wa0pCV1ZSQmJGWlVUVWhaZDBWQldVaExiMXBKZW1vd1EwRlJXVVpMTkVWRlFVTkpSRmxuUVVWaWMxRkxRemswVUhKc1YyMWFXRzVZWjNSNGVtUldTa3c0VkRCVFIxbHVaMFJTUjNCdVoyNHpUalpRVkRoS1RVVmlOMFpFYVRSaVFtMVFhRU51V2pNdmMzRTJVRVl2WTBkalMxaFhjMHcxZGs5MFpWSm9lVW8wTlhnelFWTlFOMk5QUWl0aFlXODVNR1pqY0hoVGRpOUZXa1ppYm1sQllrNW5Xa2RvU1dod1NXODBTRFpOU1VnelRVSkpSMEV4VldSRmQwVkNMM2RSU1UxQldVSkJaamhEUVZGQmQwaDNXVVJXVWpCcVFrSm5kMFp2UVZWMU4wUmxiMVpuZW1sS2NXdHBjRzVsZG5JemNuSTVja3hLUzNOM1VtZFpTVXQzV1VKQ1VWVklRVkZGUlU5cVFUUk5SRmxIUTBOelIwRlJWVVpDZWtGQ2FHbHdiMlJJVW5kUGFUaDJZakpPZW1ORE5XaGpTRUp6V2xNMWFtSXlNSFppTWs1NlkwUkJla3hYUm5kalIzaHNZMjA1ZG1SSFRtaGFlazEzVG5kWlJGWlNNR1pDUkVGM1RHcEJjMjlEY1dkTFNWbHRZVWhTTUdORWIzWk1NazU1WWtNMWFHTklRbk5hVXpWcVlqSXdkbGxZUW5kaVIxWjVZakk1TUZreVJtNU5lVFZxWTIxM2QwaFJXVVJXVWpCUFFrSlpSVVpFT0hac1EwNVNNREZFU20xcFp6azNZa0k0TldNcmJHdEhTMXBOUVRSSFFURlZaRVIzUlVJdmQxRkZRWGRKUWtKcVFWRkNaMjl4YUd0cFJ6a3lUbXRDWjBsQ1FrRkpSa0ZFUVV0Q1oyZHhhR3RxVDFCUlVVUkJkMDV2UVVSQ2JFRnFRa0ZZYUZOeE5VbDVTMjluVFVOUWRIYzBPVEJDWVVJMk56ZERZVVZIU2xoMVpsRkNMMFZ4V2tka05rTlRhbWxEZEU5dWRVMVVZbGhXV0cxNGVHTjRabXREVFZGRVZGTlFlR0Z5V2xoMlRuSnJlRlV6Vkd0VlRVa3pNM2w2ZGtaV1ZsSlVOSGQ0VjBwRE9UazBUM05rWTFvMEsxSkhUbk5aUkhsU05XZHRaSEl3YmtSSFp6MGlMQ0pOU1VsRFVYcERRMEZqYldkQmQwbENRV2RKU1V4aldEaHBUa3hHVXpWVmQwTm5XVWxMYjFwSmVtb3dSVUYzVFhkYWVrVmlUVUpyUjBFeFZVVkJkM2RUVVZoQ2QySkhWV2RWYlRsMlpFTkNSRkZUUVhSSlJXTjZUVk5aZDBwQldVUldVVkZNUkVJeFFtTklRbk5hVTBKRVdsaEtNR0ZYV25CWk1rWXdZVmM1ZFVsRlJqRmtSMmgyWTIxc01HVlVSVlJOUWtWSFFURlZSVU5uZDB0UldFSjNZa2RWWjFOWE5XcE1ha1ZNVFVGclIwRXhWVVZDYUUxRFZsWk5kMGhvWTA1TlZGRjNUa1JOZDAxVVozaFBWRUV5VjJoalRrMTZhM2RPUkUxM1RWUm5lRTlVUVRKWGFrSnVUVkp6ZDBkUldVUldVVkZFUkVKS1FtTklRbk5hVTBKVFlqSTVNRWxGVGtKSlF6Qm5VbnBOZUVwcVFXdENaMDVXUWtGelRVaFZSbmRqUjNoc1NVVk9iR051VW5CYWJXeHFXVmhTY0dJeU5HZFJXRll3WVVjNWVXRllValZOVWsxM1JWRlpSRlpSVVV0RVFYQkNZMGhDYzFwVFFrcGliVTExVFZGemQwTlJXVVJXVVZGSFJYZEtWbFY2UWpKTlFrRkhRbmx4UjFOTk5EbEJaMFZIUWxOMVFrSkJRV2xCTWtsQlFrcHFjRXg2TVVGamNWUjBhM2xLZVdkU1RXTXpVa05XT0dOWGFsUnVTR05HUW1KYVJIVlhiVUpUY0ROYVNIUm1WR3BxVkhWNGVFVjBXQzh4U0RkWmVWbHNNMG8yV1ZKaVZIcENVRVZXYjBFdlZtaFpSRXRZTVVSNWVFNUNNR05VWkdSeFdHdzFaSFpOVm5wMFN6VXhOMGxFZGxsMVZsUmFXSEJ0YTA5c1JVdE5ZVTVEVFVWQmQwaFJXVVJXVWpCUFFrSlpSVVpNZFhjemNVWlpUVFJwWVhCSmNWb3pjalk1TmpZdllYbDVVM0pOUVRoSFFURlZaRVYzUlVJdmQxRkdUVUZOUWtGbU9IZEVaMWxFVmxJd1VFRlJTQzlDUVZGRVFXZEZSMDFCYjBkRFEzRkhVMDAwT1VKQlRVUkJNbWRCVFVkVlEwMVJRMFEyWTBoRlJtdzBZVmhVVVZreVpUTjJPVWQzVDBGRldreDFUaXQ1VW1oSVJrUXZNMjFsYjNsb2NHMTJUM2RuVUZWdVVGZFVlRzVUTkdGMEszRkplRlZEVFVjeGJXbG9SRXN4UVROVlZEZ3lUbEY2TmpCcGJVOXNUVEkzYW1Ka2IxaDBNbEZtZVVaTmJTdFphR2xrUkd0TVJqRjJURlZoWjAwMlFtZEVOVFpMZVV0QlBUMGlYWDAuZXlKMGNtRnVjMkZqZEdsdmJrbGtJam9pTWpBd01EQXdNRGc0TXpjd09EazRNQ0lzSW05eWFXZHBibUZzVkhKaGJuTmhZM1JwYjI1SlpDSTZJakl3TURBd01EQTRORGt4TmpReU1UTWlMQ0ozWldKUGNtUmxja3hwYm1WSmRHVnRTV1FpT2lJeU1EQXdNREF3TURreU16ZzFPVGM0SWl3aVluVnVaR3hsU1dRaU9pSmpiMjB1WVhKdFlXdHZiUzV4Y21sdVp5NXVaWFFpTENKd2NtOWtkV04wU1dRaU9pSnhjbWx1WjE5emFXeDJaWElpTENKemRXSnpZM0pwY0hScGIyNUhjbTkxY0Vsa1pXNTBhV1pwWlhJaU9pSXlNVFl5TnpBek1TSXNJbkIxY21Ob1lYTmxSR0YwWlNJNk1UYzBNams1TnpJeE1EQXdNQ3dpYjNKcFoybHVZV3hRZFhKamFHRnpaVVJoZEdVaU9qRTNNemc0TWpRMU1UY3dNREFzSW1WNGNHbHlaWE5FWVhSbElqb3hOelF6TURnek5qRXdNREF3TENKeGRXRnVkR2wwZVNJNk1Td2lkSGx3WlNJNklrRjFkRzh0VW1WdVpYZGhZbXhsSUZOMVluTmpjbWx3ZEdsdmJpSXNJbWx1UVhCd1QzZHVaWEp6YUdsd1ZIbHdaU0k2SWxCVlVrTklRVk5GUkNJc0luTnBaMjVsWkVSaGRHVWlPakUzTkRNd09EQTVNamN3TlRRc0ltVnVkbWx5YjI1dFpXNTBJam9pVTJGdVpHSnZlQ0lzSW5SeVlXNXpZV04wYVc5dVVtVmhjMjl1SWpvaVVGVlNRMGhCVTBVaUxDSnpkRzl5WldaeWIyNTBJam9pVkZWU0lpd2ljM1J2Y21WbWNtOXVkRWxrSWpvaU1UUXpORGd3SWl3aWNISnBZMlVpT2pFNU9UazVNQ3dpWTNWeWNtVnVZM2tpT2lKVVVsa2lMQ0poY0hCVWNtRnVjMkZqZEdsdmJrbGtJam9pTnpBME1qYzJPRGcyTkRFNE5qWTBNems1SW4wLkVnY3pvbVUxNzNLa29pZ2RUS3dXQk5UNE5SNDJmcEVkaHJUb1g2TVZ1V2o5OGJVVWhhUXI2S25EREdOOXpCLXN3c2lwZklLLXFubXJndjQ1WXBlZW5BIiwic2lnbmVkUmVuZXdhbEluZm8iOiJleUpoYkdjaU9pSkZVekkxTmlJc0luZzFZeUk2V3lKTlNVbEZUVVJEUTBFM1lXZEJkMGxDUVdkSlVXWlViR1prTUdaT2RrWlhkbnBETVZsSlFVNXpXR3BCUzBKblozRm9hMnBQVUZGUlJFRjZRakZOVlZGM1VXZFpSRlpSVVVSRVJIUkNZMGhDYzFwVFFsaGlNMHB6V2toa2NGcEhWV2RTUjFZeVdsZDRkbU5IVm5sSlJrcHNZa2RHTUdGWE9YVmplVUpFV2xoS01HRlhXbkJaTWtZd1lWYzVkVWxGUmpGa1IyaDJZMjFzTUdWVVJVeE5RV3RIUVRGVlJVTjNkME5TZWxsNFJYcEJVa0puVGxaQ1FXOU5RMnRHZDJOSGVHeEpSV3gxV1hrMGVFTjZRVXBDWjA1V1FrRlpWRUZzVmxSTlFqUllSRlJKZWsxRWEzaE5ha1UxVGxSRk1VMHhiMWhFVkVreFRWUkJlRTFVUlRWT1ZFVXhUV3h2ZDJkYVNYaFJSRUVyUW1kT1ZrSkJUVTFPTVVKNVlqSlJaMUpWVGtSSlJURm9XWGxDUW1OSVFXZFZNMUoyWTIxVloxbFhOV3RKUjJ4VlpGYzFiR041UWxSa1J6bDVXbE5DVTFwWFRteGhXRUl3U1VaT2NGb3lOWEJpYldONFRFUkJjVUpuVGxaQ1FYTk5TVEJHZDJOSGVHeEpSbVIyWTIxNGEyUXliR3RhVTBKRldsaGFiR0pIT1hkYVdFbG5WVzFXYzFsWVVuQmlNalY2VFZKTmQwVlJXVVJXVVZGTFJFRndRbU5JUW5OYVUwSktZbTFOZFUxUmMzZERVVmxFVmxGUlIwVjNTbFpWZWtKYVRVSk5SMEo1Y1VkVFRUUTVRV2RGUjBORGNVZFRUVFE1UVhkRlNFRXdTVUZDUlVaRldXVXZTbkZVY1hsUmRpOWtkRmhyWVhWRVNFTlRZMVl4TWpsR1dWSldMekI0YVVJeU5HNURVV3Q2VVdZellYTklTazlPVWpWeU1GSkJNR0ZNZGtvME16Sm9lVEZUV2sxdmRYWjVabkJ0TWpacVdGTnFaMmRKU1UxSlNVTkNSRUZOUW1kT1ZraFNUVUpCWmpoRlFXcEJRVTFDT0VkQk1WVmtTWGRSV1UxQ1lVRkdSRGgyYkVOT1VqQXhSRXB0YVdjNU4ySkNPRFZqSzJ4clIwdGFUVWhCUjBORGMwZEJVVlZHUW5kRlFrSkhVWGRaYWtGMFFtZG5ja0puUlVaQ1VXTjNRVzlaYUdGSVVqQmpSRzkyVERKT2JHTnVVbnBNYlVaM1kwZDRiRXh0VG5aaVV6a3paREpTZVZwNldYVmFSMVo1VFVSRlIwTkRjMGRCVVZWR1FucEJRbWhwVm05a1NGSjNUMms0ZG1JeVRucGpRelZvWTBoQ2MxcFROV3BpTWpCMllqSk9lbU5FUVhwTVdHUXpXa2hLYms1cVFYbE5TVWxDU0dkWlJGWlNNR2RDU1VsQ1JsUkRRMEZTUlhkblowVk9RbWR2Y1docmFVYzVNazVyUWxGWlFrMUpTQ3ROU1VoRVFtZG5ja0puUlVaQ1VXTkRRV3BEUW5SbmVVSnpNVXBzWWtkc2FHSnRUbXhKUnpsMVNVaFNiMkZZVFdkWk1sWjVaRWRzYldGWFRtaGtSMVZuV1c1cloxbFhOVFZKU0VKb1kyNVNOVWxIUm5wak0xWjBXbGhOWjFsWFRtcGFXRUl3V1ZjMWFscFRRblphYVVJd1lVZFZaMlJIYUd4aWFVSm9ZMGhDYzJGWFRtaFpiWGhzU1VoT01GbFhOV3RaV0VwclNVaFNiR050TVhwSlIwWjFXa05DYW1JeU5XdGhXRkp3WWpJMWVrbEhPVzFKU0ZaNldsTjNaMWt5Vm5sa1IyeHRZVmRPYUdSSFZXZGpSemx6WVZkT05VbEhSblZhUTBKcVdsaEtNR0ZYV25CWk1rWXdZVmM1ZFVsSVFubFpWMDR3WVZkT2JFbElUakJaV0ZKc1lsZFdkV1JJVFhWTlJGbEhRME56UjBGUlZVWkNkMGxDUm1sd2IyUklVbmRQYVRoMlpETmtNMHh0Um5kalIzaHNURzFPZG1KVE9XcGFXRW93WVZkYWNGa3lSakJhVjBZeFpFZG9kbU50YkRCbFV6aDNTRkZaUkZaU01FOUNRbGxGUmtGTmN6aFFhbk0yVm1oWFIxRnNla1V5V2s5RkswZFlORTl2TDAxQk5FZEJNVlZrUkhkRlFpOTNVVVZCZDBsSVowUkJVVUpuYjNGb2EybEhPVEpPYTBKbmMwSkNRVWxHUVVSQlMwSm5aM0ZvYTJwUFVGRlJSRUYzVG05QlJFSnNRV3BGUVRoNVVrNWtjMnR3TlRBMlJFWmtVRXhuYUV4TVNuZEJkalZLT0doQ1IweGhTVGhFUlhoa1kxQllLMkZDUzJwcVR6aGxWVzg1UzNCbWNHTk9XVlZaTlZsQmFrRlFXRzFOV0VWYVRDdFJNREpoWkhKdGJYTm9Ubmg2TTA1dVMyMHJiM1ZSZDFVM2RrSlViakJNZG14Tk4zWndjekpaYzJ4V1ZHRnRVbGxNTkdGVGN6VnJQU0lzSWsxSlNVUkdha05EUVhCNVowRjNTVUpCWjBsVlNYTkhhRkozY0RCak1tNTJWVFJaVTNsallXWlFWR3A2WWs1amQwTm5XVWxMYjFwSmVtb3dSVUYzVFhkYWVrVmlUVUpyUjBFeFZVVkJkM2RUVVZoQ2QySkhWV2RWYlRsMlpFTkNSRkZUUVhSSlJXTjZUVk5aZDBwQldVUldVVkZNUkVJeFFtTklRbk5hVTBKRVdsaEtNR0ZYV25CWk1rWXdZVmM1ZFVsRlJqRmtSMmgyWTIxc01HVlVSVlJOUWtWSFFURlZSVU5uZDB0UldFSjNZa2RWWjFOWE5XcE1ha1ZNVFVGclIwRXhWVVZDYUUxRFZsWk5kMGhvWTA1TmFrVjNUWHBGTTAxcVFYcE9la1YzVjJoalRrMTZXWGROZWtVMVRVUkJkMDFFUVhkWGFrSXhUVlZSZDFGbldVUldVVkZFUkVSMFFtTklRbk5hVTBKWVlqTktjMXBJWkhCYVIxVm5Va2RXTWxwWGVIWmpSMVo1U1VaS2JHSkhSakJoVnpsMVkzbENSRnBZU2pCaFYxcHdXVEpHTUdGWE9YVkpSVVl4WkVkb2RtTnRiREJsVkVWTVRVRnJSMEV4VlVWRGQzZERVbnBaZUVWNlFWSkNaMDVXUWtGdlRVTnJSbmRqUjNoc1NVVnNkVmw1TkhoRGVrRktRbWRPVmtKQldWUkJiRlpVVFVoWmQwVkJXVWhMYjFwSmVtb3dRMEZSV1VaTE5FVkZRVU5KUkZsblFVVmljMUZMUXprMFVISnNWMjFhV0c1WVozUjRlbVJXU2t3NFZEQlRSMWx1WjBSU1IzQnVaMjR6VGpaUVZEaEtUVVZpTjBaRWFUUmlRbTFRYUVOdVdqTXZjM0UyVUVZdlkwZGpTMWhYYzB3MWRrOTBaVkpvZVVvME5YZ3pRVk5RTjJOUFFpdGhZVzg1TUdaamNIaFRkaTlGV2taaWJtbEJZazVuV2tkb1NXaHdTVzgwU0RaTlNVZ3pUVUpKUjBFeFZXUkZkMFZDTDNkUlNVMUJXVUpCWmpoRFFWRkJkMGgzV1VSV1VqQnFRa0puZDBadlFWVjFOMFJsYjFabmVtbEtjV3RwY0c1bGRuSXpjbkk1Y2t4S1MzTjNVbWRaU1V0M1dVSkNVVlZJUVZGRlJVOXFRVFJOUkZsSFEwTnpSMEZSVlVaQ2VrRkNhR2x3YjJSSVVuZFBhVGgyWWpKT2VtTkROV2hqU0VKeldsTTFhbUl5TUhaaU1rNTZZMFJCZWt4WFJuZGpSM2hzWTIwNWRtUkhUbWhhZWsxM1RuZFpSRlpTTUdaQ1JFRjNUR3BCYzI5RGNXZExTVmx0WVVoU01HTkViM1pNTWs1NVlrTTFhR05JUW5OYVV6VnFZakl3ZGxsWVFuZGlSMVo1WWpJNU1Ga3lSbTVOZVRWcVkyMTNkMGhSV1VSV1VqQlBRa0paUlVaRU9IWnNRMDVTTURGRVNtMXBaemszWWtJNE5XTXJiR3RIUzFwTlFUUkhRVEZWWkVSM1JVSXZkMUZGUVhkSlFrSnFRVkZDWjI5eGFHdHBSemt5VG10Q1owbENRa0ZKUmtGRVFVdENaMmR4YUd0cVQxQlJVVVJCZDA1dlFVUkNiRUZxUWtGWWFGTnhOVWw1UzI5blRVTlFkSGMwT1RCQ1lVSTJOemREWVVWSFNsaDFabEZDTDBWeFdrZGtOa05UYW1sRGRFOXVkVTFVWWxoV1dHMTRlR040Wm10RFRWRkVWRk5RZUdGeVdsaDJUbkpyZUZVelZHdFZUVWt6TTNsNmRrWldWbEpVTkhkNFYwcERPVGswVDNOa1kxbzBLMUpIVG5OWlJIbFNOV2R0WkhJd2JrUkhaejBpTENKTlNVbERVWHBEUTBGamJXZEJkMGxDUVdkSlNVeGpXRGhwVGt4R1V6VlZkME5uV1VsTGIxcEplbW93UlVGM1RYZGFla1ZpVFVKclIwRXhWVVZCZDNkVFVWaENkMkpIVldkVmJUbDJaRU5DUkZGVFFYUkpSV042VFZOWmQwcEJXVVJXVVZGTVJFSXhRbU5JUW5OYVUwSkVXbGhLTUdGWFduQlpNa1l3WVZjNWRVbEZSakZrUjJoMlkyMXNNR1ZVUlZSTlFrVkhRVEZWUlVObmQwdFJXRUozWWtkVloxTlhOV3BNYWtWTVRVRnJSMEV4VlVWQ2FFMURWbFpOZDBob1kwNU5WRkYzVGtSTmQwMVVaM2hQVkVFeVYyaGpUazE2YTNkT1JFMTNUVlJuZUU5VVFUSlhha0p1VFZKemQwZFJXVVJXVVZGRVJFSktRbU5JUW5OYVUwSlRZakk1TUVsRlRrSkpRekJuVW5wTmVFcHFRV3RDWjA1V1FrRnpUVWhWUm5kalIzaHNTVVZPYkdOdVVuQmFiV3hxV1ZoU2NHSXlOR2RSV0ZZd1lVYzVlV0ZZVWpWTlVrMTNSVkZaUkZaUlVVdEVRWEJDWTBoQ2MxcFRRa3BpYlUxMVRWRnpkME5SV1VSV1VWRkhSWGRLVmxWNlFqSk5Ra0ZIUW5seFIxTk5ORGxCWjBWSFFsTjFRa0pCUVdsQk1rbEJRa3BxY0V4Nk1VRmpjVlIwYTNsS2VXZFNUV016VWtOV09HTlhhbFJ1U0dOR1FtSmFSSFZYYlVKVGNETmFTSFJtVkdwcVZIVjRlRVYwV0M4eFNEZFplVmxzTTBvMldWSmlWSHBDVUVWV2IwRXZWbWhaUkV0WU1VUjVlRTVDTUdOVVpHUnhXR3cxWkhaTlZucDBTelV4TjBsRWRsbDFWbFJhV0hCdGEwOXNSVXROWVU1RFRVVkJkMGhSV1VSV1VqQlBRa0paUlVaTWRYY3pjVVpaVFRScFlYQkpjVm96Y2pZNU5qWXZZWGw1VTNKTlFUaEhRVEZWWkVWM1JVSXZkMUZHVFVGTlFrRm1PSGRFWjFsRVZsSXdVRUZSU0M5Q1FWRkVRV2RGUjAxQmIwZERRM0ZIVTAwME9VSkJUVVJCTW1kQlRVZFZRMDFSUTBRMlkwaEZSbXcwWVZoVVVWa3laVE4yT1VkM1QwRkZXa3gxVGl0NVVtaElSa1F2TTIxbGIzbG9jRzEyVDNkblVGVnVVRmRVZUc1VE5HRjBLM0ZKZUZWRFRVY3hiV2xvUkVzeFFUTlZWRGd5VGxGNk5qQnBiVTlzVFRJM2FtSmtiMWgwTWxGbWVVWk5iU3RaYUdsa1JHdE1SakYyVEZWaFowMDJRbWRFTlRaTGVVdEJQVDBpWFgwLmV5SnZjbWxuYVc1aGJGUnlZVzV6WVdOMGFXOXVTV1FpT2lJeU1EQXdNREF3T0RRNU1UWTBNakV6SWl3aVlYVjBiMUpsYm1WM1VISnZaSFZqZEVsa0lqb2ljWEpwYm1kZloyOXNaQ0lzSW5CeWIyUjFZM1JKWkNJNkluRnlhVzVuWDNOcGJIWmxjaUlzSW1GMWRHOVNaVzVsZDFOMFlYUjFjeUk2TVN3aWNtVnVaWGRoYkZCeWFXTmxJam95TkRrNU9UQXNJbU4xY25KbGJtTjVJam9pVkZKWklpd2ljMmxuYm1Wa1JHRjBaU0k2TVRjME16QTRNRGt5TnpBMU5Dd2laVzUyYVhKdmJtMWxiblFpT2lKVFlXNWtZbTk0SWl3aWNtVmpaVzUwVTNWaWMyTnlhWEIwYVc5dVUzUmhjblJFWVhSbElqb3hOelF5T1RrM01qRXdNREF3TENKeVpXNWxkMkZzUkdGMFpTSTZNVGMwTXpBNE16WXhNREF3TUN3aVlYQndWSEpoYm5OaFkzUnBiMjVKWkNJNklqY3dOREkzTmpnNE5qUXhPRFkyTkRNNU9TSjkuNmk1V0tsZENZTVRHRm9QdVNiazdEN2hncmJBdXdibkdNYkFOeWp0X2Iza1BoVlEzaUpTVjZZUm1hdkV4NHNCNkd3Q0Z1dWJPeDlDYnExdy1leXVkX1EiLCJzdGF0dXMiOjF9LCJ2ZXJzaW9uIjoiMi4wIiwic2lnbmVkRGF0ZSI6MTc0MzA4MDkyNzA4Mn0.monD8oboahcdivVS7ySaNkF40fqNwAQwB1UT-jYA6o2zT3nCeEZ_bCVHFx1eAf6cw1qTNxHRECd4OSXUUgVqgg"
}
```

Burdaki token decode edildiğinde:

```json
{
  "notificationType": "DID_CHANGE_RENEWAL_PREF",
  "subtype": "DOWNGRADE",
  "notificationUUID": "f19852eb-a8dd-4023-b49c-949cadbee84c",
  "data": {
    "appAppleId": 6738924421,
    "bundleId": "com.armakom.qring.net",
    "bundleVersion": "6",
    "environment": "Sandbox",
    "signedTransactionInfo": "eyJhbGciOiJFUzI1NiIsIng1YyI6WyJNSUlFTURDQ0E3YWdBd0lCQWdJUWZUbGZkMGZOdkZXdnpDMVlJQU5zWGpBS0JnZ3Foa2pPUFFRREF6QjFNVVF3UWdZRFZRUURERHRCY0hCc1pTQlhiM0pzWkhkcFpHVWdSR1YyWld4dmNHVnlJRkpsYkdGMGFXOXVjeUJEWlhKMGFXWnBZMkYwYVc5dUlFRjFkR2h2Y21sMGVURUxNQWtHQTFVRUN3d0NSell4RXpBUkJnTlZCQW9NQ2tGd2NHeGxJRWx1WXk0eEN6QUpCZ05WQkFZVEFsVlRNQjRYRFRJek1Ea3hNakU1TlRFMU0xb1hEVEkxTVRBeE1URTVOVEUxTWxvd2daSXhRREErQmdOVkJBTU1OMUJ5YjJRZ1JVTkRJRTFoWXlCQmNIQWdVM1J2Y21VZ1lXNWtJR2xVZFc1bGN5QlRkRzl5WlNCU1pXTmxhWEIwSUZOcFoyNXBibWN4TERBcUJnTlZCQXNNSTBGd2NHeGxJRmR2Y214a2QybGtaU0JFWlhabGJHOXdaWElnVW1Wc1lYUnBiMjV6TVJNd0VRWURWUVFLREFwQmNIQnNaU0JKYm1NdU1Rc3dDUVlEVlFRR0V3SlZVekJaTUJNR0J5cUdTTTQ5QWdFR0NDcUdTTTQ5QXdFSEEwSUFCRUZFWWUvSnFUcXlRdi9kdFhrYXVESENTY1YxMjlGWVJWLzB4aUIyNG5DUWt6UWYzYXNISk9OUjVyMFJBMGFMdko0MzJoeTFTWk1vdXZ5ZnBtMjZqWFNqZ2dJSU1JSUNCREFNQmdOVkhSTUJBZjhFQWpBQU1COEdBMVVkSXdRWU1CYUFGRDh2bENOUjAxREptaWc5N2JCODVjK2xrR0taTUhBR0NDc0dBUVVGQndFQkJHUXdZakF0QmdnckJnRUZCUWN3QW9ZaGFIUjBjRG92TDJObGNuUnpMbUZ3Y0d4bExtTnZiUzkzZDJSeVp6WXVaR1Z5TURFR0NDc0dBUVVGQnpBQmhpVm9kSFJ3T2k4dmIyTnpjQzVoY0hCc1pTNWpiMjB2YjJOemNEQXpMWGQzWkhKbk5qQXlNSUlCSGdZRFZSMGdCSUlCRlRDQ0FSRXdnZ0VOQmdvcWhraUc5Mk5rQlFZQk1JSCtNSUhEQmdnckJnRUZCUWNDQWpDQnRneUJzMUpsYkdsaGJtTmxJRzl1SUhSb2FYTWdZMlZ5ZEdsbWFXTmhkR1VnWW5rZ1lXNTVJSEJoY25SNUlHRnpjM1Z0WlhNZ1lXTmpaWEIwWVc1alpTQnZaaUIwYUdVZ2RHaGxiaUJoY0hCc2FXTmhZbXhsSUhOMFlXNWtZWEprSUhSbGNtMXpJR0Z1WkNCamIyNWthWFJwYjI1eklHOW1JSFZ6WlN3Z1kyVnlkR2xtYVdOaGRHVWdjRzlzYVdONUlHRnVaQ0JqWlhKMGFXWnBZMkYwYVc5dUlIQnlZV04wYVdObElITjBZWFJsYldWdWRITXVNRFlHQ0NzR0FRVUZCd0lCRmlwb2RIUndPaTh2ZDNkM0xtRndjR3hsTG1OdmJTOWpaWEowYVdacFkyRjBaV0YxZEdodmNtbDBlUzh3SFFZRFZSME9CQllFRkFNczhQanM2VmhXR1FsekUyWk9FK0dYNE9vL01BNEdBMVVkRHdFQi93UUVBd0lIZ0RBUUJnb3Foa2lHOTJOa0Jnc0JCQUlGQURBS0JnZ3Foa2pPUFFRREF3Tm9BREJsQWpFQTh5Uk5kc2twNTA2REZkUExnaExMSndBdjVKOGhCR0xhSThERXhkY1BYK2FCS2pqTzhlVW85S3BmcGNOWVVZNVlBakFQWG1NWEVaTCtRMDJhZHJtbXNoTnh6M05uS20rb3VRd1U3dkJUbjBMdmxNN3ZwczJZc2xWVGFtUllMNGFTczVrPSIsIk1JSURGakNDQXB5Z0F3SUJBZ0lVSXNHaFJ3cDBjMm52VTRZU3ljYWZQVGp6Yk5jd0NnWUlLb1pJemowRUF3TXdaekViTUJrR0ExVUVBd3dTUVhCd2JHVWdVbTl2ZENCRFFTQXRJRWN6TVNZd0pBWURWUVFMREIxQmNIQnNaU0JEWlhKMGFXWnBZMkYwYVc5dUlFRjFkR2h2Y21sMGVURVRNQkVHQTFVRUNnd0tRWEJ3YkdVZ1NXNWpMakVMTUFrR0ExVUVCaE1DVlZNd0hoY05NakV3TXpFM01qQXpOekV3V2hjTk16WXdNekU1TURBd01EQXdXakIxTVVRd1FnWURWUVFERER0QmNIQnNaU0JYYjNKc1pIZHBaR1VnUkdWMlpXeHZjR1Z5SUZKbGJHRjBhVzl1Y3lCRFpYSjBhV1pwWTJGMGFXOXVJRUYxZEdodmNtbDBlVEVMTUFrR0ExVUVDd3dDUnpZeEV6QVJCZ05WQkFvTUNrRndjR3hsSUVsdVl5NHhDekFKQmdOVkJBWVRBbFZUTUhZd0VBWUhLb1pJemowQ0FRWUZLNEVFQUNJRFlnQUVic1FLQzk0UHJsV21aWG5YZ3R4emRWSkw4VDBTR1luZ0RSR3BuZ24zTjZQVDhKTUViN0ZEaTRiQm1QaENuWjMvc3E2UEYvY0djS1hXc0w1dk90ZVJoeUo0NXgzQVNQN2NPQithYW85MGZjcHhTdi9FWkZibmlBYk5nWkdoSWhwSW80SDZNSUgzTUJJR0ExVWRFd0VCL3dRSU1BWUJBZjhDQVFBd0h3WURWUjBqQkJnd0ZvQVV1N0Rlb1ZnemlKcWtpcG5ldnIzcnI5ckxKS3N3UmdZSUt3WUJCUVVIQVFFRU9qQTRNRFlHQ0NzR0FRVUZCekFCaGlwb2RIUndPaTh2YjJOemNDNWhjSEJzWlM1amIyMHZiMk56Y0RBekxXRndjR3hsY205dmRHTmhaek13TndZRFZSMGZCREF3TGpBc29DcWdLSVltYUhSMGNEb3ZMMk55YkM1aGNIQnNaUzVqYjIwdllYQndiR1Z5YjI5MFkyRm5NeTVqY213d0hRWURWUjBPQkJZRUZEOHZsQ05SMDFESm1pZzk3YkI4NWMrbGtHS1pNQTRHQTFVZER3RUIvd1FFQXdJQkJqQVFCZ29xaGtpRzkyTmtCZ0lCQkFJRkFEQUtCZ2dxaGtqT1BRUURBd05vQURCbEFqQkFYaFNxNUl5S29nTUNQdHc0OTBCYUI2NzdDYUVHSlh1ZlFCL0VxWkdkNkNTamlDdE9udU1UYlhWWG14eGN4ZmtDTVFEVFNQeGFyWlh2TnJreFUzVGtVTUkzM3l6dkZWVlJUNHd4V0pDOTk0T3NkY1o0K1JHTnNZRHlSNWdtZHIwbkRHZz0iLCJNSUlDUXpDQ0FjbWdBd0lCQWdJSUxjWDhpTkxGUzVVd0NnWUlLb1pJemowRUF3TXdaekViTUJrR0ExVUVBd3dTUVhCd2JHVWdVbTl2ZENCRFFTQXRJRWN6TVNZd0pBWURWUVFMREIxQmNIQnNaU0JEWlhKMGFXWnBZMkYwYVc5dUlFRjFkR2h2Y21sMGVURVRNQkVHQTFVRUNnd0tRWEJ3YkdVZ1NXNWpMakVMTUFrR0ExVUVCaE1DVlZNd0hoY05NVFF3TkRNd01UZ3hPVEEyV2hjTk16a3dORE13TVRneE9UQTJXakJuTVJzd0dRWURWUVFEREJKQmNIQnNaU0JTYjI5MElFTkJJQzBnUnpNeEpqQWtCZ05WQkFzTUhVRndjR3hsSUVObGNuUnBabWxqWVhScGIyNGdRWFYwYUc5eWFYUjVNUk13RVFZRFZRUUtEQXBCY0hCc1pTQkpibU11TVFzd0NRWURWUVFHRXdKVlV6QjJNQkFHQnlxR1NNNDlBZ0VHQlN1QkJBQWlBMklBQkpqcEx6MUFjcVR0a3lKeWdSTWMzUkNWOGNXalRuSGNGQmJaRHVXbUJTcDNaSHRmVGpqVHV4eEV0WC8xSDdZeVlsM0o2WVJiVHpCUEVWb0EvVmhZREtYMUR5eE5CMGNUZGRxWGw1ZHZNVnp0SzUxN0lEdll1VlRaWHBta09sRUtNYU5DTUVBd0hRWURWUjBPQkJZRUZMdXczcUZZTTRpYXBJcVozcjY5NjYvYXl5U3JNQThHQTFVZEV3RUIvd1FGTUFNQkFmOHdEZ1lEVlIwUEFRSC9CQVFEQWdFR01Bb0dDQ3FHU000OUJBTURBMmdBTUdVQ01RQ0Q2Y0hFRmw0YVhUUVkyZTN2OUd3T0FFWkx1Tit5UmhIRkQvM21lb3locG12T3dnUFVuUFdUeG5TNGF0K3FJeFVDTUcxbWloREsxQTNVVDgyTlF6NjBpbU9sTTI3amJkb1h0MlFmeUZNbStZaGlkRGtMRjF2TFVhZ002QmdENTZLeUtBPT0iXX0.eyJ0cmFuc2FjdGlvbklkIjoiMjAwMDAwMDg4MzcwODk4MCIsIm9yaWdpbmFsVHJhbnNhY3Rpb25JZCI6IjIwMDAwMDA4NDkxNjQyMTMiLCJ3ZWJPcmRlckxpbmVJdGVtSWQiOiIyMDAwMDAwMDkyMzg1OTc4IiwiYnVuZGxlSWQiOiJjb20uYXJtYWtvbS5xcmluZy5uZXQiLCJwcm9kdWN0SWQiOiJxcmluZ19zaWx2ZXIiLCJzdWJzY3JpcHRpb25Hcm91cElkZW50aWZpZXIiOiIyMTYyNzAzMSIsInB1cmNoYXNlRGF0ZSI6MTc0Mjk5NzIxMDAwMCwib3JpZ2luYWxQdXJjaGFzZURhdGUiOjE3Mzg4MjQ1MTcwMDAsImV4cGlyZXNEYXRlIjoxNzQzMDgzNjEwMDAwLCJxdWFudGl0eSI6MSwidHlwZSI6IkF1dG8tUmVuZXdhYmxlIFN1YnNjcmlwdGlvbiIsImluQXBwT3duZXJzaGlwVHlwZSI6IlBVUkNIQVNFRCIsInNpZ25lZERhdGUiOjE3NDMwODA5MjcwNTQsImVudmlyb25tZW50IjoiU2FuZGJveCIsInRyYW5zYWN0aW9uUmVhc29uIjoiUFVSQ0hBU0UiLCJzdG9yZWZyb250IjoiVFVSIiwic3RvcmVmcm9udElkIjoiMTQzNDgwIiwicHJpY2UiOjE5OTk5MCwiY3VycmVuY3kiOiJUUlkiLCJhcHBUcmFuc2FjdGlvbklkIjoiNzA0Mjc2ODg2NDE4NjY0Mzk5In0.EgczomU173KkoigdTKwWBNT4NR42fpEdhrToX6MVuWj98bUUhaQr6KnDDGN9zB-swsipfIK-qnmrgv45YpeenA",
    "signedRenewalInfo": "eyJhbGciOiJFUzI1NiIsIng1YyI6WyJNSUlFTURDQ0E3YWdBd0lCQWdJUWZUbGZkMGZOdkZXdnpDMVlJQU5zWGpBS0JnZ3Foa2pPUFFRREF6QjFNVVF3UWdZRFZRUURERHRCY0hCc1pTQlhiM0pzWkhkcFpHVWdSR1YyWld4dmNHVnlJRkpsYkdGMGFXOXVjeUJEWlhKMGFXWnBZMkYwYVc5dUlFRjFkR2h2Y21sMGVURUxNQWtHQTFVRUN3d0NSell4RXpBUkJnTlZCQW9NQ2tGd2NHeGxJRWx1WXk0eEN6QUpCZ05WQkFZVEFsVlRNQjRYRFRJek1Ea3hNakU1TlRFMU0xb1hEVEkxTVRBeE1URTVOVEUxTWxvd2daSXhRREErQmdOVkJBTU1OMUJ5YjJRZ1JVTkRJRTFoWXlCQmNIQWdVM1J2Y21VZ1lXNWtJR2xVZFc1bGN5QlRkRzl5WlNCU1pXTmxhWEIwSUZOcFoyNXBibWN4TERBcUJnTlZCQXNNSTBGd2NHeGxJRmR2Y214a2QybGtaU0JFWlhabGJHOXdaWElnVW1Wc1lYUnBiMjV6TVJNd0VRWURWUVFLREFwQmNIQnNaU0JKYm1NdU1Rc3dDUVlEVlFRR0V3SlZVekJaTUJNR0J5cUdTTTQ5QWdFR0NDcUdTTTQ5QXdFSEEwSUFCRUZFWWUvSnFUcXlRdi9kdFhrYXVESENTY1YxMjlGWVJWLzB4aUIyNG5DUWt6UWYzYXNISk9OUjVyMFJBMGFMdko0MzJoeTFTWk1vdXZ5ZnBtMjZqWFNqZ2dJSU1JSUNCREFNQmdOVkhSTUJBZjhFQWpBQU1COEdBMVVkSXdRWU1CYUFGRDh2bENOUjAxREptaWc5N2JCODVjK2xrR0taTUhBR0NDc0dBUVVGQndFQkJHUXdZakF0QmdnckJnRUZCUWN3QW9ZaGFIUjBjRG92TDJObGNuUnpMbUZ3Y0d4bExtTnZiUzkzZDJSeVp6WXVaR1Z5TURFR0NDc0dBUVVGQnpBQmhpVm9kSFJ3T2k4dmIyTnpjQzVoY0hCc1pTNWpiMjB2YjJOemNEQXpMWGQzWkhKbk5qQXlNSUlCSGdZRFZSMGdCSUlCRlRDQ0FSRXdnZ0VOQmdvcWhraUc5Mk5rQlFZQk1JSCtNSUhEQmdnckJnRUZCUWNDQWpDQnRneUJzMUpsYkdsaGJtTmxJRzl1SUhSb2FYTWdZMlZ5ZEdsbWFXTmhkR1VnWW5rZ1lXNTVJSEJoY25SNUlHRnpjM1Z0WlhNZ1lXTmpaWEIwWVc1alpTQnZaaUIwYUdVZ2RHaGxiaUJoY0hCc2FXTmhZbXhsSUhOMFlXNWtZWEprSUhSbGNtMXpJR0Z1WkNCamIyNWthWFJwYjI1eklHOW1JSFZ6WlN3Z1kyVnlkR2xtYVdOaGRHVWdjRzlzYVdONUlHRnVaQ0JqWlhKMGFXWnBZMkYwYVc5dUlIQnlZV04wYVdObElITjBZWFJsYldWdWRITXVNRFlHQ0NzR0FRVUZCd0lCRmlwb2RIUndPaTh2ZDNkM0xtRndjR3hsTG1OdmJTOWpaWEowYVdacFkyRjBaV0YxZEdodmNtbDBlUzh3SFFZRFZSME9CQllFRkFNczhQanM2VmhXR1FsekUyWk9FK0dYNE9vL01BNEdBMVVkRHdFQi93UUVBd0lIZ0RBUUJnb3Foa2lHOTJOa0Jnc0JCQUlGQURBS0JnZ3Foa2pPUFFRREF3Tm9BREJsQWpFQTh5Uk5kc2twNTA2REZkUExnaExMSndBdjVKOGhCR0xhSThERXhkY1BYK2FCS2pqTzhlVW85S3BmcGNOWVVZNVlBakFQWG1NWEVaTCtRMDJhZHJtbXNoTnh6M05uS20rb3VRd1U3dkJUbjBMdmxNN3ZwczJZc2xWVGFtUllMNGFTczVrPSIsIk1JSURGakNDQXB5Z0F3SUJBZ0lVSXNHaFJ3cDBjMm52VTRZU3ljYWZQVGp6Yk5jd0NnWUlLb1pJemowRUF3TXdaekViTUJrR0ExVUVBd3dTUVhCd2JHVWdVbTl2ZENCRFFTQXRJRWN6TVNZd0pBWURWUVFMREIxQmNIQnNaU0JEWlhKMGFXWnBZMkYwYVc5dUlFRjFkR2h2Y21sMGVURVRNQkVHQTFVRUNnd0tRWEJ3YkdVZ1NXNWpMakVMTUFrR0ExVUVCaE1DVlZNd0hoY05NakV3TXpFM01qQXpOekV3V2hjTk16WXdNekU1TURBd01EQXdXakIxTVVRd1FnWURWUVFERER0QmNIQnNaU0JYYjNKc1pIZHBaR1VnUkdWMlpXeHZjR1Z5SUZKbGJHRjBhVzl1Y3lCRFpYSjBhV1pwWTJGMGFXOXVJRUYxZEdodmNtbDBlVEVMTUFrR0ExVUVDd3dDUnpZeEV6QVJCZ05WQkFvTUNrRndjR3hsSUVsdVl5NHhDekFKQmdOVkJBWVRBbFZUTUhZd0VBWUhLb1pJemowQ0FRWUZLNEVFQUNJRFlnQUVic1FLQzk0UHJsV21aWG5YZ3R4emRWSkw4VDBTR1luZ0RSR3BuZ24zTjZQVDhKTUViN0ZEaTRiQm1QaENuWjMvc3E2UEYvY0djS1hXc0w1dk90ZVJoeUo0NXgzQVNQN2NPQithYW85MGZjcHhTdi9FWkZibmlBYk5nWkdoSWhwSW80SDZNSUgzTUJJR0ExVWRFd0VCL3dRSU1BWUJBZjhDQVFBd0h3WURWUjBqQkJnd0ZvQVV1N0Rlb1ZnemlKcWtpcG5ldnIzcnI5ckxKS3N3UmdZSUt3WUJCUVVIQVFFRU9qQTRNRFlHQ0NzR0FRVUZCekFCaGlwb2RIUndPaTh2YjJOemNDNWhjSEJzWlM1amIyMHZiMk56Y0RBekxXRndjR3hsY205dmRHTmhaek13TndZRFZSMGZCREF3TGpBc29DcWdLSVltYUhSMGNEb3ZMMk55YkM1aGNIQnNaUzVqYjIwdllYQndiR1Z5YjI5MFkyRm5NeTVqY213d0hRWURWUjBPQkJZRUZEOHZsQ05SMDFESm1pZzk3YkI4NWMrbGtHS1pNQTRHQTFVZER3RUIvd1FFQXdJQkJqQVFCZ29xaGtpRzkyTmtCZ0lCQkFJRkFEQUtCZ2dxaGtqT1BRUURBd05vQURCbEFqQkFYaFNxNUl5S29nTUNQdHc0OTBCYUI2NzdDYUVHSlh1ZlFCL0VxWkdkNkNTamlDdE9udU1UYlhWWG14eGN4ZmtDTVFEVFNQeGFyWlh2TnJreFUzVGtVTUkzM3l6dkZWVlJUNHd4V0pDOTk0T3NkY1o0K1JHTnNZRHlSNWdtZHIwbkRHZz0iLCJNSUlDUXpDQ0FjbWdBd0lCQWdJSUxjWDhpTkxGUzVVd0NnWUlLb1pJemowRUF3TXdaekViTUJrR0ExVUVBd3dTUVhCd2JHVWdVbTl2ZENCRFFTQXRJRWN6TVNZd0pBWURWUVFMREIxQmNIQnNaU0JEWlhKMGFXWnBZMkYwYVc5dUlFRjFkR2h2Y21sMGVURVRNQkVHQTFVRUNnd0tRWEJ3YkdVZ1NXNWpMakVMTUFrR0ExVUVCaE1DVlZNd0hoY05NVFF3TkRNd01UZ3hPVEEyV2hjTk16a3dORE13TVRneE9UQTJXakJuTVJzd0dRWURWUVFEREJKQmNIQnNaU0JTYjI5MElFTkJJQzBnUnpNeEpqQWtCZ05WQkFzTUhVRndjR3hsSUVObGNuUnBabWxqWVhScGIyNGdRWFYwYUc5eWFYUjVNUk13RVFZRFZRUUtEQXBCY0hCc1pTQkpibU11TVFzd0NRWURWUVFHRXdKVlV6QjJNQkFHQnlxR1NNNDlBZ0VHQlN1QkJBQWlBMklBQkpqcEx6MUFjcVR0a3lKeWdSTWMzUkNWOGNXalRuSGNGQmJaRHVXbUJTcDNaSHRmVGpqVHV4eEV0WC8xSDdZeVlsM0o2WVJiVHpCUEVWb0EvVmhZREtYMUR5eE5CMGNUZGRxWGw1ZHZNVnp0SzUxN0lEdll1VlRaWHBta09sRUtNYU5DTUVBd0hRWURWUjBPQkJZRUZMdXczcUZZTTRpYXBJcVozcjY5NjYvYXl5U3JNQThHQTFVZEV3RUIvd1FGTUFNQkFmOHdEZ1lEVlIwUEFRSC9CQVFEQWdFR01Bb0dDQ3FHU000OUJBTURBMmdBTUdVQ01RQ0Q2Y0hFRmw0YVhUUVkyZTN2OUd3T0FFWkx1Tit5UmhIRkQvM21lb3locG12T3dnUFVuUFdUeG5TNGF0K3FJeFVDTUcxbWloREsxQTNVVDgyTlF6NjBpbU9sTTI3amJkb1h0MlFmeUZNbStZaGlkRGtMRjF2TFVhZ002QmdENTZLeUtBPT0iXX0.eyJvcmlnaW5hbFRyYW5zYWN0aW9uSWQiOiIyMDAwMDAwODQ5MTY0MjEzIiwiYXV0b1JlbmV3UHJvZHVjdElkIjoicXJpbmdfZ29sZCIsInByb2R1Y3RJZCI6InFyaW5nX3NpbHZlciIsImF1dG9SZW5ld1N0YXR1cyI6MSwicmVuZXdhbFByaWNlIjoyNDk5OTAsImN1cnJlbmN5IjoiVFJZIiwic2lnbmVkRGF0ZSI6MTc0MzA4MDkyNzA1NCwiZW52aXJvbm1lbnQiOiJTYW5kYm94IiwicmVjZW50U3Vic2NyaXB0aW9uU3RhcnREYXRlIjoxNzQyOTk3MjEwMDAwLCJyZW5ld2FsRGF0ZSI6MTc0MzA4MzYxMDAwMCwiYXBwVHJhbnNhY3Rpb25JZCI6IjcwNDI3Njg4NjQxODY2NDM5OSJ9.6i5WKldCYMTGFoPuSbk7D7hgrbAuwbnGMbANyjt_b3kPhVQ3iJSV6YRmavEx4sB6GwCFuubOx9Cbq1w-eyud_Q",
    "status": 1
  },
  "version": "2.0",
  "signedDate": 1743080927082
}
```

Öncelikle buradaki notificationType propertysi için:

```json
{
  "DID_RENEW": "Abonelik, otomatik yenileme ile başarıyla yenilendi.",
  "DID_FAIL_TO_RENEW": "Abonelik, otomatik yenileme sırasında başarısız oldu.",
  "CANCEL": "Abonelik, kullanıcı tarafından iptal edildi.",
  "INTERACTIVE_RENEWAL": "Kullanıcı, uygulama içinden aboneliği manuel olarak yeniledi.",
  "DID_CHANGE_RENEWAL_STATUS": "Aboneliğin yenileme durumu değiştirildi.",
  "INITIAL_BUY": "Kullanıcı, ilk kez abonelik satın aldı.",
  "PRICE_INCREASE_CONSENT": "Fiyat artışı için kullanıcı onayı alındı.",
  "REFUND": "Abonelik için geri ödeme yapıldı.",
  "REVOKE": "Abonelik, Apple tarafından iptal edildi.",
  "SUBSCRIBED": "Kullanıcı, aboneliği başlattı.",
  "EXPIRED": "Abonelik süresi doldu.",
  "GRACE_PERIOD_EXPIRED": "Abonelik, ek süre sonunda sona erdi.",
  "BILLING_RETRY": "Ödeme işlemi tekrar denenecek.",
  "BILLING_RECOVERY": "Ödeme bilgileri güncellendi ve abonelik devam ediyor.",
  "PENDING": "Abonelik beklemede.",
  "RENEWAL_EXTENSION": "Abonelik yenileme süresi uzatıldı.",
  "OFFER_REDEEMED": "Kullanıcı, bir promosyon teklifini kullandı.",
  "EXTERNAL_PURCHASE": "Harici bir satın alma işlemi gerçekleşti."
}
```
subtype propertysi için:

```json
{
  "ACCEPTED": "Müşteri, abonelik değişikliğini kabul etti.",
  "AUTO_RENEW_DISABLED": "Müşteri aboneliğin otomatik yenilemesini devre dışı bıraktı veya Apple aboneliği iptal etti.",
  "AUTO_RENEW_ENABLED": "Müşteri aboneliğin otomatik yenilemesini etkinleştirdi.",
  "BILLING_RECOVERY": "Abonelik daha önce yenilenememişti, ödeme bilgisi güncellenip başarıyla yenilendi.",
  "BILLING_RETRY": "Abonelik, ödeme yenilemesi başarısız oldu ve faturalama yeniden denenecek.",
  "DOWNGRADE": "Müşteri aboneliğini daha düşük bir plana değiştirdi.",
  "FAILURE": "Abonelik yenileme tarihi uzatma isteği başarısız oldu.",
  "GRACE_PERIOD": "Abonelik, ödeme sorunu nedeniyle ek süreye alındı.",
  "INITIAL_BUY": "Müşteri aboneliği ilk kez satın aldı.",
  "PENDING": "Müşteri abonelik fiyat artışına henüz onay vermedi, beklemede.",
  "PRICE_INCREASE": "Müşteri abonelik fiyat artışına onay vermediği için abonelik sona erdi.",
  "PRODUCT_NOT_FOR_SALE": "Abonelik yenilenmek istediğinde ürün satışta değildi, abonelik sona erdi.",
  "RESUBSCRIBE": "Müşteri süresi dolmuş veya iptal edilmiş aboneliğini yeniden etkinleştirdi.",
  "SUMMARY": "Tüm aboneler için abonelik yenileme tarihi uzatma işlemi tamamlandı.",
  "UPGRADE": "Müşteri aboneliğini daha yüksek bir plana yükseltti.",
  "UNREPORTED": "Apple, uygulaman için bir dış satın alma token’ı oluşturdu ancak rapor alamadı.",
  "VOLUNTARY": "Müşteri, aboneliğin otomatik yenilenmesini devre dışı bıraktıktan sonra abonelik süresi doldu."
}

```

Bu iki propertyden ödeme durumu ile ilgili bilgiler alınır. Ardından signedTransactionInfo decode edilir.

```json
{
  "transactionId": "2000000883708980",
  "originalTransactionId": "2000000849164213",
  "webOrderLineItemId": "2000000092385978",
  "bundleId": "com.armakom.qring.net",
  "productId": "qring_silver",
  "subscriptionGroupIdentifier": "21627031",
  "purchaseDate": 1742997210000,
  "originalPurchaseDate": 1738824517000,
  "expiresDate": 1743083610000,
  "quantity": 1,
  "type": "Auto-Renewable Subscription",
  "inAppOwnershipType": "PURCHASED",
  "signedDate": 1743080927054,
  "environment": "Sandbox",
  "transactionReason": "PURCHASE",
  "storefront": "TUR",
  "storefrontId": "143480",
  "price": 199990,
  "currency": "TRY",
  "appTransactionId": "704276886418664399"
}
```

Buradan ödeme detayları alınır. originalTransactionId ilk ödeme ile birlikte oluşan transactionId'dir değişmez. transactionId unique'dir. Burdaki detaylar kullanılarak ilgili kullanıcının abonelik bilgileri güncellenir.
