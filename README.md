# checker_rldc
# Wykonawcze streszczenie

Celem zaproponowanego narzędzia jest **monitoring i analiza** działania bota tradingowego RLdC_AiNalyzator. System ten będzie składać się z dwóch głównych modułów: 1) **monitoringu zdarzeń run-time**, który przechwytuje wszystkie operacje handlowe (zlecenia, transakcje) oraz wyjściowe sygnały i analizy rynku, a następnie poddaje je ocenie przez model AI (np. lokalny Ollama) w celu generowania zaleceń optymalizacyjnych; 2) **statycznej analizy kodu**, skanującej resztę repozytorium w poszukiwaniu błędów, niespójności i luk w testach. Narzędzie będzie oferować zarówno interfejs wiersza poleceń (CLI), jak i (opcjonalnie) prosty interfejs API. Przewidziano warianty implementacji w **Pythonie** i **Node.js**, ze szczególnym uwzględnieniem bibliotek AI dostępnych w obu środowiskach. Proponowany format logów i raportów to struktury JSON przechowujące szczegóły zdarzeń oraz rekomendacje. Integracja z lokalnym Ollama (przez jego REST API) umożliwi generowanie tekstu diagnostycznego, z możliwością awaryjnego przełączenia na Llama.cpp lub zdalne API OpenAI. Narzędzie będzie rejestrować metryki (np. liczba wykrytych anomalii, czas analizy, skuteczność rekomendacji) i generować raporty zbiorcze. Plan implementacji obejmuje krok po kroku dodanie modułów ekstraktora zdarzeń, normalizatora, klienta AI, generatora rekomendacji, loggera i analizatora statycznego wraz z testami jednostkowymi. Przedstawiono przybliżony harmonogram prac i porównawczą tabelę zalet implementacji w Pythonie i Node.js. 

## Analiza repozytorium RLdC_AiNalyzator

Repozytorium **RLdC_AiNalyzator** jest botem tradingowym z backendem FastAPI (Python 3.11+) i frontendem Next.js (Node.js 20.9+)【6†L43-L47】. Backend zawiera m.in. **`backend/collector.py`** – główną pętlę handlu i zbierania danych, oraz **`backend/analysis.py`** – moduł analizy technicznej i generowania sygnałów/insightów. Operacje handlowe są realizowane poprzez tabele bazy danych: m.in. *Orders* (zlecenia) oraz *PendingOrders* dla oczekujących potwierdzeń. Klasy `Order` i `PendingOrder` definiują pola zleceń (symbol, typ, ilość, status, tryb demo/live, parametry wykonania)【49†L106-L115】. Każda transakcja jest zapisywana również w tabeli *DecisionTrace* z pełnym kontekstem decyzji. Analizy rynkowe są zapisywane w tabeli *signals* (kolumny: symbol, typ sygnału [BUY/SELL/HOLD], pewność, cena, wskaźniki, uzasadnienie)【17†L92-L100】. System loguje zdarzenia w tabeli *system_logs* (poziom, moduł, wiadomość, wyjątek)【49†L237-L246】. Punktem wejścia jest `backend/app.py`, który uruchamia FastAPI i kolektor. 

Kluczowe moduły związane z handlem i analizą to:
- **collector.py** – realizuje pętlę zbierania danych rynkowych (tickery, świece) i wykonywania zleceń (funkcje `_demo_trading`, `_execute_confirmed_pending_orders`), zapisując wszystko w DB i do *SystemLog* przy pomocy `log_to_db`【25†L9-L17】.  
- **analysis.py** – generuje insighty rynkowe i sygnały AI; funkcja `persist_insights_as_signals()` tworzy obiekty `Signal` w DB na podstawie listy insightów【22†L1425-L1434】. W module są też funkcje obliczające wskaźniki techniczne (RSI, EMA itd.) i wywołujące modele językowe (OpenAI/Gemini/Groq/Ollama) do generacji rekomendacji.  
- **Risk, accounting, trade_exit** – moduły oceniające ryzyko i obliczające PnL pozycje.  
- **Deklaracje modeli DB** – znajdziemy w `backend/database.py`, np. klasy `Signal`, `Order`, `Position` i `SystemLog` (jak wyżej)【17†L92-L100】【49†L106-L115】.  

