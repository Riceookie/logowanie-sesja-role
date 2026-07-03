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

## Trzy pojęcia w jednym zdaniu

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
