# 🔍 Sprawdzanie Kodu

Prosty system do kolejkowania i sprawdzania publicznych repozytoriów GitHub. Użytkownik loguje się, wysyła link do repozytorium, a osobny worker pobiera kod, uruchamia go w Dockerze i zapisuje wyniki.

---

## 🚀 Co robi aplikacja

- ✅ Rejestruje użytkowników
- 🔐 Loguje użytkowników i zwraca JWT
- 📎 Przyjmuje link do repozytorium GitHub
- 📋 Zapisuje test do kolejki w MySQL
- ⚙️ Worker pobiera najstarszy oczekujący test
- 📦 Klonuje repozytorium przez `git clone`
- 🐳 Uruchamia test w kontenerze Docker z Pythonem
- 📚 Instaluje `requirements.txt`, jeśli istnieje
- ✔️ Sprawdza składnię przez `python -m compileall .`
- 💾 Zapisuje kroki testu i wynik w bazie
- 💻 CLI pozwala logować się, rejestrować, dodawać repo i oglądać wyniki

---

## 📋 Wymagania

| Wymaganie | Wersja |
|-----------|--------|
| Python | 3.12+ |
| MySQL | `127.0.0.1:3306` |
| Database | `testy` |
| Git | ✓ |
| Docker Desktop | ✓ (musi być uruchomiony!) |

> ⚠️ **Ważne:** Docker Desktop musi być uruchomiony przed startem workera. Sama komenda `docker` jest tylko klientem, a worker potrzebuje działającego Docker Engine.

---

## 🔒 Bezpieczeństwo Dockera

Worker uruchamia cudzej kod w kontenerze Docker, a nie bezpośrednio na systemie hosta. Kontener jest odpalany z limitami:

```
--memory 512m          # Limit pamięci RAM
--cpus 1               # Limit CPU
--pids-limit 128       # Limit liczby procesów
--security-opt no-new-privileges  # Brak podnoszenia uprawnień
```

**⚠️ Ważne:** Etap `pip install -r requirements.txt` potrzebuje internetu, więc kontener podczas instalacji zależności ma dostęp do sieci. To oznacza, że złośliwe zależności nadal mogą wykonywać kod w tym momencie.

**❌ Nie dodawaj:**
```
--privileged
-v /var/run/docker.sock:/var/run/docker.sock
-v C:\:/host
```

**💡 Przyszłość:** Bezpieczniejszy wariant to rozdzielenie etapów: instalacja zależności z sieci, a późniejsze uruchamianie testów/kompilacji z `--network none`.

---

## ⚙️ Konfiguracja

### Plik `.env`

W katalogu projektu utwórz plik `.env`:

```env
SECRET_KEY=twoj_sekretny_klucz
```

### Połączenie z bazą danych

Aktualne połączenie z bazą jest ustawione w `app/baza.py`:

```
host:     127.0.0.1 / localhost
port:     3306
user:     root
password: root
database: testy
```

---

## 🗄️ Tabele w bazie danych

### `repo_tests` — główne testy repozytoriów

```sql
CREATE TABLE repo_tests (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    repo_url VARCHAR(500) NOT NULL,
    status ENUM('czeka','w_trakcie','gotowe') NOT NULL DEFAULT 'czeka',
    result ENUM('success','fail') NULL
);
```

### `repo_test_steps` — kroki testu

```sql
CREATE TABLE repo_test_steps (
    id INT AUTO_INCREMENT PRIMARY KEY,
    repo_test_id INT NOT NULL,
    step_order INT NOT NULL,
    name VARCHAR(100) NOT NULL,
    command TEXT NULL,
    status ENUM('czeka','w_trakcie','success','fail','skipped') NOT NULL DEFAULT 'czeka',
    exit_code INT NULL,
    stdout TEXT NULL,
    stderr TEXT NULL,
    FOREIGN KEY (repo_test_id) REFERENCES repo_tests(id) ON DELETE CASCADE
);
```

---

## 🏃 Uruchomienie

### 1️⃣ Backend (FastAPI)

Z głównego katalogu projektu:

```powershell
uvicorn main:app --reload
```

**Backend będzie dostępny pod:**
- API: `http://127.0.0.1:8000`
- Dokumentacja: `http://127.0.0.1:8000/docs` (Swagger UI)

### 2️⃣ Worker