Istnieją już szczegółowe **logi systemowe** – każde istotne zdarzenie (np. `log_to_db("INFO","live_trading",...)`) zapisuje w tabeli `system_logs`【49†L237-L246】. W kodzie są też setki testów typu smoke i integracyjnych (w `tests/`). Format logów (tekst w `SystemLog.message`) jest nieustrukturyzowany, więc narzędzie będzie potrzebować normalizacji tych informacji. 

## Architektura narzędzia

Narzędzie podzielono na dwa bloki funkcjonalne: **Monitor Runtime** i **Analiza Statyczna**.

- **Monitor Runtime**: nasłuchuje zdarzeń generowanych przez bota tradingowego.  
  - *Ekstraktor Zdarzeń*: subskrybuje operacje bota (nowe zlecenia, wykonane transakcje, wygenerowane sygnały). Może to być np. moduł czytający nowe wpisy z tabel DB (*orders*, *signals*, *decision_traces*) lub hook do funkcji `log_to_db`.  
  - *Normalizator Danych*: przekształca surowe zdarzenia na ujednolicone struktury (np. JSON: typu “trade” lub “signal”, z polami symbol, cena, ilość, uzasadnienie, itp.).  
  - *Integracja AI*: moduł komunikujący się z lokalnym serwerem Ollama (np. przez jego REST API na `http://localhost:11434/api`) lub biblioteką klienta (Python/JS). Przyjmuje pojedyncze zdarzenie, formułuje prompt i wysyła do modelu.  
  - *Generator Rekomendacji*: parsuje odpowiedź LLM i formatuje zalecenia (np. lista punktów do poprawy) dla danego zdarzenia.  
  - *Logger i Raport*: zapisuje szczegółowy log działania narzędzia (gdzie i jakie zdarzenie, treść prompta/odpowiedzi) oraz generuje raporty PDF/CSV/JSON zbiorcze.  

- **Analiza Statyczna Kod**: okresowo skanuje cały kod repozytorium.  
  - *Analizator Składniowy*: wykorzystuje np. narzędzia lintujące (Flake8, Pylint) i analizatory statyczne (Bandit/Semgrep) do wykrycia błędów, antywzorów, braków w testach (np. brak `pytest` przy kluczowych funkcjach).  
  - *Analizator Architektury*: np. sprawdza zgodność struktury (np. plik `Dockerfile` nie istnieje, brak migracji Bazy danych) z rekomendacjami, korzysta z komentarzy dokumentacji w `docs/`.  
  - *Generator Raportów kodu*: wypisuje znalezione problemy w formie raportu technicznego (plik tekstowy lub HTML).  

Interfejs:
- **CLI**: pozwala uruchomić analizy (np. `rldc_monitor run --mode=monitor` i `rldc_monitor run --mode=codecheck`). 
- **API (opcjonalne)**: proste HTTP API do zlecaniu ręcznym sprawdzenia kodu lub przeglądania najnowszych rekomendacji, np. GET `/api/reports` zwracające JSON z wnioskami.

Wymagania środowiskowe: Python 3.11+ (np. FastAPI, SQLAlchemy, ollama-python) lub Node.js 20+ (Express/Koa, sequelize, ollama-js) – planujemy dwie ścieżki implementacji. Repo wymaga też bazy SQLite i kluczy Binance (jak w oryginale). 

```mermaid
graph LR
    TS[System tradingowy] --> EE[Ekstraktor Zdarzeń]
    EE --> DN[Normalizator Danych]
    DN --> AI[Moduł AI (Ollama/Llama/OpenAI)]
    AI --> RG[Generator Rekomendacji]
    RG --> LG[Moduł Logowania i Raportowania]
    Repo[Kod źródłowy] --> SA[Statyczny Analizator Kod]
    SA --> RPT[Raport o błędach i zaleceniach]
```

