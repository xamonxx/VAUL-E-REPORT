# Knowledge Map

Halaman ini menjelaskan cara membaca vault.

## Jalur baca cepat

- Agent backend: [[03-Backend/Backend-Map]] -> [[15-Backend-Internals/Backend-Internals]] -> [[06-API/API-Catalog]] -> modul terkait di [[11-Modules/Module-Index]].
- Agent frontend: [[04-Frontend/Frontend-Map]] -> [[14-Frontend-UI/UI-and-Navigation]] -> modul terkait -> [[06-API/API-Catalog]].
- Agent database: [[05-Database/Data-Model]] -> [[15-Backend-Internals/Models-and-Relationships]] -> [[15-Backend-Internals/Migrations-and-Indexes]].
- Agent ops/deploy: [[08-Ops/Runbook]] -> [[08-Ops/Deployment-Notes]] -> [[12-Security/Security-and-Access]].

## Cluster graph yang diharapkan

- Cluster domain: [[11-Modules/Consultations]], [[11-Modules/Survey]], [[11-Modules/Reporting-and-Analytics]].
- Cluster teknis backend: [[15-Backend-Internals/Backend-Internals]], [[15-Backend-Internals/Services]], [[15-Backend-Internals/Policies]].
- Cluster teknis frontend: [[14-Frontend-UI/UI-and-Navigation]], [[14-Frontend-UI/Frontend-Hooks]], [[14-Frontend-UI/Frontend-State-and-Cache]].
- Cluster keamanan: [[12-Security/Security-and-Access]], [[12-Security/Auth-Flow]], [[12-Security/Role-Matrix]].
- Cluster maintenance: [[08-Ops/Runbook]], [[13-Testing/Testing-Guide]], [[10-Change-Log/Change-Log]].

## Prinsip update

- Jika mengubah endpoint, update [[06-API/API-Catalog]] dan modul terkait.
- Jika mengubah table/model, update [[05-Database/Data-Model]] dan [[15-Backend-Internals/Models-and-Relationships]].
- Jika mengubah UI route, update [[04-Frontend/Frontend-Map]] dan [[14-Frontend-UI/UI-and-Navigation]].
- Jika mengubah role/izin, update [[12-Security/Role-Matrix]] dan [[15-Backend-Internals/Policies]].
- Semua perubahan tetap dicatat di [[10-Change-Log/Change-Log]].

