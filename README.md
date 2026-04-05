# KodLand 🚀 — Tam Quraşdırma Təlimatı

## Fayllar
```
index.html          ← Giriş səhifəsi (username + şifrə)
editor.html         ← Şagird redaktoru
admin.html          ← Admin panel
supabase-config.js  ← ⚠️ Öz məlumatlarınızı buraya yazın
```

---

## Addım 1 — Supabase Hesabı

1. [supabase.com](https://supabase.com) → **Start for free**
2. **New Project** → ad verin, region: **Frankfurt (EU)**
3. **Settings → API** bölməsinə gedin, iki şeyi kopyalayın:
   - `Project URL`
   - `anon public` key

`supabase-config.js` faylını açın, dəyərləri yazın:
```js
const SUPABASE_URL  = 'https://xxxxxxxxxxxx.supabase.co';
const SUPABASE_ANON = 'eyJhbGciOiJIUzI1NiIs...';
const DOMAIN_SUFFIX = '@kodland.local';
```

---

## Addım 2 — Email Konfirmasiyasını Söndürün

**Supabase → Authentication → Providers → Email**

- **"Confirm email"** — SÖNDÜRÜN (OFF)
- Saxlayın

> Bu olmadan şagirdlər sistemə girə bilməz!

---

## Addım 3 — Verilənlər Bazası

**Supabase → SQL Editor → New query** — aşağıdakı SQL-i yapışdırın → **Run**:

```sql
-- ═══════════════════════════════════════
--  1. PROFILES CƏDVƏLİ
-- ═══════════════════════════════════════
CREATE TABLE IF NOT EXISTS profiles (
  id          UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email       TEXT,
  full_name   TEXT,
  username    TEXT UNIQUE,
  role        TEXT DEFAULT 'user',
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- ═══════════════════════════════════════
--  2. KOD ARXİVİ
-- ═══════════════════════════════════════
CREATE TABLE IF NOT EXISTS code_archive (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID REFERENCES profiles(id) ON DELETE CASCADE,
  html_code     TEXT DEFAULT '',
  css_code      TEXT DEFAULT '',
  combined_code TEXT DEFAULT '',
  active_tab    TEXT DEFAULT 'html',
  lang          TEXT DEFAULT 'az',
  trigger       TEXT DEFAULT 'run',  -- 'run' və ya 'newtab'
  score         INTEGER CHECK (score BETWEEN 1 AND 5),
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

-- ═══════════════════════════════════════
--  3. OTOMATİK PROFİL YARAT
-- ═══════════════════════════════════════
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.profiles (id, email, full_name, username, role)
  VALUES (
    NEW.id,
    NEW.email,
    COALESCE(NEW.raw_user_meta_data->>'full_name', ''),
    COALESCE(NEW.raw_user_meta_data->>'username',
             SPLIT_PART(NEW.email, '@', 1)),
    'user'
  )
  ON CONFLICT (id) DO UPDATE SET
    full_name = EXCLUDED.full_name,
    username  = EXCLUDED.username;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

DROP TRIGGER IF EXISTS on_auth_user_created ON auth.users;
CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();

-- ═══════════════════════════════════════
--  4. ROW LEVEL SECURITY
-- ═══════════════════════════════════════
ALTER TABLE profiles     ENABLE ROW LEVEL SECURITY;
ALTER TABLE code_archive ENABLE ROW LEVEL SECURITY;

-- profiles: öz profilini gör, admin hamısını görsün
DROP POLICY IF EXISTS "profiles_select" ON profiles;
CREATE POLICY "profiles_select" ON profiles FOR SELECT
  USING (
    auth.uid() = id
    OR EXISTS (SELECT 1 FROM profiles WHERE id = auth.uid() AND role = 'admin')
  );

DROP POLICY IF EXISTS "profiles_insert" ON profiles;
CREATE POLICY "profiles_insert" ON profiles FOR INSERT WITH CHECK (true);

DROP POLICY IF EXISTS "profiles_update" ON profiles;
CREATE POLICY "profiles_update" ON profiles FOR UPDATE USING (true);

DROP POLICY IF EXISTS "profiles_delete" ON profiles;
CREATE POLICY "profiles_delete" ON profiles FOR DELETE
  USING (EXISTS (SELECT 1 FROM profiles WHERE id = auth.uid() AND role = 'admin'));

-- code_archive: öz kodunu gör; admin hamısını görsün və qiymət versin
DROP POLICY IF EXISTS "archive_all" ON code_archive;
CREATE POLICY "archive_all" ON code_archive FOR ALL
  USING (
    auth.uid() = user_id
    OR EXISTS (SELECT 1 FROM profiles WHERE id = auth.uid() AND role = 'admin')
  );
```

---

## Addım 4 — İlk Admin Yaradın

1. **Supabase → Authentication → Users → Add user**
   - Email: `admin@kodland.local`
   - Password: seçdiyiniz şifrə
   - **"Auto Confirm User"** — işarələyin

2. **SQL Editor**-da:
```sql
UPDATE profiles SET role = 'admin', username = 'admin'
WHERE email = 'admin@kodland.local';
```

3. Sistemə giriş edin:
   - İstifadəçi adı: `admin`
   - Şifrə: yuxarıda seçdiyiniz

---

## Addım 5 — Netlify Deploy

1. Bütün faylları GitHub reposuna atın
2. [netlify.com](https://netlify.com) → **Add new site → Import from Git**
3. Reposu seçin → **Deploy site**
4. Hazırdır! 🎉

---

## Sistem Necə İşləyir

### Şagird girişi
- Yalnız **İstifadəçi adı + Şifrə** daxil edir
- Admin hər şagird üçün hesab yaradır

### Arxiv
- Şagird **Run Et** vuranda da, **Yeni Tab** seçəndə də kod arxivə düşür
- Arxiv çekmecəsindən keçmiş kodlara baxa bilər, yükləyə bilər

### Qiymətləndirmə (Admin)
- Arxiv → "Bax & Qiymət" düyməsi → kodun nəticəsini görür
- 1-5 ulduz verir
- Şagirdin **ortalama balı** həm admin paneldə, həm şagirdin öz səhifəsinin yuxarısında görünür

### Ortalama bal (Şagird)
- Header-də: `⭐⭐⭐ 3.5` formatında görünür
- Hər yeni qiymət veriləndə avtomatik yenilənir

---

## Səhifələr Xülasəsi

| Səhifə | Kimə açıqdır |
|--------|-------------|
| `index.html` | Hamı |
| `editor.html` | Login olmuş şagirdlər |
| `admin.html` | Yalnız adminlər |