## Schematy danych i format logów

Dla ujednolicenia logów i raportów zalecamy stosować format **JSON**. Przykładowe rekordy:

- **Zdarzenie handlowe (trade)**:
  ```json
  {
    "type": "trade",
    "symbol": "BTCUSDT",
    "side": "BUY",
    "quantity": 0.5,
    "executed_price": 30000.0,
    "fee": 0.0001,
    "gross_pnl": 50.0,
    "net_pnl": 45.0,
    "timestamp": "2026-05-23T20:00:00Z",
    "decision_context": {
        "reason_code": "rsi_oversold",
        "config_snapshot_id": "abc123",
        "signal_summary": {...}
    }
  }
  ```
- **Zdarzenie analizy (signal)**:
  ```json
  {
    "type": "signal",
    "symbol": "ETHUSDT",
    "signal_type": "SELL",
    "confidence": 0.85,
    "price": 2000.0,
    "indicators": {"rsi14": 75.3, "ema20": 1980.5},
    "reason": "RSI > 70",
    "timestamp": "2026-05-23T20:05:00Z"
  }
  ```
- **Rekomendacja (od AI)**:
  ```json
  {
    "event_type": "trade",
    "symbol": "BTCUSDT",
    "side": "BUY",
    "timestamp": "2026-05-23T20:00:00Z",
    "recommendations": [
      "Zwiększyć stop-loss, bo wzrosty są krótkotrwałe",
      "Sprawdzić ostatnią godzinę - wysoka zmienność może dawać fałszywe sygnały"
    ]
  }
  ```

Logi narzędzia mogą być rejestrowane do własnych plików lub tabeli (np. `monitor_logs`). Raporty zbiorcze mogą być formatu JSON/CSV/HTML zawierające listę wszystkich wykrytych anomalii i rekomendacji w danym okresie.

## Plan implementacji

1. **Inicjalizacja projektu**: utworzenie struktury `monitoring/` w repo, utworzenie `requirements.txt`/`package.json` dla zależności (m.in. `ollama-python` lub `ollama-js`, `sqlalchemy`/`sequelize`, `pandas`, lint).  
2. **Ekstrakcja zdarzeń**:  
   - *Wariant 1 (integracja)*: zmodyfikować `backend` – np. rozszerzyć funkcję `log_to_db` tak, aby wysyłała kopię logów do naszego modułu (np. HTTP POST do lokalnego API narzędzia) albo dodanie nowego event hooka po dodaniu `Order`/`Signal` w DB.  
   - *Wariant 2 (oddzielny proces)*: utworzyć osobny skrypt, który w pętli monitoruje plik DB lub podlicza nowe wiersze w tabelach `orders`, `signals`, `decision_traces` (np. przez SQL `SELECT ... WHERE timestamp > ostatni_time` co minutę).  
3. **Normalizator danych**: zaimplementować funkcje przekształcające pobrane wiersze DB lub logi do ujednoliconego formatu JSON (jak wyżej).  
4. **Moduł AI**:  
   - Zaprojektować klasę `AIClient` korzystającą z `ollama.Client` (Python) lub `new Ollama()` (JavaScript) do wysyłania promptów. Np. w Pythonie: 
     ```python
     from ollama import Client
     ollama = Client()
     response = ollama.generate(model='llama3.2', prompt=prompt_text)
     ```
     Albo w Node: 
     ```js
     import ollama from 'ollama';
     const res = await ollama.chat({model:'llama3.1', messages:[{role:'user', content: prompt_text}]});
     ```  
   - Definiować szablony promptów (w języku polskim) np. `"Przeanalizuj następującą transakcję: ... i podaj zalecenia optymalizacyjne."`.  
