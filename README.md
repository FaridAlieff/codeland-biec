# KodLand 🚀 — Quraşdırma Təlimatı

## Fayllar
```
index.html          ← Login səhifəsi
editor.html         ← Əsas redaktor
admin.html          ← Admin panel
supabase-config.js  ← ⚠️ SİZİN MƏLUMATLARINIZI BURAYA YAZIN
```

---

## 1. Supabase Hesabı Açın

1. [supabase.com](https://supabase.com) → **Start for free**
2. GitHub ilə giriş edin
3. **New Project** yaradın → ad verin, şifrə seçin, region: **Frankfurt (EU)**

---

## 2. supabase-config.js Faylını Doldurun

Project yarandıqdan sonra:
**Settings → API** bölməsinə gedin

```js
const SUPABASE_URL  = 'https://XXXXXXXXXXXX.supabase.co';  // Project URL
const SUPABASE_ANON = 'eyJhbGciOiJIUzI1NiIs...';           // anon public key
```

---

## 3. Verilənlər Bazası Cədvəllərini Yaradın

Supabase → **SQL Editor** → **New query** → aşağıdakı SQL-i yapışdırın → **Run**:

```sql
-- İstifadəçi profilləri
CREATE TABLE profiles (
  id          UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email       TEXT,
  full_name   TEXT,
  role        TEXT DEFAULT 'user',  -- 'user' və ya 'admin'
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Kod arxivi
CREATE TABLE code_archive (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID REFERENCES profiles(id) ON DELETE CASCADE,
  html_code     TEXT,
  css_code      TEXT,
  combined_code TEXT,
  active_tab    TEXT DEFAULT 'html',
  lang          TEXT DEFAULT 'az',
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

-- Profili avtomatik yarat (hər qeydiyyatda)
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.profiles (id, email, full_name, role)
  VALUES (
    NEW.id,
    NEW.email,
    NEW.raw_user_meta_data->>'full_name',
    'user'
  )
  ON CONFLICT (id) DO NOTHING;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE OR REPLACE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();
```

---

## 4. Row Level Security (RLS) Qaydaları

```sql
-- profiles: hər kəs öz profilini görsün, admin hamısını
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own profile"
  ON profiles FOR SELECT
  USING (auth.uid() = id OR EXISTS (
    SELECT 1 FROM profiles WHERE id = auth.uid() AND role = 'admin'
  ));

CREATE POLICY "Users can update own profile"
  ON profiles FOR UPDATE USING (auth.uid() = id);

CREATE POLICY "Admin can insert profiles"
  ON profiles FOR INSERT
  WITH CHECK (true);

CREATE POLICY "Admin can delete profiles"
  ON profiles FOR DELETE
  USING (EXISTS (
    SELECT 1 FROM profiles WHERE id = auth.uid() AND role = 'admin'
  ));

-- code_archive: hər kəs öz kodlarını görsün, admin hamısını
ALTER TABLE code_archive ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users manage own archive"
  ON code_archive FOR ALL
  USING (auth.uid() = user_id OR EXISTS (
    SELECT 1 FROM profiles WHERE id = auth.uid() AND role = 'admin'
  ));
```

---

## 5. İlk Admin İstifadəçisini Əlavə Edin

**Supabase → Authentication → Users → Invite user** ilə özünüzü əlavə edin.

Sonra **SQL Editor**-da:
```sql
-- Öz e-poçtunuzu yazın
UPDATE profiles SET role = 'admin'
WHERE email = 'siz@email.com';
```

---

## 6. Netlify-ə Yükləyin

1. Bütün faylları GitHub reposuna atın
2. [netlify.com](https://netlify.com) → **Add new site → Import from Git**
3. Reposu seçin → **Deploy**
4. Hazırdır! 🎉

---

## Xülasə: Səhifələr

| Səhifə | URL | Kimə |
|--------|-----|------|
| `index.html` | `/` | Hamı (login) |
| `editor.html` | `/editor.html` | Login olmuş istifadəçilər |
| `admin.html` | `/admin.html` | Yalnız adminlər |

---

## Funksiyalar

- ✅ **Login / Qeydiyyat** — Supabase Auth
- ✅ **3 dil** — AZ / EN / RU (bayraq düymələri)
- ✅ **HTML + CSS redaktor** — Run Et / Yeni Tab
- ✅ **Arxiv** — hər Run etdikdə avtomatik saxlanır
- ✅ **Arxiv Çekmeci** — tarix/saat ilə göstərilir, yükləmək/silmək olur
- ✅ **Admin Panel** — istifadəçi əlavə et/sil, arxivi izlə, statistika
