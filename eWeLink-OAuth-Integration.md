
# 🔌 eWeLink OAuth Entegrasyonu (React Native + Backend)

Bu doküman, kullanıcıların kendi **eWeLink** hesaplarındaki cihazları senin mobil uygulaman üzerinden kontrol etmesini sağlamak için OAuth 2.0 tabanlı entegrasyonu adım adım açıklar.

---

## 📦 İçerik
- [Hedef](#🎯-hedef)
- [Gereksinimler](#🛠️-gereksinimler)
- [Adım 1: eWeLink OAuth Linki Oluştur](#🚀-adım-1-ewelink-oauth-linki-oluştur)
- [Adım 2: Authorization Code → Access Token](#🔐-adım-2-authorization-code-→-access-token)
- [Adım 3: Kullanıcı UID'si ile eşleştir](#🧩-adım-3-kullanıcı-uidsi-ile-eşleştir)
- [Adım 4: Cihazları Listele ve Kontrol Et](#📱-adım-4-cihazları-listele-ve-kontrol-et)
- [Adım 5: Access Token Yenileme](#🔁-adım-5-access-token-yenileme)

---

## 🎯 Hedef

Kullanıcı, kendi eWeLink hesabına giriş yaparak cihazlarını senin uygulamanla ilişkilendirecek. Sen de bu cihazlara API üzerinden erişip komut gönderebileceksin.

---

## 🛠️ Gereksinimler

- eWeLink Developer hesabı (https://dev.ewelink.cc/)
- Kayıtlı bir uygulama (Client ID, Client Secret)
- Redirect URI (backend endpoint)
- React Native frontend
- Python/Node.js backend

---

## 🚀 Adım 1: eWeLink OAuth Linki Oluştur

React Native uygulamasında kullanıcıyı şu bağlantıya yönlendir:

```ts
const clientId = 'YOUR_CLIENT_ID';
const redirectUri = 'https://your-backend.com/oauth/ewelink-callback';
const state = 'random_state';

const oauthUrl = \`https://eu-disp.coolkit.cc/oauth2/authorize?client_id=\${clientId}&redirect_uri=\${encodeURIComponent(redirectUri)}&response_type=code&scope=user:info&state=\${state}\`;

Linking.openURL(oauthUrl);
```

---

## 🔐 Adım 2: Authorization Code → Access Token

Kullanıcı giriş yaptıktan sonra `redirect_uri`'na `code` gelir:

```
https://your-backend.com/oauth/ewelink-callback?code=XYZ&state=abc
```

### Python Örneği (Flask):

```python
@app.route("/oauth/ewelink-callback")
def ewelink_callback():
    code = request.args.get("code")
    token_data = get_ewelink_token(code)
    save_mapping(app_user_id, token_data["uid"], token_data["access_token"], token_data["refresh_token"])
    return "eWeLink hesabı başarıyla bağlandı!"
```

### Access Token Al:

```python
def get_ewelink_token(code):
    payload = {
        "grant_type": "authorization_code",
        "code": code,
        "client_id": "YOUR_CLIENT_ID",
        "client_secret": "YOUR_CLIENT_SECRET",
        "redirect_uri": "https://your-backend.com/oauth/ewelink-callback"
    }
    headers = { "Content-Type": "application/x-www-form-urlencoded" }
    response = requests.post("https://eu-disp.coolkit.cc/oauth2/token", data=payload, headers=headers)
    return response.json()
```

---

## 🧩 Adım 3: Kullanıcı UID'si ile eşleştir

| App User ID | eWeLink UID | Access Token | Refresh Token |
|-------------|-------------|---------------|---------------|

```python
def save_mapping(app_user_id, uid, access_token, refresh_token):
    db.insert({
        "user_id": app_user_id,
        "ewelink_uid": uid,
        "access_token": access_token,
        "refresh_token": refresh_token
    })
```

---

## 📱 Adım 4: Cihazları Listele ve Kontrol Et

### Cihazları Listele:

```python
def get_devices(access_token):
    headers = { "Authorization": f"Bearer {access_token}" }
    resp = requests.get("https://eu-api.coolkit.cc/v2/device", headers=headers)
    return resp.json()
```

### Cihaz Kontrol Et:

```python
def toggle_device(device_id, access_token, state=True):
    headers = {
        "Authorization": f"Bearer {access_token}",
        "Content-Type": "application/json"
    }
    body = {
        "deviceid": device_id,
        "params": { "switch": "on" if state else "off" }
    }
    return requests.post("https://eu-api.coolkit.cc/v2/device/thing/status", json=body, headers=headers).json()
```

---

## 🔁 Adım 5: Access Token Yenileme

```python
def refresh_ewelink_token(refresh_token):
    payload = {
        "grant_type": "refresh_token",
        "refresh_token": refresh_token,
        "client_id": "YOUR_CLIENT_ID",
        "client_secret": "YOUR_CLIENT_SECRET",
    }
    headers = { "Content-Type": "application/x-www-form-urlencoded" }
    response = requests.post("https://eu-disp.coolkit.cc/oauth2/token", data=payload, headers=headers)
    return response.json()
```

---

## 🧪 Test

- Gerçek cihazlarla test etmek için eWeLink uygulamasına cihazları ekleyin.
- Test kullanıcıları oluşturun ve redirect URI’nızın whitelist'te olduğundan emin olun.

---

## 📬 İletişim

Bu entegrasyonla ilgili soru, öneri ve katkılarınız için bana ulaşabilirsiniz.
