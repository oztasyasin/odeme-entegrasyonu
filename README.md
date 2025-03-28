
# Ödeme Entegrasyonu

Bu döküman baştan sona bir react-native projesinin ödeme sisteminin entegrasyonunu içerir.




## Adımlar

- react-native-iap entegrasyonu ve mobil ödeme entegrasyonu
- Google ile yapılan ödemelerin server sideda kontrolü ve işlenmesi
- Google server notifications
- Apple ile yapılan ödemelerin server sideda kontrolü ve işlenmesi
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
