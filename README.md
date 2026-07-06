# Logowanie, sesja i role

**Zadanie:** opisać krok po kroku, co dzieje się od wpisania hasła do zobaczenia
pulpitu — z uwzględnieniem **haszowania**, **sesji**, **roli** i **ochrony trasy**.
Na końcu: *co by się stało, gdyby haseł nie haszowano?*

Przykłady z mojego realnego projektu **Serialoteka** (Next.js App Router +
Supabase Auth): https://github.com/Riceookie/serialoteka. „Pulpit" = chroniona
strona `/watchlist`.

## Zanim się zalogujesz: jak hasło w ogóle trafia do bazy (rejestracja)

Baza **nigdy nie przechowuje Twojego hasła**. Przy zakładaniu konta serwer robi
z hasła **hash** — jednokierunkowy skrót (Supabase używa **bcrypt**), do którego
dokłada się **sól** (losowy dodatek, żeby dwa takie same hasła dały różne hashe).
W bazie ląduje np. `$2a$10$N9qo8uL... `, a nie `tajneHaslo123`.

Kluczowa cecha: z hasła łatwo policzyć hash, ale **z hasha nie da się odtworzyć
hasła**. Dlatego nawet admin bazy nie zna Twojego hasła.

## Krok po kroku: od wpisania hasła do pulpitu

1. **Wpisujesz e-mail + hasło** w formularzu (`app/login/page.jsx`) i klikasz
   „Zaloguj". Formularz wywołuje `signIn(email, password)`.

2. **Hasło leci po HTTPS** do serwera uwierzytelniania (u nas Supabase Auth).
   Szyfrowany kanał = po drodze nikt go nie podejrzy. Hasła **nie zapisujemy**
   nigdzie po stronie przeglądarki.

