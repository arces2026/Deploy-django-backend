# Step-by-Step Flow: Deploy della Webapp (Didattica)

Questo documento descrive il flusso completo, passo dopo passo, per il deploy della vostra applicazione web (Frontend Vue e Backend Django + PostgreSQL), sfruttando l'integrazione con GitHub per la CI/CD.

L'architettura proposta separa il **Frontend statico**, hostato gratuitamente su Vercel, dal **Backend e Database**, deployati su Render.com utilizzando il codice sorgente direttamente (senza containerizzazione).

IMPORTANTE !! I due repository da linkare sono:

- Deploy-django-backend
- Deploy-vue-frontend

---

## Fase 1: Deploy Automatico del Frontend (Vue) su Vercel
Vercel è la strada più rapida e indolore per Vue: si aggancia a GitHub e compie le operazioni di build e deploy in modo autonomo a ogni `git push`.

1. **Preparazione su GitHub:** Assicuratevi che il progetto Vue si trovi in una cartella specifica del repository (es. `/frontend`) o in un repository dedicato.
2. **Collegamento a Vercel:**
   * Accedete a Vercel utilizzando il vostro account GitHub.
   * Cliccate su **Add New Project** e importate il repository.
   * Vercel riconoscerà automaticamente il framework Vite/Vue e configurerà il comando di build.
3. **Variabili d'ambiente:** Andate nelle impostazioni del progetto su Vercel (`Settings` -> `Environment Variables`) e aggiungete l'URL di produzione del vostro backend (es. `VUE_APP_API_BASE_URL=https://vostro-backend.onrender.com`).
4. **Deploy Iniziale e CI/CD:** Cliccate su **Deploy**. Da questo momento in poi, ogni `git push` sul branch principale farà scattare il build e aggiornerà il frontend live automaticamente.

---

## Fase 2: Configurazione del Backend (Django) e Database (PostgreSQL) su Render

Render.com offre un piano gratuito per servizi web e database PostgreSQL, ideale per progetti didattici. A differenza della soluzione con container Docker, qui il deploy avviene direttamente dal codice sorgente.

### 2.1 Configurazione del Database PostgreSQL su Render