5. **Generator rekomendacji**: parsować odpowiedź AI (np. lista punktów) i tworzyć czytelny format JSON.  
6. **Logger i raporty**: implementacja zapisu logów (np. do pliku lub tabeli `monitor_logs`) i agregacji wyników (wyeksportowanie do pliku `json/markdown`).  
7. **Analiza statyczna**:  
   - Konfiguracja narzędzi: Flake8/Pylint (Python) lub ESLint/TypeScript (Node) do wykrywania błędów składniowych i anti-patternów.  
   - Własne kontrole: skrypt uruchamiający pytest (lub `npm test`) w celu sprawdzenia pokrycia testami, ewentualnie `pytest-cov`.  
   - Automatyczna analiza: uruchomienie Bandit/Semgrep na wrażliwe fragmenty (np. klucze API w kodzie).  
   - Wykrywanie braków: np. sprawdzenie obecności `docker-compose.yml` czy migracji (Alembic).  
8. **Testy jednostkowe i integracyjne**: stworzyć testy symulujące przykładowe zdarzenia:  
   - Podanie sztucznego sygnału do AIClient, weryfikacja formatu rekomendacji.  
   - Sprawdzenie ekstraktora (np. test DB).  
   - Analizator statyczny: np. wczytać fragment błędnego kodu i oczekiwać wykrycia błędu przez Pylint.  

Tabela przykładowych plików:

| Plik                        | Zmiana                     | Opis                                                   |
|-----------------------------|---------------------------|--------------------------------------------------------|
| `monitoring/event_extractor.py`  | **+ nowy plik**              | Logika wyłapywania zdarzeń z DB lub logów bota.        |
| `monitoring/data_normalizer.py`  | **+ nowy plik**              | Ujednolicenie i formatowanie pobranych danych.         |
| `monitoring/ai_client.py`        | **+ nowy plik**              | Integracja z Ollama (Python) lub `monitoring/index.js` dla Node. |
| `monitoring/recommender.py`      | **+ nowy plik**              | Przetwarzanie odpowiedzi AI i generowanie rekomendacji.|
| `monitoring/logger.py`           | **+ nowy plik**              | Zapis logów zdarzeń narzędzia i raportów.             |
| `monitoring/static_analyzer.py`  | **+ nowy plik**              | Skrypty uruchamiające lint/testy i analizę kodu.       |
| `monitoring/cli.py`             | **+ nowy plik**              | Interfejs CLI (np. użycie Click / Commander).          |
| `backend/system_logger.py`       | **modyfikacja**             | (Opcjonalnie) dodanie hooka przesyłającego logi.      |
| `backend/collector.py`          | **modyfikacja** (opcja)      | Wstrzyknięcie wywołań monitoringu po wykonaniu zleceń. |

Przykładowy kod (Python) integracji z Ollama w `ai_client.py`:
```python
from ollama import Client
class OllamaClient:
    def __init__(self, model='llama3.2'):
        self.client = Client()
        self.model = model
    def query(self, prompt: str) -> str:
        resp = self.client.generate(model=self.model, prompt=prompt)
        return resp['outputs'][0]  # tekst odpowiedzi
```
Analogiczny (JS) w `monitoring/index.js`:
```js
import ollama from 'ollama';
export async function askOllama(model, prompt) {
    const res = await ollama.chat({model, messages:[{role:'user', content: prompt}]});
    return res.message.content;
}
```

## Integracja z lokalnym Ollama i alternatywy

Narzędzie zakłada obecność **lokalnego serwera Ollama** (np. na `localhost:11434`), który hostuje model LLM. Ollama dostarcza oficjalne biblioteki **Python** (`ollama-client`) oraz **JavaScript**【27†L112-L120】. Przykładowo w Node.js używa się:
```js
import ollama from 'ollama';
const response = await ollama.chat({ model: 'llama3.1', messages: [{ role: 'user', content: prompt }] });
```
co pokazuje oficjalne przykłady użycia【33†L289-L297】. W Pythonie podobnie korzystamy z `ollama.Client()` (dokumentacja [27†L112-L120]). 