3. **Serwer weryfikuje hasło przez hash.** Bierze podane hasło, **haszuje je tą
   samą metodą** i porównuje wynik z hashem zapisanym przy rejestracji.
   - hashe się zgadzają → hasło poprawne,
   - nie zgadzają → `Invalid login credentials` (u nas tłumaczone na „Złe dane
     logowania").

   Zwróć uwagę: serwer **porównuje hashe, nie hasła** — bo hasła w czystej postaci
   nie ma nigdzie zapisanego.

4. **Serwer wystawia sesję.** Po poprawnym haśle generuje **token sesji**
   (`access_token`, w formacie JWT). Token to podpisany „bilet": zawiera m.in. `id`
   użytkownika i **rolę**, oraz datę ważności. Podpis serwera sprawia, że tokenu
   nie da się podrobić ani zmienić (np. dopisać sobie roli `admin`).

5. **Przeglądarka zapamiętuje sesję.** `AuthProvider` (`components/AuthProvider.jsx`)
   trzyma `session` i nasłuchuje jej zmian (`onAuthStateChange`). Od teraz apka
   „wie", że jesteś zalogowany.

6. **Przekierowanie na pulpit.** `router.replace('/watchlist')` przenosi Cię na
   chronioną stronę.

7. **Ochrona trasy — dwie warstwy:**
   - **Front (wygoda/UX):** `/watchlist` na wejściu sprawdza
     `if (!session) router.replace('/login')`. To tylko chowa widok przed
     niezalogowanym — **nie jest prawdziwym zabezpieczeniem**, bo front można oszukać.
   - **Serwer (prawdziwa ochrona):** każde pobranie danych to `fetch` z dołączonym
     tokenem (`Authorization: Bearer <token>` w `authFetch`). Endpoint
     `app/api/watchlist/route.js` woła `getUserFromRequest(req)`, który **weryfikuje
     token**; bez ważnego tokenu zwraca **401 „Wymagane logowanie"** i danych nie wyda.

8. **Rola decyduje, co wolno.** Token niesie rolę użytkownika, a serwer sprawdza ją,
   zanim wyda dane. W Serialotece egzekwowana reguła autoryzacji brzmi „widzisz
   **tylko swoje** wpisy" — endpoint filtruje po zweryfikowanym użytkowniku:
   `.eq('user_id', user.id)`. Gdyby doszła rola `admin`, endpoint mógłby np.
   pozwolić jej czytać wpisy wszystkich (`if (user.role === 'admin') …`). **Rolę
   zawsze sprawdza serwer**, nigdy front.

9. **Pulpit się renderuje** danymi zwróconymi przez API — widzisz swoją listę.

```
[hasło] ──HTTPS──> serwer: hash(podane) == hash(zapisany)?
                              │ tak
                              ▼
                     wystawia token sesji (JWT: id + rola)
                              │
        przeglądarka zapisuje sesję ──> router → /watchlist (pulpit)
                              │
        każdy fetch: Authorization: Bearer <token>
                              ▼
        API: getUserFromRequest → brak/zły token = 401
             rola/właściciel OK → .eq('user_id', user.id) → dane → pulpit
```

## Co się dzieje po usunięciu (lub wygaśnięciu) tokena?

Token nie jest w Serialotece trzymany „w powietrzu” — biblioteka `supabase-js`
zapisuje sesję w **`localStorage`** przeglądarki (domyślny `persistSession`).
Dlatego przeładowanie strony nie wylogowuje: `supabase.auth.getSession()`
(`components/AuthProvider.jsx`) odczytuje token z powrotem.

Co się stanie, gdy token **zniknie** — bo user kliknął „Wyloguj”
(`supabase.auth.signOut()` czyści `localStorage`), wyczyścił dane przeglądarki,
albo token po prostu **wygasł**:

1. **Front natychmiast „wie”, że nie ma sesji.** `onAuthStateChange` odpala się ze
   stanem `null`, `AuthProvider` ustawia `session = null`. Strona `/watchlist`
   ma w `useEffect` warunek `if (!session) router.replace('/login')` — więc
   niezalogowany widok znika, a użytkownik ląduje na logowaniu.
2. **Serwer i tak by nie wydał danych.** Nawet gdyby ktoś obszedł front (wpisał
   `/watchlist` ręcznie, wywołał API z konsoli), `authFetch` nie dołączy nagłówka
   `Authorization` (bo `session?.access_token` jest `undefined`). Endpoint woła
   `getUserFromRequest(req)`, ten nie znajduje tokenu → zwraca `null` → API
   odpowiada **401 „Wymagane logowanie”**. Zero danych.
3. **Wygaśnięcie to osobny przypadek niż usunięcie.** `access_token` (JWT) ma krótką
   datę ważności; `supabase-js` w tle odświeża go **refresh tokenem**
   (`autoRefreshToken`). Jeśli jednak i refresh token jest nieważny/usunięty —
   odświeżenie się nie uda i efekt jest taki sam jak przy `signOut()`: sesja pada,
   następny `getUser(token)` po stronie serwera zwróci błąd → 401.

Krótko: **usunięcie tokena = wylogowanie**. Front chowa widok od razu, a serwer
i tak blokuje dostęp — bo prawdziwą bramką jest weryfikacja tokenu, nie stan Reacta.

## Jak rola z JWT mapuje się na konkretny widok?

W tokenie (JWT) siedzi claim `role`. W Serialotece każdy zalogowany dostaje tę samą
rolę Supabase — `authenticated` — więc realnie działa tu jedna reguła autoryzacji:
„widzisz **tylko swoje** wpisy” (`.eq('user_id', user.id)` w
`app/api/watchlist/route.js`). To autoryzacja **po właścicielu**, nie po roli.

Gdyby wprowadzić drugą rolę (np. `admin`), mapowanie rola→widok wygląda tak i ma
**dwie warstwy**:

- **Front — który widok w ogóle pokazać (UX).** Rolę można odczytać z sesji
  (`session.user`, claim w tokenie) i na tej podstawie decydować, co wyrenderować:
  ```js
  const { user } = useAuth()
  const isAdmin = user?.app_metadata?.role === 'admin'
  // pokaż link „Panel admina” i trasę /admin tylko adminowi
  ```
  To wyłącznie wygoda — chowanie/pokazywanie elementów. Front **nie jest
  zabezpieczeniem** (można go obejść).
- **Serwer — czy te dane naprawdę wolno wydać (egzekucja).** Endpoint widoku
  admina najpierw weryfikuje token, a potem **sprawdza rolę** ze zweryfikowanego
  tokenu, zanim cokolwiek zwróci:
  ```js
  const user = await getUserFromRequest(req)            // weryfikacja tokenu
  if (!user) return json({ error: 'Wymagane logowanie' }, 401)
  if (user.app_metadata?.role !== 'admin')              // egzekucja roli
    return json({ error: 'Brak uprawnień' }, 403)
  // dopiero teraz: dane wszystkich użytkowników (widok admina)
  ```

Czyli: **rola w tokenie → front wybiera widok do pokazania → serwer sprawdza tę
samą rolę, zanim wyda dane tego widoku.** Wiążąca jest zawsze warstwa serwerowa;
front tylko dopasowuje interfejs. Rola musi siedzieć w **podpisanym** tokenie
(`app_metadata`, ustawiane przez serwer/Supabase), a nie w czymś, co user może
sobie sam zmienić — inaczej każdy „awansowałby się” na admina.

## Dlaczego JWT (token), a nie klasyczne ciasteczko sesji?

Oba rozwiązania dają to samo: „po zalogowaniu nie pytamy o hasło co żądanie”.
Różnią się tym, **co** dołączamy do żądania i **jak** serwer to sprawdza.

- **Ciasteczko sesji (klasyka):** w cookie leży tylko **nieprzejrzysty identyfikator**.
  Przy każdym żądaniu serwer musi **odpytać magazyn sesji** (bazę/Redis), żeby
  sprawdzić, do kogo ten identyfikator należy i czy jeszcze ważny. Stan trzyma serwer.
- **JWT (nasz wybór):** token jest **samowystarczalny** — niesie `id`, `role`, `exp`
  i **podpis**. Serwer weryfikuje sam podpis i datę ważności, **bez zaglądania do
  żadnego magazynu sesji**. U nas robi to `getUserFromRequest` przez
  `supabase.auth.getUser(token)`.

Dlaczego to pasuje akurat tutaj:

1. **Bezstanowość.** Nie utrzymujemy własnej tabeli sesji — tożsamość i rola
   „jadą” w samym tokenie. Prościej i taniej przy Supabase, które i tak wydaje JWT.
2. **Rozdzielony front i API / różne domeny.** Front (SPA/statyk, np. GitHub Pages)
   gada z Supabase i z naszym API pod **innym adresem**. `Authorization: Bearer <token>`
   działa niezależnie od domeny; ciasteczka są przypięte do domeny i wymagają
   dłubania w `SameSite`, `CORS` i `withCredentials`.
3. **Model Supabase Auth.** To on generuje JWT i podpisuje go swoim kluczem —
   idziemy z ziarnem narzędzia, nie pod prąd.

**Uczciwie o kompromisie:** ciasteczko z flagą **`httpOnly`** jest odporne na
kradzież przez JavaScript, a JWT w `localStorage` **można wykraść atakiem XSS**.
Dlatego wybór „JWT vs cookie” to nie „lepsze/gorsze”, tylko dopasowanie do
architektury: dla rozdzielonego frontu i API wielu dostawców (w tym Supabase) JWT
jest naturalny — ceną jest większa dbałość o brak XSS i krótki czas życia tokenu.

## Cztery pojęcia w jednym zdaniu

- **Haszowanie** — zamiana hasła w nieodwracalny skrót; w bazie leży hash, nie hasło.
- **Sesja** — podpisany token („bilet"), który po zalogowaniu dołączamy do każdego
  żądania, żeby serwer wiedział, kim jesteś, bez pytania o hasło za każdym razem.
- **Rola** — informacja „co wolno temu użytkownikowi" (np. `user` vs `admin`),
  sprawdzana **po stronie serwera** przed wydaniem danych/akcji.
- **Ochrona trasy** — pilnowanie, że pod dany adres/endpoint wejdzie tylko ktoś
  uprawniony; front chowa widok, ale **wiążąca jest kontrola na serwerze**.

## „Co by się stało, gdyby haseł nie haszowano?”

Gdyby hasła leżały w bazie **jawnym tekstem**:

1. **Wyciek bazy = natychmiastowy wyciek wszystkich haseł.** Jeden udany atak na
   bazę (albo nieuczciwy pracownik z dostępem) i napastnik zna hasła wszystkich —
   bez łamania czegokolwiek.
2. **Efekt domina na inne serwisy.** Ludzie reużywają haseł. Hasło z Twojej apki
   pasuje wtedy do ich poczty, banku, social mediów — jeden wyciek kompromituje
   konta w zupełnie innych miejscach (tzw. *credential stuffing*).
3. **Każdy z dostępem do bazy widzi cudze hasła.** Admin, wsparcie, proces logujący
   zapytania — hasło przestaje być tajemnicą znaną tylko użytkownikowi.
4. **Nie da się „cofnąć” szkody.** Hasła to nie numery kart, które można unieważnić
   — trzeba wymusić reset u wszystkich i liczyć, że nikt nie ucierpiał gdzie indziej.

Dlatego hasła **haszuje się jednokierunkowo (bcrypt) z solą**: nawet po wycieku
bazy napastnik dostaje tylko hashe, z których odtworzenie oryginalnych haseł jest
niepraktyczne, a sól psuje gotowe tablice (*rainbow tables*). Serwer i tak nie
potrzebuje znać hasła — do logowania wystarcza **porównanie hashy**.
