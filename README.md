# Andron - a complementary currency platform

Andron is a **complementary-currency system for local communities**. Citizens, businesses and charitable organisations (ETS, "enti del terzo settore") exchange services, products and community needs paid in *andron*, a local token (1 andron is tied to an hour of service through a configurable ratio). Trusted *guarantor* entities (the *garante*) approve accounts, services, needs and products and handle reports.

The repository contains five loosely coupled pieces: a citizen/business mobile app, a guarantor back-office, a Flask REST API backed by MySQL, a Hyperledger Fabric chaincode with a Node.js REST bridge, and the Fabric test network used to run it.

> **Important - current wiring.** In the committed code the Flask API keeps balances in MySQL (table `Saldo`, `Transazioni`) and does **not** call the blockchain: every `AndronCall.*` call in `Server/server.py` is commented out (marked "to remove in the blockchain version"). The chaincode, the bridge and the Python client (`Server/blockchainRequest.py`) are complete components that can be run on their own, but they are not on the request path of the apps.

## Contents

1. [Architecture](#architecture)
2. [Requirements](#requirements)
3. [Installation and first run](#installation-and-first-run)
4. [Configuration](#configuration)
5. [API reference](#api-reference)
6. [End-to-end walkthrough](#end-to-end-walkthrough)
7. [Repository layout](#repository-layout)
8. [Testing and verification status](#testing-and-verification-status)
9. [Troubleshooting](#troubleshooting)
10. [Limitations and security notes](#limitations-and-security-notes)
11. [Changes made in this revision](#changes-made-in-this-revision)
12. [Extending the project](#extending-the-project)

## Architecture

```
  +-------------------+      +-----------------------+
  | App/  (Ionic 4/5, |      | Amministrazione/      |
  | Angular 11)       |      | (Ionic 5, Angular 11) |
  | citizens, ETS,    |      | guarantor back-office |
  | businesses        |      |                       |
  +---------+---------+      +-----------+-----------+
            |  HTTP + JWT (Bearer)       |  HTTP + JWT (role=garante)
            v                            v
  +--------------------------------------------------+      +-------------------------+
  | Server/  Flask API  (default http://localhost:5001)|---->| MySQL 8 (db "Andron")   |
  | login, registration, services, products, orders,  |      | accounts, catalogue,    |
  | reports, feedback, /garante/* area, e-mail (SMTP) |      | balances (Saldo)        |
  +----------------------+---------------------------+      +-------------------------+
                         | (optional, currently unwired)
                         | blockchainRequest.py -> HTTP :8080
                         v
  +--------------------------------------------------+
  | fabric/server  Express bridge (:8080, fabric-network)|
  +----------------------+---------------------------+
                         | Fabric gateway (channel "mychannel", chaincode "andron")
                         v
  +--------------------------------------------------+
  | Fabric test network (2 orgs, CA, CouchDB, raft)   |
  | chaincode/andron : wallet | transaction | deposit |
  |                    settings | andronHistory       |
  +--------------------------------------------------+
```

Data flow of the working path: app -> `POST /login` -> JWT -> app calls the REST endpoints with `Authorization: Bearer <token>` -> Flask reads/writes MySQL. Guarantor flow: back-office -> `POST /garante/login` -> JWT carrying `role=garante` -> `/garante/*` endpoints (approve accounts/content, settings).

| Directory | Stack | Role |
|---|---|---|
| `App/` | Ionic / Angular 11 / Cordova | wallet, services, products/cart, orders, community services and needs, feedback, reports |
| `Amministrazione/` | Ionic 5 / Angular 11 | guarantor: activate users/businesses, create ETS and businesses, approve services/needs/products, handle reports, edit settings, map |
| `Server/` | Python / Flask / PyMySQL | REST API, e-mail templates (`TemplateEmail/`), SQL schema |
| `fabric/chaincode/andron/` | TypeScript chaincode (fabric-contract-api 2) | five contracts, see [Chaincode](#chaincode-contracts) |
| `fabric/server/` | Node.js / Express / fabric-network 2.2 | REST bridge to the chaincode |
| `fabric/test-network/`, `fabric/config/`, `fabric/installCC.sh` | Hyperledger Fabric samples | local network and one-shot deploy script |

Domain vocabulary (Italian names are used in the code): `utente` citizen, `impresa` business, `ente` ETS, `garante` guarantor, `servizio` service, `bene` product, `ordine` order, `richiesta` service request, `bisogno` need, `segnalazione` report, `saldo` balance.

## Requirements

| Component | Version |
|---|---|
| Python | 3.8+ (verified with 3.10) |
| Python packages | `Server/requirements.txt`: Flask 2.0-2.2 (the code uses `@app.before_first_request`, removed in Flask 2.3), Werkzeug < 3, Flask-JWT-Extended 4.0-4.5, flask-cors, PyMySQL, cryptography, requests |
| MySQL | 8.0 (the schema uses a `json` column and `utf8mb4_0900_ai_ci`; MariaDB is not compatible). Verified with `mysql:8.0` in Docker |
| Node.js (apps) | 14 or 16. Angular 11 builds fail on Node >= 17 (`ERR_PACKAGE_PATH_NOT_EXPORTED` for `postcss`). Verified with Node 16.20 |
| npm | 6-8 (use `--legacy-peer-deps`, the apps mix Angular 11 with Ionic 4/5 packages) |
| Ionic / Cordova | Ionic CLI 5/6, `cordova-android ^8` (App) for device builds - not verified here |
| Node.js (bridge, chaincode) | Fabric Node SDK 2.2 and `fabric-shim 2` support Node 12/14/16; chaincode `engines.node >= 12` |
| TypeScript | chaincode 3.9, apps ~4.0 |
| Hyperledger Fabric | 2.x binaries (`peer`, `configtxgen`, ...) and Docker images, installed into `fabric/bin` (not stored in the repo). Version used by the original authors is not recorded; use the 2.2/2.3 samples that match `fabric/test-network` |
| Docker | required by the test network |

## Installation and first run

Run the components in this order. Only steps 1-3 and the build of step 4 were verified in this environment; see [Testing](#testing-and-verification-status).

### 1. Database

```bash
mysql -u root -p -e 'CREATE DATABASE Andron CHARACTER SET utf8mb4'
mysql -u root -p Andron < Server/schema.sql
```

`schema.sql` contains the 20 tables and **no data**, so there is no administrator: create the guarantor and its settings yourself. The password is stored as `md5(password || salt)`:

```sql
INSERT INTO notAdministration (login, password, salt, JsonValue)
VALUES ('garante', md5(CONCAT('CHOOSE_A_PASSWORD', 'SOME_SALT')), 'SOME_SALT',
        '{"andron": 10, "percentuale": 5}');
```

`andron` is the number of andron granted per hour of service (and the initial balance of newly activated users); `percentuale` is the ETS reward percentage.

For a throwaway database with Docker:

```bash
docker run -d --name andron-mysql -e MYSQL_ROOT_PASSWORD=testpw -e MYSQL_DATABASE=Andron -p 127.0.0.1:13306:3306 mysql:8.0
docker exec -i andron-mysql mysql -uroot -ptestpw Andron < Server/schema.sql
```

### 2. Flask API

```bash
cd Server
python3 -m venv .venv && . .venv/bin/activate        # .venv is not tracked; keep it out of commits
pip install -r requirements.txt
export ANDRON_DB_HOST=127.0.0.1 ANDRON_DB_PORT=3306 ANDRON_DB_USER=root ANDRON_DB_PASSWORD=...   # see Configuration
python server.py                                       # http://localhost:5001
```

Run it from inside `Server/`: e-mail templates and the image store (`Server_File_Storage/`, created on the first request, git-ignored) are resolved relative to the working directory.

### 3. Fabric network, chaincode and bridge (optional - not on the request path)

```bash
# install Fabric binaries into fabric/bin first (see the Fabric docs), then:
cd fabric
./installCC.sh          # network down/up with CAs, create "mychannel", deploy chaincode "andron" (TypeScript),
                        # invoke settings:InitLedger, enrol admin and appUser into fabric/server/wallet
cd server
npm install             # node_modules is not tracked
PORT=8080 node main.js  # bridge on http://localhost:8080 (PORT defaults to 8080)
```

`installCC.sh` deploys the chaincode with `./network.sh deployCC -ccn andron -ccp ../chaincode/andron -ccl typescript`, so the chaincode is built by the Fabric Docker tooling. Wallet identities are generated locally and git-ignored. The bridge exits its identity check with a message if `appUser` is missing from `fabric/server/wallet`. **Not verified in this environment** (no Fabric binaries/images, no network run).

### 4. Mobile app and back-office

```bash
cd App                       # or: cd Amministrazione
npm ci --legacy-peer-deps    # Node 14/16
# set the API base URL (see Configuration), then:
npx ng serve                 # browser, http://localhost:4200 ; or: npx ionic serve
npx ng build                 # output in www/ (verified with Node 16)
# Android (App only), not verified here:
npx ionic cordova platform add android && npx ionic cordova run android
```

The CORS configuration of the API allows any origin, so the dev server can talk to the API directly. Cordova platforms/plugins and `www/` are not tracked and are regenerated by the commands above.

## Configuration

### Flask API (environment variables)

| Variable | Default | Meaning |
|---|---|---|
| `ANDRON_DB_HOST` | `127.0.0.1` | MySQL host |
| `ANDRON_DB_PORT` | `3306` | MySQL port |
| `ANDRON_DB_USER` | `root` | MySQL user |
| `ANDRON_DB_PASSWORD` | empty | MySQL password |
| `ANDRON_DB_NAME` | `Andron` | database name |
| `ANDRON_SMTP_USER` / `ANDRON_SMTP_PASSWORD` | empty | SMTP credentials for registration e-mails |
| `ANDRON_SMTP_HOST` / `ANDRON_SMTP_PORT` | `smtp.gmail.com` / `587` | SMTP server (STARTTLS) |
| `ANDRON_HOST` | `localhost` | bind address (use `0.0.0.0` so a phone on the LAN can reach it) |
| `ANDRON_PORT` | `5001` | listen port (matches the URLs hard-coded in the apps) |
| `ANDRON_DEBUG` | `0` | `1` enables the Flask debugger/reloader (never on an exposed host) |

Example `.env`-style file (the API does not load it by itself; `export` it or use `set -a; . ./andron.env; set +a`):

```
ANDRON_DB_HOST=127.0.0.1
ANDRON_DB_PORT=3306
ANDRON_DB_USER=andron
ANDRON_DB_PASSWORD=<db password>
ANDRON_DB_NAME=Andron
ANDRON_SMTP_USER=<mailbox>
ANDRON_SMTP_PASSWORD=<app password>
ANDRON_PORT=5001
```

Other settings are code constants: the JWT secret is `os.urandom(24)` generated at start-up (all tokens are invalidated by a restart); tokens never expire (`expires_delta=False`); the logout blacklist is in memory; `Server/blockchainRequest.py` targets `http://localhost:8080`.

### Bridge and apps

| Where | Setting | Default |
|---|---|---|
| `fabric/server/main.js` | env `PORT` | `8080` |
| `fabric/server/main.js` | connection profile | `fabric/test-network/organizations/peerOrganizations/org1.example.com/connection-org1.json` (generated by `network.sh up ... -ca`) |
| `fabric/server/main.js`, `routes/*.js` | wallet dir, identity, channel, chaincode | `fabric/server/wallet`, `appUser`, `mychannel`, `andron` |
| `fabric/server/src/enrollAdmin.*` | CA admin credentials | `admin` / `adminpw` (Fabric samples defaults - test network only) |
| `App/src/app/services/request.service.ts` | `url` | `http://192.168.1.53:5001` - **edit to your API host** |
| `Amministrazione/src/app/services/request.service.ts` | `url` | `http://localhost:5001/garante/` |
| `App/resources/android/xml/network_security_config.xml` | cleartext HTTP rules | allows cleartext (development); remove for production |
| `App/config.xml`, `Amministrazione/config.xml` | widget id | `com.example.myapp`, `io.ionic.starter` - change before publishing |

The `environment*.ts` files only hold the `production` flag.

## API reference

All Flask endpoints return JSON; successful list/detail calls wrap the payload as `{"_msg": ...}`. Unless noted, endpoints require `Authorization: Bearer <JWT>`. Missing/invalid token gives 401/422 (`flask-jwt-extended` message). Errors from handlers are mostly bare status codes: 400 (invalid input or business rule), 401, 404, 500 (database or internal error). Request-field validation (`Server/function.py`, `validRequest`) only checks that the listed keys are present and non-empty; the length/type hints in the table are not enforced.

### Flask API, citizen/business area

| Method and path | Auth | Description |
|---|---|---|
| `POST /login` | no | `{"username","password"}` -> `{"access_token"}`. 401 with `{"info":"0"}` wrong credentials, `"-1"` account not yet activated by the guarantor |
| `DELETE /logout` | yes | revokes the current token (in-memory blacklist) |
| `POST /register` | no | `_accountType` = `utente` or `impresa` plus profile fields (`_username,_password,_nome,_cognome,_date,_cf,_telefono,_mail,_stato,_regione,_citta,_via,_cap`; business: `_nome,_tipo,_iva,...`) and `_foto` (base64 JPEG). Account stays inactive until approved; a welcome e-mail is attempted |
| `GET, PUT /info` | yes | profile of the caller (user, business or ETS); `PUT` updates contact data (users only) |
| `PUT /info/changePassword` | yes | `{"_oldpsw","_newpsw"}` |
| `PUT /info/changeProfilePicture` | yes | `{"foto": base64}` |
| `GET /budget` | yes | `{"_msg":{"andron":N}}` |
| `GET /transaction?_page=&_psize=` | yes | paginated transaction history |
| `GET /transaction/<id>` | yes | one transaction (404 if missing) |
| `GET, POST /services/mine` | yes | list / create own services (`nome,descrizione,preavviso,ore`); new services need guarantor approval |
| `DELETE /services/mine/<id>` | yes | delete own service |
| `GET /services?_page=&_psize=&filtro=&termine=` | yes | browse approved services (`filtro=false` for no filter) |
| `GET, POST /services/<id>` | yes | service details / request it (`_msg,_data,_ore`) |
| `GET /requests/mine` | yes | requests I made |
| `DELETE, PUT, POST /requests/mine/<id>` | yes | cancel; change state (`_stato`); confirm with feedback (`_msg,_valutazione`) |
| `GET /services/mine/requests` | yes | requests received for my services |
| `POST /report` | yes | report a service or order (`id,_msg,utente,_tipo` = `servizio`/`ordine`) |
| `GET, POST /products/mine` | yes | business: list / create products (`nome,marca,descrizione,foto,prezzo,categoria,quantita`) |
| `DELETE, PUT /products/mine/<id>` | yes | delete / add stock (`quantita`) |
| `GET, POST /products` | yes | browse the catalogue (`_page,_psize,filtro,...`) / buy (cart JSON; stock and balance checked, rolled back on failure) |
| `GET /orders`, `PUT /orders/<id>` | yes | business: received orders / mark one as handled |
| `GET /orders/mine`, `PUT /orders/mine/<id>` | yes | my purchases / confirm receipt with feedback (`_a,_msg,_valutazione`) |
| `GET /feedback?_page=&_psize=&id=&tipo=` | yes | feedback about a user or business (`tipo` = `utente`/`impresa`) |
| `GET /services/community/ets` | yes | list of ETS |
| `GET, POST /services/community` | yes | community services (volunteering for an ETS): list / propose (`msg,date,bisognoSegnalato,idEnte,foto`) |
| `PUT, DELETE /services/community/<id>` | yes | ETS confirms (rewards the user) / refuses |
| `GET, POST /needs` | yes | community needs: list (`_all`) / report (`nome,descrizione,indirizzo,citta`) |

### Flask API, guarantor area (`/garante/*`)

`POST /garante/login` takes `{"login","password"}` and returns a token carrying the claim `role=garante`. All other `/garante/*` endpoints require that claim (otherwise 401).

| Method and path | Description |
|---|---|
| `GET, PUT /garante/settings` | read settings (`[{"Record":{"andronGeneration","rewardEts"}}]`); `PUT {"options":"andron","andron":N}` or `{"options":"percentuale","percentuale":N}` |
| `GET /garante/account/user` | users waiting for activation |
| `PUT, DELETE /garante/account/user?id=` | activate (creates the balance row with the configured andron amount) / refuse |
| `POST /garante/account/ets` | create an ETS (`_accountType:"ente"`, `_username,_password,_nome,_telefono,_email,_regione,_citta`), credentials e-mailed |
| `GET, PUT, DELETE /garante/account/impresa[?id=]`, `POST` | pending businesses / activate / refuse / create an already-active business |
| `GET, PUT, DELETE /garante/approve/services[?id=]` | approve or refuse services |
| `GET, PUT, DELETE /garante/approve/needs[?id=]` | approve or refuse needs |
| `GET, PUT, DELETE /garante/approve/products[?id=]` | approve or refuse products |
| `GET, PUT /garante/report/services` | service reports; `PUT` with `operazione` |
| `GET, PUT /garante/report/orders` | order reports; `PUT` with `operazione` = `concludi` or `rimborso` (refund) |

### Fabric bridge (`fabric/server`, port 8080)

| Method and path | Chaincode call |
|---|---|
| `POST /wallet` | `wallet:createWallet` body `{"walletID","user":{"nome","cognome","account"}}` (`account` = `utente`, `ets`, `impresa`) |
| `POST /wallet/reward` | `wallet:reward` body `{"user","ets","hour","service"}` |
| `POST /transaction` | `transaction:createTransaction` body `{"uuid","fromID","toID","amount","reason"}` |
| `POST /deposit` | `deposit:deposit` body `{"depositorID","allowedWithdrawer","amount","reasonDeposit","uuid"}` |
| `PUT /deposit/:id` | `deposit:withdrawal` body `{"withdrawer","reasonWithdraw"}` |
| `PUT /setting/ratio`, `/setting/reward`, `/setting/andron` | `settings:setRatio`, `setRewardEts`, `setAndronGeneration` (body `{ratio}`, `{reward}`, `{andron}`) |
| `GET /andronHistory/generated`, `/available`, `/deposited` | totals of andron generated / available / deposited |
| `GET /:asset`, `/:asset/:id`, `/:asset/history/:id` | list / read / key history for `asset` in `wallet, deposit, andronHistory, settings, transaction` |

Responses are the chaincode result `{"status":"success"|"error","message":...}` (HTTP 400 on `error`, 500 on SDK failure).

### Chaincode contracts

`fabric/chaincode/andron/src/index.ts` exports five contracts (invoked as `<contract>:<function>`): `wallet` (`createWallet`, `reward`), `transaction` (`createTransaction`), `deposit` (`deposit`, `withdrawal` - escrow of andron for a counterpart), `settings` (`InitLedger` with ratio 20, rewardEts 50, andronGeneration 20; `setRatio`, `setRewardEts`, `setAndronGeneration`), `andronHistory` (`getGeneratedAndron`, `getAvailableAndron`, `getDepositedAndron`). All inherit `get`, `getAll`, `getHistory` from `ContractExtension`. State keys are `<type>-<id>`; assets are validated with `yup`.

### Apps (pages)

`App/` routes (`app-routing.module.ts`, all but `login` behind `AuthGuard` that checks the stored token): `login`, `register`, `register-utente`, `register-impresa`, `home`, `edit-profile`, `wallet`, `descrizione-transazione`, `servizi`, `richiedi-servizi`, `richiesta`, `richieste-mie`, `richieste-miei-servizi`, `nuovo-servizio`, `compra`, `prodotti`, `descrizione-prodotto`, `nuovo-prodotto`, `cart-modal`, `ordini`, `servizi-collettivita`, `certifica-servizi`, `bisogni`, `segnala-bisogni`, `segnalazione`, `feedback`, `infofeedback`, `search-filter`, `popover`.
`Amministrazione/` routes: `login` (public) and, under `private/`: `home` (settings), `activate-user`, `activate-impresa`, `add-ente`, `add-impresa`, `service`, `product`, `need`, `service-management` (service reports), `order-management` (order reports), `map`. Each page maps onto the `/garante/*` endpoints above.

## End-to-end walkthrough

Assumes the throwaway MySQL above, the guarantor row from step 1, and the API on port 5001 (`ANDRON_PORT=5001`, the default). The outputs below were obtained against a real MySQL 8.0 and the Flask server (served on another port because 5000 was busy).

```bash
P=http://localhost:5001; J='Content-Type: application/json'

# 1. a citizen registers (inactive until approved)
curl -s -X POST $P/register -H "$J" -d '{"_accountType":"utente","_username":"mario.rossi","_password":"Passw0rd!","_nome":"Mario","_cognome":"Rossi","_date":"1990-05-01T00:00:00","_cf":"RSSMRA90E01H501X","_telefono":"3331234567","_mail":"mario.rossi@example.org","_stato":"Italia","_regione":"Sicilia","_citta":"Messina","_via":"Via Roma 1","_cap":"98100","_foto":"aGVsbG8="}'
# -> HTTP 200 (empty body)

# 2. login is refused until activation
curl -s -X POST $P/login -H "$J" -d '{"username":"mario.rossi","password":"Passw0rd!"}'      # {"info": "-1"}  (HTTP 401)

# 3. the guarantor logs in, lists and activates the user
G=$(curl -s -X POST $P/garante/login -H "$J" -d '{"login":"garante","password":"CHOOSE_A_PASSWORD"}' | python3 -c 'import sys,json;print(json.load(sys.stdin)["access_token"])')
curl -s -H "Authorization: Bearer $G" $P/garante/account/user        # [{"id": 47, "nome": "Mario", ...}]
curl -s -X PUT -H "Authorization: Bearer $G" "$P/garante/account/user?id=47"   # {"cognome": "Rossi", "nome": "Mario"}

# 4. the citizen logs in, checks the balance and publishes a service
U=$(curl -s -X POST $P/login -H "$J" -d '{"username":"mario.rossi","password":"Passw0rd!"}' | python3 -c 'import sys,json;print(json.load(sys.stdin)["access_token"])')
curl -s -H "Authorization: Bearer $U" $P/budget                      # {"_msg": {"andron": 10}}
curl -s -X POST -H "$J" -H "Authorization: Bearer $U" $P/services/mine -d '{"nome":"Ripetizioni","descrizione":"Aiuto compiti","preavviso":1,"ore":2}'   # {"_msg": true}

# 5. the guarantor sees the service waiting for approval; a normal user cannot use /garante/*
curl -s -H "Authorization: Bearer $G" $P/garante/approve/services    # [{"id": 26, "nome": "Ripetizioni", ...}]
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $U" $P/garante/account/user   # 401
```

(The user id `47` comes from the AUTO_INCREMENT value stored in the dump; use the id returned by step 3's list.) Registration e-mails need SMTP settings; without them the failure is logged and the API still answers 200 for self-registration. Bridge example (needs the Fabric network, not verified):

```bash
curl -s -X POST localhost:8080/wallet -H 'Content-Type: application/json' \
  -d '{"walletID":"u1","user":{"nome":"Mario","cognome":"Rossi","account":"utente"}}'
# {"status":"success","message":"Operazione effettuata"}
curl -s localhost:8080/wallet/u1
```

In the apps: open the back-office (`http://localhost:4200`), log in as the guarantor, approve the new user in "activate-user", then log into the citizen app with the user's credentials.

## Repository layout

```
Server/
  server.py              Flask app: routes, CORS, JWT (+ blacklist), isGarante() check
  db.py                  PyMySQL wrapper (DB) and all SQL (Query class)
  function.py            request validation tables, base64 image storage helpers
  mailSender.py          SMTP sender and e-mail template rendering
  blockchainRequest.py   HTTP client for the Fabric bridge (currently unused by server.py)
  schema.sql             MySQL schema (no data)
  requirements.txt       Python dependencies
  TemplateEmail/         plain-text e-mail templates (string.Template)
  test.py                manual script exercising the bridge (stale: uses an older createUser signature)
App/                     Ionic/Angular citizen & business app (src/app/pages/*, services/*, guards/)
Amministrazione/         Ionic/Angular guarantor back-office (src/app/private/*, public/login)
fabric/
  installCC.sh           bring the network up, deploy chaincode, init ledger, enrol identities
  chaincode/andron/      TypeScript chaincode (src/subcontracts/<contract>/...)
  server/                Express bridge: main.js, routes/*.js, src/{enrollAdmin,registerUser,invoke,query}
  test-network/, config/ Fabric samples (network.sh, docker-compose, cryptogen, configtx)
```

## Testing and verification status

Verified in this environment:

- Flask API: all modules compile (`python3 -I -m py_compile`); server started against MySQL 8.0 (Docker) loaded from `schema.sql`; exercised `/register`, `/login` (including the inactive-account response), `/garante/login`, `/garante/account/user`, `/budget`, `/info`, `/services/mine`, `/garante/approve/services`, `/garante/settings`, `/transaction`, `/logout` (token revoked afterwards) and the 401 for a non-guarantor token on `/garante/*`. The remaining endpoints were read but not called.
- Apps: `npm ci --legacy-peer-deps` and `ng build` succeed for `App/` and `Amministrazione/` with Node 16 (`tsc -p tsconfig.app.json` on `App/` only reports type errors inside `node_modules` packages).
- Bridge: `node --check` on all JS files; `main.js` starts and reports the missing wallet identity. Chaincode: `tsc --noEmit --types node --skipLibCheck` passes with TypeScript 3.9.

**Not verified in this environment:** the Fabric network, chaincode deployment and every bridge call that reaches it (`installCC.sh`, `/wallet`, `/transaction`, ...); Android/iOS builds and on-device behaviour (camera, geolocation, image picker); SMTP delivery; the unit/e2e suites (`ng test`, `ng e2e` - only Angular-generated specs exist). There are no automated tests for the API; `Server/test.py` is a manual script for the bridge.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `AttributeError: 'Flask' object has no attribute 'before_first_request'` | Flask >= 2.3 installed; use `pip install -r Server/requirements.txt` (Flask < 2.3) |
| `Port 5000 is in use` (macOS AirPlay or other service) | the API now defaults to 5001; set `ANDRON_PORT` |
| `/login` always 401 `{"info":"0"}` | wrong credentials, or the DB is empty; `{"info":"-1"}` means the account is waiting for the guarantor |
| API answers 500 on everything | MySQL not reachable (`DB.connetti` prints the error); check `ANDRON_DB_*`, including `ANDRON_DB_PORT` |
| `Access denied` / charset errors loading the schema | use MySQL 8.0, not MariaDB |
| `/garante/*` returns 401 | token is not from `/garante/login` (no `role=garante` claim), or the server was restarted (JWT secret changes at every start) |
| 422 `Bad Authorization header` | send `Authorization: Bearer <token>` |
| Apps cannot reach the API | set `url` in the two `request.service.ts`; bind the API with `ANDRON_HOST=0.0.0.0` for devices on the LAN; Android needs cleartext allowed (`network_security_config.xml`) |
| `ERR_PACKAGE_PATH_NOT_EXPORTED ... postcss` / `ERR_OSSL_EVP_UNSUPPORTED` | Node too new for Angular 11; use Node 14/16 |
| `npm install` ERESOLVE | add `--legacy-peer-deps` |
| `An identity for the user "appUser" does not exist in the wallet` | run `node enrollAdmin.js && node registerUser.js` from `fabric/server/src` after the network is up |
| Bridge returns 500 with no body | gateway connection failed: network down, wrong `connection-org1.json`, or chaincode not deployed |
| No registration e-mail | `ANDRON_SMTP_*` missing/invalid; the error is printed to the server log |

## Limitations and security notes

- The blockchain is not integrated into the working API path (see top note); balances live in MySQL.
- Passwords are stored as salted MD5 (`md5(password + salt)`), which is weak; replace with bcrypt/argon2 before any real use. The guarantor password is stored the same way.
- JWTs never expire; the revocation list and the signing secret are per-process (restart logs everybody out, multi-worker deployments will not share the blacklist).
- Before this revision `Query.isAdmin()` always returned `True`, so any authenticated user could call every `/garante/*` endpoint; it is now enforced through a `role=garante` token claim (see below).
- CORS allows any origin; `createResponse` also sets `Access-Control-Allow-Origin: *`. The CORS resource table contains typos (`originis`) for some `/garante` routes; these are harmless because the header is also set manually.
- The e-mails sent when the guarantor creates an ETS or business contain the plaintext initial password; `garanteRegisterEmail` always reports success even if the mail fails.
- The old SMTP password and TLS keys were committed in the past and must be treated as compromised; rotate them. Never commit `.env`, wallets, `*.pem`, `*.key` (git-ignored).
- `Server/test.py`, the apps and the bridge use plain HTTP; use TLS termination in front of anything public. The bridge has no authentication and trusts every caller; keep it on localhost.
- Flask is run with the development server; use gunicorn/uwsgi behind a proxy for deployment.
- Input validation is shallow (presence checks only); several endpoints (for example `POST /products` purchases) accept unvalidated payloads.
- The sample CA credentials (`admin/adminpw`) and the test network are for development only.

## Changes made in this revision

- `Server/server.py`: the `/garante/*` endpoints now check a `role=garante` JWT claim issued by `/garante/login` (previously `isAdmin()` returned `True` for everybody). Host, port and debug are configurable (`ANDRON_HOST`, `ANDRON_PORT`, `ANDRON_DEBUG`); the default port is now 5001 (the apps already pointed to 5001 while the server listened on 5000) and the Werkzeug debugger is off by default.
- `Server/db.py`: MySQL port configurable through `ANDRON_DB_PORT`.
- `Server/requirements.txt` added.
- `Amministrazione/src/app/services/request.service.ts`: removed leading spaces from the API base URL.
- `fabric/server/main.js`: the start-up identity check called the Express router named `wallet` instead of the Fabric wallet (`global.wallet`), so it never detected a missing identity; fixed. Port configurable through `PORT`.
- `fabric/installCC.sh`: replaced hard-coded `/Users/antoniopipitone/...` paths with relative ones.
- `fabric/server/node_modules` (2,340 files) was still tracked despite the earlier cleanup; removed from the index (it is git-ignored and regenerated by `npm install`).

## Extending the project

- **New REST endpoint**: add the SQL method to `Query` in `Server/db.py`, a route in `Server/server.py` (use `@jwt_required()`, and `isGarante()` for guarantor-only), a `validRequest` entry in `function.py`, and, if browsers call it cross-origin from a new path, a CORS entry.
- **New e-mail**: add a template to `Server/TemplateEmail/` and a function in `mailSender.py`.
- **Turn on the blockchain**: start the bridge, then un-comment the `AndronCall.*` calls in `server.py` (user/business/ETS creation on activation, rewards on community-service confirmation, transactions on purchases) and remove the corresponding MySQL balance updates in `db.py`; `blockchainRequest.py` already wraps the bridge.
- **New chaincode contract**: create `src/subcontracts/<name>/<name>Controller.ts` extending `ContractExtension` with `super("<name>")`, export and register it in `src/index.ts`, add a bridge route file and mount it in `fabric/server/main.js`, add the asset name to the `asset` list in `routes/getter.js` for generic reads; redeploy with `./network.sh deployCC -ccv <new version> -ccs <new sequence> ...`.
- **New app page**: `npx ng generate page <folder>/<name>` and add the route in the module's routing file; call the API through `RequestService`.

## Credits

Developed by Luca D'Agati and Antonio Pipitone.
