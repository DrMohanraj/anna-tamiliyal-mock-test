# அண்ணா தமிழியல் நிறுவனம் — Supabase Backend அமைப்பு வழிகாட்டி

இந்த வழிகாட்டி UG TRB Mock Test தளத்திற்கு **Auth + Database + Payment** ஆகியவற்றை
ஒரே Supabase project-ல் இணைக்க உதவும். இதை உங்கள் Supabase dashboard-ல் நேரடியாகப்
பயன்படுத்தலாம்.

---

## 1. Supabase Project உருவாக்குதல்

1. https://supabase.com → **New Project**
2. Project பெயர், database password கொடுத்து உருவாக்கவும்
3. Settings → API-ல் இருந்து இவற்றை copy செய்யவும்:
   - `Project URL`
   - `anon public key`

இவை இரண்டும் தான் உங்கள் HTML site-ல் Supabase-உடன் பேச தேவைப்படும்.

---

## 2. GitHub OAuth Login அமைத்தல்

Supabase-ல் GitHub OAuth **built-in support** உள்ளது — நீங்கள் ஏதும் தனியாக code
எழுத வேண்டியதில்லை.

1. GitHub.com → Settings → Developer settings → OAuth Apps → **New OAuth App**
   - Homepage URL: உங்கள் site URL
   - Authorization callback URL: Supabase dashboard-ல் Authentication → Providers →
     GitHub-ல் கொடுக்கப்பட்டிருக்கும் URL-ஐ இங்கே ஒட்டவும்
2. Client ID & Client Secret-ஐ Supabase Dashboard → Authentication → Providers →
   GitHub-ல் ஒட்டி **Enable** செய்யவும்

இதன்பின் site-ல் இருந்து:
```js
await supabase.auth.signInWithOAuth({ provider: 'github' });
```
இதை அழைத்தால் போதும் — GitHub login flow தானாக நடக்கும்.

**கவனிக்க:** இது "யார் login ஆனார்" எனச் சொல்லும் — "இவர் பணம் கட்டியவரா" எனச்
சொல்லாது. அதற்கு கீழே `profiles` table தேவை.

---

## 3. Database Schema (SQL)

Supabase Dashboard → SQL Editor-ல் இதை run செய்யவும்:

```sql
-- ஒவ்வொரு பயனருக்கும் கூடுதல் தகவல் (premium status உள்பட)
create table profiles (
  id uuid references auth.users primary key,
  full_name text,
  is_premium boolean default false,
  premium_until timestamp with time zone,
  created_at timestamp with time zone default now()
);

-- தேர்வுகள் (mock test / unit test)
create table tests (
  id uuid primary key default gen_random_uuid(),
  title text not null,
  tier text check (tier in ('free','premium')) default 'free',
  duration_minutes int default 60,
  created_at timestamp with time zone default now()
);

-- ஒவ்வொரு தேர்வுக்குமான வினாக்கள்
create table questions (
  id uuid primary key default gen_random_uuid(),
  test_id uuid references tests(id) on delete cascade,
  topic text,
  question text not null,
  option_a text not null,
  option_b text not null,
  option_c text not null,
  option_d text not null,
  correct_index int check (correct_index between 0 and 3),
  position int
);

-- பயனர் ஒரு தேர்வை எழுதிய பதிவு
create table attempts (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id),
  test_id uuid references tests(id),
  score int,
  total int,
  answers jsonb,
  submitted_at timestamp with time zone default now()
);

-- சந்தா பணம் பதிவுகள் (webhook மூலம் நிரப்பப்படும்)
create table payments (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id),
  provider text,              -- 'razorpay' etc.
  provider_payment_id text,
  amount int,                 -- paise-ல்
  status text,                -- 'success' | 'failed'
  created_at timestamp with time zone default now()
);
```

### Row Level Security (RLS) — முக்கியம்

```sql
alter table profiles enable row level security;
alter table attempts enable row level security;
alter table tests enable row level security;
alter table questions enable row level security;

-- பயனர் தன் சொந்த profile-ஐ மட்டும் பார்க்க/திருத்த முடியும்
create policy "own profile" on profiles
  for select using (auth.uid() = id);

-- எல்லோரும் tests பட்டியலைப் பார்க்கலாம்
create policy "public tests" on tests for select using (true);

-- Free test வினாக்கள் எல்லோருக்கும்; Premium test வினாக்கள் is_premium=true ஆனவர்களுக்கு மட்டும்
create policy "gated questions" on questions
  for select using (
    exists (
      select 1 from tests t
      where t.id = questions.test_id
      and (
        t.tier = 'free'
        or exists (
          select 1 from profiles p
          where p.id = auth.uid() and p.is_premium = true
        )
      )
    )
  );

-- பயனர் தன் attempts-ஐ மட்டும் பார்க்க/சேர்க்க முடியும்
create policy "own attempts" on attempts
  for all using (auth.uid() = user_id);
```