**Format promptów:** generujemy szczegółowe instrukcje w języku polskim zawierające opis transakcji lub sygnału oraz pytanie o analizę. Przykład promptu dla transakcji: 
```
"Przeanalizuj wykonany zakup {symbol} o cenie {price} i ilości {qty}. Oblicz korzyści i koszty oraz zasugeruj możliwe poprawki strategii handlowej."
```
Ważne jest zabezpieczenie: dostęp do Ollama odbywa się lokalnie (brak kluczy API), a prompty nie wysyłają poufnych danych poza lokalny serwer. W razie braku lokalnego środowiska można użyć **Llama.cpp** jako alternatywy – jest to lekki silnik C/C++ do uruchamiania modeli GGUF lokalnie, bez potrzeby Pythona czy GPU【37†L144-L153】. Inną opcją jest zdalne API **OpenAI** (np. modele GPT), jednak wiąże się z przekazywaniem danych poza system i dodatkowymi kosztami. Dzięki bibliotekom kompatybilnym z OpenAI można łatwo przełączyć prompty na `openai.ChatCompletion` w Pythonie. 

W przypadku braku komunikacji z Ollama (błąd, brak sieci) narzędzie powinno logować ostrzeżenie i pracować w trybie awaryjnym – np. generować rekomendacje metodami heurystycznymi lub ograniczyć się do zbierania logów bez AI. 

## Metryki i KPI

Monitorowane metryki:

- **Liczba wykrytych anomalii**: np. liczba nietypowych transakcji (np. poniżej minimalnej wartości) czy sprzecznych sygnałów.  
- **Czas analizy**: średni czas od przyjęcia zdarzenia do wygenerowania rekomendacji (warto optymalizować czasową responsywność).  
- **Dokładność rekomendacji**: udział rekomendacji uznanych za użyteczne (np. przez operatora) – może być oceniany subiektywnie lub testowo w środowisku symulacyjnym.  
- **Pokrycie testami**: procent pokrycia kodu testami (dla statycznej analizy).  
- **Błędy statyczne**: liczba błędów kodu/testów wykrytych przy każdej analizie kodu.  

**Format raportów**: np. tygodniowe/ miesięczne raporty PDF/HTML zawierające podsumowanie zdarzeń (tabela "data, symbol, typ, rekomendacja"), wykresy (np. słupkowe liczba transakcji na dzień, liczbę zaleceń), oraz listy kluczowych wniosków (doładowanie kapitału, zmiana parametrów strategii, naprawa kodu). Można też generować API JSON z bieżącymi statystykami, które frontend/metriky mogą wyświetlać. 

## Harmonogram i estymacja czasu

Prace podzielono na etapy MVP (podstawowa funkcjonalność) i wersję produkcyjną (pełna automatyzacja, interfejsy). Oto przykładowy rozkład czasów:

| Etap                            | Zadania                                                  | Czas (h) |
|---------------------------------|----------------------------------------------------------|---------:|
| **1. Przygotowanie środowiska** | Analiza repo, konfiguracja projektu, instalacja Ollama    |       4 |
| **2. Ekstrakcja zdarzeń**       | Implementacja czytania DB albo hooków logów                |       8 |
| **3. Normalizacja danych**      | Mapowanie zdarzeń do JSON                                   |       6 |
| **4. Moduł AI (MVP)**           | Integracja z Ollama (wywołanie API, prosty prompt)         |       8 |
| **5. Generator rekomendacji**   | Parsowanie odpowiedzi, formatowanie JSON                   |       6 |
| **6. Logger i raporty (MVP)**   | Logowanie zdarzeń i generowanie przykładowego raportu      |       6 |
| **7. Statyczny analizator**     | Konfiguracja lint, testów, generowanie raportu             |       8 |
| **8. Testy i dokumentacja**     | Testy jednostkowe/integracyjne, instrukcja obsługi         |       6 |
| **_Razem MVP_**                 |                                                          |      44 |
| **9. Optymalizacje**            | Rozbudowa promptów, wydajność, obsługa błędów AI           |       6 |
| **10. Interfejsy**              | CLI, API dla wyników, interfejs web (opcjonalnie)          |       8 |
| **11. Dodatkowe funkcje**       | Heurystyka fallback, integracja z Jenkins/cron            |       6 |
| **12. Końcowe testy i wdrożenie**| Pełny testing end-to-end, uwagi bezpieczeństwa            |       6 |
| **_Razem produkcja_**           |                                                          |      20 |
| **_SUMA całkowita_**            |                                                          |      64 |