Najpierw uruchom Docker Desktop i poczekaj, aż Docker Engine wystartuje.

Sprawdzenie statusu:
```powershell
docker info
```

Potem w osobnym terminalu:

```powershell
python -m app.testowanie
```

> 📌 Worker działa jako osobny proces. Backend tylko dodaje testy do kolejki, a worker sam pobiera rekordy ze statusem `czeka`.

---

## 🔌 API Endpointy

### 📝 Rejestracja

```http
POST /api-log/register
```

```json
{
  "mail": "user@example.com",
  "haslo": "haslo",
  "powtorzone": "haslo"
}
```

### 🔑 Logowanie

```http
POST /api-log/login
```

```json
{
  "mail": "user@example.com",
  "haslo": "haslo"
}
```

**Odpowiedź zawiera token JWT:**

```json
{
  "status": "ok",
  "details": "logowanie przebiegło pomyślnie",
  "token": "eyJ..."
}
```

### 📦 Dodanie repozytorium do kolejki

```http
POST /api-operacje/sprawdz
```

**Header:**
```http
Authorization: Bearer <JWT>
```

**Body:**
```json
{
  "link": "https://github.com/user/repo"
}
```

### 📊 Lista testów użytkownika

```http
GET /api-operacje/wyciagnij
```

**Header:**
```http
Authorization: Bearer <JWT>
```

Zwraca ostatnie testy użytkownika.

### 🔎 Szczegóły testu

```http
GET /api-operacje/szczegoly?id=<ID_TESTU>
```

**Header:**
```http
Authorization: Bearer <JWT>
```

Zwraca kroki danego testu (jeśli test należy do zalogowanego użytkownika).

---

## 💻 CLI — Frontend

Frontend CMD znajduje się w katalogu `front`.

### Rejestracja
```powershell
python front/main.py -r
```

### Logowanie
```powershell
python front/main.py -l
```

Token jest zapisywany lokalnie w:
```
front/token.json
```

> 🔒 Ten plik jest ignorowany przez Git.

### Dodanie repozytorium do testu
```powershell
python front/main.py -t https://github.com/user/repo
```

### Lista wyników
```powershell
python front/main.py -w
```

CLI pobiera listę testów, pokazuje je jako numerowaną listę, a potem pozwala wybrać test do wyświetlenia szczegółów.

---

## ⚙️ Jak działa worker

Worker wykonuje uproszczony test Python-only:

1. 📥 Pobiera najstarszy rekord z `repo_tests`, gdzie `status = 'czeka'`
2. ⏱️ Ustawia status głównego testu na `w_trakcie`
3. 📂 Klonuje repozytorium przez `git clone`
4. 📝 Zapisuje krok `pobranie projektu`
5. 🐳 Odpala Docker:
   ```
   python:3.12-slim
   ```
6. 🔧 W kontenerze z limitami wykonuje:
   ```sh
   if [ -f requirements.txt ]; then pip install -r requirements.txt; fi && python -m compileall .
   ```
7. 💾 Zapisuje wynik kroku
8. ✅ Ustawia `repo_tests.status = 'gotowe'`
9. 📊 Ustawia `repo_tests.result = 'success'` albo `fail`
10. 🗑️ Usuwa katalog tymczasowy

---

## 🧪 Przykładowy test

Możesz sprawdzić prostym repozytorium lub cistem:

```powershell
python front/main.py -t https://gist.github.com/hbisneto/42349b9d709387e90c93dfeee4a105e1.git
```

Potem wyświetl wyniki:

```powershell
python front/main.py -w
```

---

## 🐛 Typowe problemy

### 🚫 Docker nie działa

Jeśli widzisz:
```
failed to connect to the docker API
```

Uruchom Docker Desktop i sprawdź:

```powershell
docker info
```

### 📁 Błąd usuwania folderu `.git`

Na Windowsie pliki z `.git` mogą mieć atrybut readonly albo być chwilowo trzymane przez proces. Worker używa sprzątania z `chmod`, ale jeśli problem wróci, sprawdź czy repo nie jest otwarte w innym narzędziu.

### 🔐 Token nie działa

Token wysyła się w headerze:

```http
Authorization: Bearer <JWT>
```

Jeśli CLI nie znajduje tokena, zaloguj się ponownie:

```powershell
python front/main.py -l
```