இதுவே **"பணம் கட்டியவர்கள் மட்டும் 100+ தேர்வு"** என்ற rule-ஐ database level-லேயே
உறுதி செய்யும் — front-end-ல் மட்டும் lock வைப்பதற்குப் பதிலாக, question data
கூட server-ல் இருந்தே premium இல்லாதவருக்குக் கிடைக்காது.

---

## 4. Payment Webhook (Razorpay உதாரணம்)

Indian audience-க்கு Razorpay பொதுவாகப் பயன்படுத்தப்படுகிறது. Flow:

1. Site-ல் "சந்தா எடு" பொத்தான் → Razorpay Checkout திறக்கும்
2. பணம் வெற்றி → Razorpay உங்கள் **Supabase Edge Function**-க்கு webhook அனுப்பும்
3. அந்த function `payments` table-ல் பதிவு செய்து, `profiles.is_premium = true` ஆக்கும்

Supabase Edge Function (`supabase/functions/razorpay-webhook/index.ts`):

```ts
import { serve } from "https://deno.land/std/http/server.ts";
import { createClient } from "https://esm.sh/@supabase/supabase-js";

const supabase = createClient(
  Deno.env.get("SUPABASE_URL")!,
  Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
);

serve(async (req) => {
  const payload = await req.json();
  // Razorpay signature-ஐ சரிபார்க்கவும் (webhook secret வைத்து) — production-ல் கட்டாயம்

  if (payload.event === "payment.captured") {
    const userId = payload.payload.payment.entity.notes.user_id;
    await supabase.from("payments").insert({
      user_id: userId,
      provider: "razorpay",
      provider_payment_id: payload.payload.payment.entity.id,
      amount: payload.payload.payment.entity.amount,
      status: "success",
    });
    await supabase.from("profiles")
      .update({ is_premium: true })
      .eq("id", userId);
  }

  return new Response("ok", { status: 200 });
});
```

Deploy: `supabase functions deploy razorpay-webhook` — பின் அந்த URL-ஐ Razorpay
Dashboard → Webhooks-ல் பதிவு செய்யவும்.

---

## 5. Site-ல் இருந்து Supabase-ஐ அழைப்பது

தற்போதைய HTML-ல் `TESTS` array hardcode ஆக இருக்கிறது. Backend இணைத்தபின்:

```html
<script src="https://esm.sh/@supabase/supabase-js@2"></script>
<script>
  const supabase = window.supabase.createClient(
    "https://YOUR-PROJECT.supabase.co",
    "YOUR-ANON-KEY"
  );

  async function loadTests(){
    const { data, error } = await supabase.from('tests').select('*');
    // இதை TESTS array-க்குப் பதிலாகப் பயன்படுத்தலாம்
  }

  async function loadQuestions(testId){
    const { data } = await supabase
      .from('questions').select('*')
      .eq('test_id', testId)
      .order('position');
    // Premium test-க்கு RLS தானாகவே அணுகலைத் தடுக்கும்
  }
</script>
```

இதன் மூலம் "நீங்களே எளிதாக வினாக்களை மாற்றுவது" கேள்விக்கும் தீர்வு கிடைக்கும் —
JS file-ஐ திருத்துவதற்குப் பதிலாக, Supabase Table Editor-ல் (spreadsheet போன்ற
UI) நேரடியாக வினாக்களைச் சேர்க்கலாம்/திருத்தலாம்.

---

## அடுத்த படிகள் (சுருக்கம்)

| படி | வேலை |
|---|---|
| 1 | Supabase project + GitHub OAuth enable |
| 2 | மேலே உள்ள SQL schema-ஐ run செய்தல் |
| 3 | Razorpay account + webhook edge function deploy |
| 4 | HTML site-ஐ static `TESTS` array-க்குப் பதில் Supabase calls-ஆக மாற்றுதல் |
| 5 | Table Editor மூலம் 100+ வினாக்களை நேரடியாகச் சேர்த்தல் |

இது ஒரு multi-day engineering வேலை — full working integration-ஐ படிப்படியாகக்
கட்ட விரும்பினால், **Claude Code** (terminal/desktop app) இதை repo-ஆக வைத்து
build செய்ய மிகவும் ஏற்றது, ஏனெனில் இதற்கு பல கோப்புகள், testing, deployment
config தேவைப்படும்.