Łącznie około **64 godziny** pracy dla MVP i pełnej wersji. Szacunki uwzględniają czas researchu, testów i dokumentacji.

## Porównanie Python vs Node.js

| Kryterium                      | Python                          | Node.js                         |
|--------------------------------|---------------------------------|---------------------------------|
| **Integracja z repozytorium**  | Kod bota już w Pythonie (FastAPI) – łatwy dostęp do tych samych obiektów DB i procedur【49†L106-L115】【22†L1425-L1434】. | Wymaga komunikacji międzyprocesowej (np. API lub socket) do bota w Pythonie.  |
| **Biblioteki AI**              | `ollama-python` oficjalne, bogaty ekosystem ML/Pandas. Łatwe przetwarzanie danych. | `ollama-js` oficjalne dla Node/JS【33†L289-L297】. Dużo narzędzi webowych (np. React) oraz event-driven. |
| **Wydajność**                  | Python może być wolniejszy w przetwarzaniu dużych ilości zdarzeń (Global Interpreter Lock), ale do asynchronicznego IO (bazodanego/HTTP) jest OK. | Node.js jest zoptymalizowany pod asynchroniczne operacje I/O, może obsługiwać wiele połączeń jednocześnie. |
| **Utrzymanie**                 | Dla zespołu znającego już backend (Python). Łatwiejsza integracja z istniejącymi modelami i testami. | Jeśli zespół pracuje z JS, to zaleta. Wymaga koordynacji dwóch środowisk (Python bot + Node monitor). |
| **Bezpieczeństwo**             | Przez SSH lub CI – ew. biblioteki bezpieczeństwa Pythona. Łatwiejsza izolacja sandbox. | Node.js może mieć więcej podatności w pakietach NPM (uwaga na narzędzia do AI). Trzeba zabezpieczyć API (cors, uwierzytelnienie). |

Źródła potwierdzają, że zarówno Python, jak i Node mają oficjalne klienty Ollama【27†L112-L120】【33†L289-L297】. W projekcie, który już używa Pythona, naturalnym wyborem jest rozszerzenie go o narzędzie monitoringowe. Alternatywnie Node.js może być użyty, ale wymaga interfejsu komunikacyjnego (np. REST) z botem.

## Podsumowanie

Proponowane narzędzie kompleksowo uzupełnia funkcjonalność RLdC_AiNalyzator o mechanizmy monitoringu i samooceny. Moduły do wyłapywania zdarzeń handlowych oraz AI dostarczą operatorowi przejrzyste rekomendacje optymalizacyjne, a analiza statyczna poprawi jakość samego kodu. Architektura modułowa umożliwia skalowanie i dalszy rozwój (np. obsługę kolejnych LLM). Warianty Python i Node zostały porównane pod kątem integracji i zasobów dostępnych bibliotek AI. Szczegółowy plan wdrożenia oraz harmonogram zapewnią terminowe wykonanie MVP i późniejszego wdrożenia. 

**Źródła:** dokumentacja repozytorium RLdC_AiNalyzator (modele DB, schematy tabel)【17†L92-L100】【49†L106-L115】, instrukcje Ollama (Python/JS API)【27†L112-L120】【33†L289-L297】, dokumentacja llama.cpp dla lokalnego uruchamiania modeli【37†L144-L153】.
