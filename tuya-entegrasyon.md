
# 🔌 Tuya OAuth Entegrasyonu (React Native + Backend)

Bu doküman, kullanıcıların kendi Tuya hesaplarındaki cihazları **senin mobil uygulaman üzerinden kontrol etmesini** sağlamak için OAuth 2.0 tabanlı Tuya entegrasyonunu adım adım anlatır.

---

## 📦 İçerik
- [Hedef](#🎯-hedef)
- [Gereksinimler](#🛠️-gereksinimler)
- [Adım 1: Tuya OAuth Linki Oluştur](#🚀-adım-1-tuya-oauth-linki-oluştur)
- [Adım 2: Authorization Code → Access Token](#🔐-adım-2-authorization-code-→-access-token)
- [Adım 3: Kullanıcı UID'si ile eşleştir](#🧩-adım-3-kullanıcı-uidsi-ile-eşleştir)
- [Adım 4: Cihazları Listele ve Kontrol Et](#📱-adım-4-cihazları-listele-ve-kontrol-et)
- [Adım 5: Access Token Yenileme](#🔁-adım-5-access-token-yenileme)

---

## 🎯 Hedef

Kendi uygulamana giriş yapan kullanıcıların, Tuya platformundaki cihazlarını görebilmesini ve kontrol edebilmesini sağlamak.

---

## 🛠️ Gereksinimler

- Tuya IoT Developer hesabı
- Cloud Project + App SDK yapılandırması (client_id, client_secret)
- React Native frontend
- Python/Node.js backend
- Bir redirect URI (backend endpoint)

---

## 🚀 Adım 1: Tuya OAuth Linki Oluştur

React Native uygulamasında kullanıcıyı Tuya'nın yetkilendirme sayfasına yönlendir:

```ts
const clientId = 'TUYA_CLIENT_ID';
const redirectUri = 'https://your-backend.com/oauth/callback';
const state = 'random_string';
const oauthUrl = `https://auth.tuya.com/?client_id=${clientId}&response_type=code&redirect_uri=${encodeURIComponent(redirectUri)}&state=${state}&scope=all`;

Linking.openURL(oauthUrl);
```

---

## 🔐 Adım 2: Authorization Code → Access Token

Kullanıcı yetkilendirme sonrası şu URI'ya yönlendirilir:

```
https://your-backend.com/oauth/callback?code=XXXX&state=...
```

Backend bu kodla token ister:

### 🔸 Python (Flask) örneği:

```python
@app.route("/oauth/callback")
def oauth_callback():
    code = request.args.get("code")
    access_data = exchange_code_for_token(code)
    save_mapping_to_db(app_user_id, access_data["uid"], access_data["access_token"], access_data["refresh_token"])
    return "Cihaz bağlantısı başarılı."
```

### 🔸 Token alma işlemi:

```python
def create_signature(client_id, secret, t):
    message = client_id + str(t)
    return hmac.new(secret.encode(), message.encode(), hashlib.sha256).hexdigest().upper()

def exchange_code_for_token(code):
    t = int(time.time() * 1000)
    sign = create_signature(TUYA_CLIENT_ID, TUYA_CLIENT_SECRET, t)
    headers = {
        "client_id": TUYA_CLIENT_ID,
        "sign": sign,
        "t": str(t),
        "sign_method": "HMAC-SHA256"
    }
    payload = { "grant_type": "authorization_code", "code": code }
    resp = requests.post(f"{TUYA_ENDPOINT}/v1.0/token", headers=headers, json=payload)
    return resp.json().get("result", {})
```

---

## 🧩 Adım 3: Kullanıcı UID'si ile eşleştir

Access token alındığında dönen `uid` ile kendi kullanıcı ID'ni eşleştir:

| App User ID | Tuya UID | Access Token | Refresh Token |
|-------------|----------|--------------|---------------|

```python
def save_mapping_to_db(app_user_id, tuya_uid, access_token, refresh_token):
    db.insert({
        "user_id": app_user_id,
        "tuya_uid": tuya_uid,
        "access_token": access_token,
        "refresh_token": refresh_token
    })
```

---

## 📱 Adım 4: Cihazları Listele ve Kontrol Et

### 🔸 Cihazları listele

```python
def get_devices_for_user(tuya_uid, access_token):
    headers = { "Authorization": f"Bearer {access_token}" }
    url = f"{TUYA_ENDPOINT}/v1.0/users/{tuya_uid}/devices"
    return requests.get(url, headers=headers).json()
```

### 🔸 Cihaz kontrol et (örnek: aç/kapat)

```python
def send_command(device_id, access_token, command):
    headers = { "Authorization": f"Bearer {access_token}" }
    url = f"{TUYA_ENDPOINT}/v1.0/devices/{device_id}/commands"
    body = {
        "commands": [{
            "code": "switch_1",
            "value": command
        }]
    }
    return requests.post(url, headers=headers, json=body).json()
```

---

## 🔁 Adım 5: Access Token Yenileme

```python
def refresh_access_token(refresh_token):
    t = int(time.time() * 1000)
    sign = create_signature(TUYA_CLIENT_ID, TUYA_CLIENT_SECRET, t)
    headers = {
        "client_id": TUYA_CLIENT_ID,
        "sign": sign,
        "t": str(t),
        "sign_method": "HMAC-SHA256"
    }
    payload = {
        "grant_type": "refresh_token",
        "refresh_token": refresh_token
    }
    resp = requests.post(f"{TUYA_ENDPOINT}/v1.0/token", headers=headers, json=payload)
    return resp.json().get("result", {})
```

---

## ✅ Ekstra Bilgiler

- `access_token` genellikle 2 saat geçerli.
- `refresh_token` yaklaşık 30 gün geçerli.
- Cihazlara özel `code` alanları cihaz modeline göre değişir. Bunları Tuya IoT Console'da veya `GET /devices/{device_id}/functions` ile alabilirsiniz.

---

## 🧪 Test

- Tuya test cihaz hesabı açarak test kullanıcıları oluşturabilirsiniz.
- Tuya cihazlarınızı önce Tuya Smart veya Smart Life app ile eklemeyi unutmayın.

---

## 📬 İletişim

Bu entegrasyonla ilgili sorularınız varsa issue açabilir veya doğrudan bana ulaşabilirsiniz.