1. **Creare il Database:**
   * Accedete al vostro dashboard su [Render.com](https://render.com).
   * Cliccate su **New +** e selezionate **PostgreSQL**.
   * Inserite un nome per il database (es. `mydb`).
   * Selezionate il piano **Free** (limite 1GB di storage).
   * Cliccate su **Create Database**.

2. **Ottenere le credenziali:**
   * Dopo la creazione, Render vi fornirà automaticamente:
     - **Internal Database URL** (per connettervi dall'interno del servizio Render)
     - **External Database URL** (per connettervi dall'esterno, es. da DBeaver)
   * Queste URL contengono già tutte le credenziali (username, password, host, porta, nome del database).
   * Render imposta automaticamente la variabile d'ambiente `DATABASE_URL` per il servizio web collegato.

3. **Accedere al Database con DBeaver (o altri client SQL):**
   * Aprite DBeaver e create una nuova connessione **PostgreSQL**.
   * Inserite i seguenti parametri (ricavati dalla **External Database URL** fornita da Render):
     - **Host:** `dpg-xxxxxxxxxxxxxx-a.oregon-postgres.render.com` (esempio)
     - **Porta:** `5432` (di default)
     - **Database:** `mydb_xxxx`
     - **Username:** `mydb_xxxx_user`
     - **Password:** `la_password_fornita`
   * **Nota importante:** Per abilitare le connessioni esterne, andate nelle impostazioni del database su Render e sotto **Access Control** aggiungete l'indirizzo IP da cui vi connetterete (o usate `0.0.0.0/0` per abilitare l'accesso da qualsiasi IP, non raccomandato in produzione).
   * Cliccate su **Test Connection** per verificare e poi su **Finish**.

### 2.2 Deploy del Backend Django su Render

Il progetto Django è già configurato per gestire automaticamente la connessione al database in base all'ambiente:

- **Su Render:** utilizza la variabile d'ambiente `DATABASE_URL` per connettersi a PostgreSQL
- **In locale:** utilizza le variabili d'ambiente per connettersi a MySQL

1. **Preparazione del repository:**
   * Assicuratevi che nel vostro repository siano presenti:
     - `requirements.txt` con tutte le dipendenze (inclusi `psycopg2-binary` per PostgreSQL, `pymysql` per MySQL locale, `dj-database-url`, `gunicorn`, `whitenoise`)
     - `manage.py`
     - Il file `dump.json` per il caricamento dei dati iniziali
   * Il file `settings.py` è già configurato correttamente con:
     - La logica di connessione al database tramite `dj_database_url` quando `DATABASE_URL` è presente
     - `ALLOWED_HOSTS` che include già `deploy-django-backend.onrender.com`
     - `CORS_ALLOW_ORIGINS` configurato per il frontend Vercel
     - `DEBUG = True` (da impostare a `False` in produzione)
     - `WhiteNoise` per servire i file statici
     - `REST_FRAMEWORK` configurato con JWT authentication

2. **Collegamento a Render:**
   * Accedete a Render e cliccate su **New +** -> **Web Service**.
   * Selezionate **Build and deploy from a Git repository** e collegate il vostro repository GitHub.
   * Selezionate il repository corretto (Deploy-django-backend).

3. **Configurazione del Web Service:**
   * **Nome:** inserite un nome per il servizio (es. `deploy-django-backend`).
   * **Environment:** selezionate **Python 3**.
   * **Build Command:** 
     ```
     pip install -r requirements.txt ; python manage.py migrate ; python manage.py createsuperuser --noinput; python manage.py loaddata dump.json;
     ```
   * **Start Command:** 
     ```
     gunicorn bookshelf.wsgi:application --bind 0.0.0.0:$PORT
     ```
     (Nota: il nome del progetto è `bookshelf`, come da configurazione nel file `settings.py`)
   * **Plan:** selezionate **Free** (limite di 750 ore/mese).
   * **Environment Variables:** 
     - **Variabile automatica:** Render collegherà automaticamente il database PostgreSQL e imposterà la variabile `DATABASE_URL` quando il servizio web e il database sono nello stesso account. **Questo è il comportamento preferito**.
     - **Variabili manuali (se necessario):** Se il database è su un account diverso o se volete configurarlo manualmente, aggiungete:
       ```
       DATABASE_URL=postgresql://username:password@host:5432/database_name
       ```
       In alternativa, potete impostare le singole variabili:
       ```
       PGHOST=host
       PGPORT=5432
       PGDATABASE=nome_db
       PGUSER=username
       PGPASSWORD=password
       ```
     - **Variabili per il superuser (obbligatorie per il comando createsuperuser):**
       ```
       DJANGO_SUPERUSER_USERNAME=admin
       DJANGO_SUPERUSER_EMAIL=admin@example.com
       DJANGO_SUPERUSER_PASSWORD=password_sicura
       ```
     - **Altre variabili consigliate:**
       ```
       SECRET_KEY=una_chiave_criptografica_sicura_per_la_produzione
       DEBUG=False
       ```

4. **Deploy Iniziale:**
   * Cliccate su **Create Web Service**.
   * Render inizierà automaticamente il build e il deploy. Potete monitorare il progresso nei log.
   * Il Build Command eseguirà nell'ordine:
     1. Installazione delle dipendenze
     2. Migrazioni del database
     3. Creazione del superuser (se non esiste già)
     4. Caricamento dei dati dal file `dump.json`
   * Al termine, il backend sarà accessibile all'URL: `https://deploy-django-backend.onrender.com`

### 2.3 Automatizzazione CI/CD

Render offre un'integrazione nativa con GitHub: ogni `git push` sul branch principale triggererà automaticamente un nuovo deploy. Non è necessario configurare GitHub Actions.

1. **Workflow automatico:**
   - Ogni push sul repository triggera il rebuild del servizio su Render.
   - Il Build Command viene eseguito da capo, applicando le migrazioni e ricaricando i dati.
   - Il servizio viene riavviato automaticamente con il nuovo codice.

2. **Variabili d'ambiente su Render:**
   * Se dovete modificare le variabili d'ambiente, andate nelle impostazioni del vostro Web Service su Render (`Settings` -> `Environment Variables`).
   * Modificate o aggiungete nuove variabili e cliccate su **Save Changes**. Render riavvierà automaticamente il servizio.

3. **Note sul Build Command:**
   * Il comando è unico e combinato con `;` per eseguire tutte le operazioni necessarie al deploy.
   * `python manage.py createsuperuser --noinput`: crea il superuser utilizzando le variabili d'ambiente `DJANGO_SUPERUSER_USERNAME`, `DJANGO_SUPERUSER_EMAIL` e `DJANGO_SUPERUSER_PASSWORD`.
   * `python manage.py loaddata dump.json`: carica i dati dal file dump.json. Assicuratevi che il file esista nella root del progetto.
   * **Importante:** Poiché l'accesso alla shell su Render non è gratuito, questo comando combinato è essenziale per eseguire tutte le operazioni necessarie in un unico passaggio durante il deploy.

4. **Configurazione di produzione:**
   * Prima del deploy in produzione, modificate il file `settings.py` per:
     - Impostare `DEBUG = False`
     - Generare una nuova `SECRET_KEY` sicura (da inserire come variabile d'ambiente)
   * Render si aspetta che il comando `gunicorn` punti al file `wsgi.py` del progetto (`bookshelf.wsgi`).

---

## Fase 3: Collegamento Frontend-Backend

1. **Aggiornare le variabili d'ambiente su Vercel:**
   * Andate nelle impostazioni del progetto Vercel (`Settings` -> `Environment Variables`).
   * Impostate `VUE_APP_API_BASE_URL` con l'URL del backend su Render (es. `https://deploy-django-backend.onrender.com`).

2. **Verifica del funzionamento:**
   * Effettuate un push sul frontend per triggerare il deploy su Vercel.
   * Verificate che il frontend riesca a comunicare correttamente con il backend.

---

## Note Finali e Best Practices

- **Database:** Il database PostgreSQL di Render Free ha un limite di 1GB. Per progetti didattici è più che sufficiente.
- **Backend:** Il piano Free di Render mette a disposizione 750 ore di esecuzione al mese. Se il servizio rimane inattivo, Render lo sospende automaticamente e lo riattiva alla prossima richiesta (potrebbe richiedere alcuni secondi di cold start).
- **File Statici:** Il progetto utilizza WhiteNoise per servire i file statici in produzione, come configurato nel `settings.py`.
- **Variabili d'ambiente:** Non includere mai credenziali sensibili nel codice. Utilizzate sempre le variabili d'ambiente.
- **Dump dei dati:** Mantenete il file `dump.json` aggiornato per avere dati di test coerenti ad ogni deploy.
- **CORS:** Le impostazioni CORS nel `settings.py` sono già configurate per permettere al frontend su Vercel di comunicare con il backend.
- **JWT:** L'autenticazione JWT è già configurata con token di accesso della durata di 60 minuti e refresh token di 7 giorni.
- **PostgreSQL vs MySQL:** Il progetto supporta entrambi i database tramite la configurazione condizionale:
  - **In produzione (su Render):** Utilizza PostgreSQL tramite `dj_database_url`
  - **In sviluppo locale:** Utilizza MySQL tramite variabili d'ambiente