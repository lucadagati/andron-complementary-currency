# Andron — a complementary currency platform

Andron is a **complementary-currency system for local communities**. Citizens, businesses and charitable organisations (ETS) exchange services, products and community needs paid in *andron*, a local token whose ledger lives on a **Hyperledger Fabric** blockchain. Trusted *guarantor* entities (enti garanti) approve accounts, services, needs, products and handle reports.

## Components

| Directory | Stack | Role |
|---|---|---|
| `App/` | Ionic / Angular / Cordova | citizen & business mobile app: wallet, services, products, orders, needs, feedback |
| `Amministrazione/` | Ionic / Angular | guarantor back-office: activate users and businesses, approve content, map, orders |
| `Server/` | Python / Flask / MySQL | REST API (login, registration, services, products, orders, reports, guarantor area), e-mail templates |
| `fabric/chaincode/andron/` | TypeScript chaincode | wallet, deposit, transaction, settings and history contracts |
| `fabric/server/` | Node.js / Express + fabric-network | REST bridge between the API server and the chain |
| `fabric/test-network/`, `fabric/config/` | Hyperledger Fabric samples | local two-organisation test network |

## Architecture

```
mobile app / back-office  ──HTTP──►  Flask API (Server/)  ──►  MySQL (accounts, catalogue)
                                          │
                                          └─HTTP─►  Node bridge (fabric/server)  ──►  Fabric network + andron chaincode
```

## Running locally

Prerequisites: Python 3.8+, MySQL 8, Node 14, Docker, [Hyperledger Fabric 2.x binaries and images](https://hyperledger-fabric.readthedocs.io/en/latest/install.html) (not stored in this repository), Ionic CLI.

1. **Blockchain** – `cd fabric && ./installCC.sh` (starts the test network and deploys the chaincode), then in `fabric/server` run `npm install`, enrol the admin/user (`src/enrollAdmin`, `src/registerUser`) and `node main.js`. Wallet identities are generated locally and are git-ignored.
2. **Database** – `mysql -u root -p < Server/schema.sql`.
3. **API server** –
   ```bash
   cd Server
   pip install flask flask-cors pymysql
   export ANDRON_DB_PASSWORD=...            # also ANDRON_DB_HOST / _USER / _NAME
   export ANDRON_SMTP_USER=... ANDRON_SMTP_PASSWORD=...   # optional, for registration e-mails
   python server.py
   ```
4. **Apps** – `cd App && npm install && ionic serve` (same for `Amministrazione/`). Set the backend URL in `App/src/app/services/request.service.ts`.

## Repository hygiene

Dependencies (`node_modules`), Cordova platforms/plugins, build output, Fabric binaries, TLS keys, wallet identities, uploaded images and database dumps with sample data were removed from version control. Credentials are read from environment variables; the previously committed SMTP password and TLS keys must be considered compromised and rotated.

## Credits

Developed at the University of Messina (MDSLab) by Luca D'Agati and Antonio Pipitone.
