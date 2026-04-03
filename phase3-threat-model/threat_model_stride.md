# Threat Model STRIDE — OpenPLC Runtime v3
**Phase 3 — Analyse de la surface d'attaque**
*Mastère Cybersécurité 5ème année — TP Projet Final*

---

## Méthodologie

Analyse STRIDE appliquée à OpenPLC Runtime v3 (commit bb35f696), basée sur les résultats de l'analyse dynamique et de l'exploitation des CVE.

**Cible :** OpenPLC Runtime v3 — interface web Flask (port 8080), base SQLite, endpoint `/hardware`

---

## Matrice STRIDE

### S — Spoofing (Usurpation d'identité)

| ID | Vecteur | Description | CVSS | CVE/CWE |
|----|---------|-------------|------|---------|
| S-01 | Credentials par défaut | Le compte `openplc/openplc` permet à n'importe quel attaquant de s'authentifier comme administrateur sans connaissance préalable | **9.8** | CWE-1392 |
| S-02 | Absence de lockout | Aucune limitation de tentatives de login → brute force possible | **7.5** | CWE-307 |
| S-03 | Cookie sans attributs sécurisés | Cookie de session sans `Secure` flag → rejouable sur réseau non chiffré | **5.4** | CWE-614 |

---

### T — Tampering (Altération)

| ID | Vecteur | Description | CVSS | CVE/CWE |
|----|---------|-------------|------|---------|
| T-01 | RCE via `/hardware` | Upload d'un fichier de configuration matérielle non validé, compilé et exécuté en tant que root → modification arbitraire du système | **8.8** | CVE-2021-47770 / CWE-94 |
| T-02 | Modification SQLite | Accès direct à `openplc.db` (pas de chiffrement) → altération des credentials, settings, programmes PLC | **7.2** | CWE-312 |
| T-03 | Modification config via `/settings` | L'endpoint `/settings` permet de changer les ports Modbus/EtherNet/IP depuis l'interface web sans validation supplémentaire | **6.5** | CWE-284 |

---

### R — Repudiation (Répudiation)

| ID | Vecteur | Description | CVSS | CVE/CWE |
|----|---------|-------------|------|---------|
| R-01 | Absence de logs d'audit | Aucune trace des actions administratives (upload, modification settings, ajout utilisateur) → impossible de retracer une intrusion | **5.3** | CWE-778 |
| R-02 | Pas de traçabilité des connexions | Les tentatives de login (réussies ou échouées) ne sont pas journalisées | **4.3** | CWE-223 |

---

### I — Information Disclosure (Divulgation d'information)

| ID | Vecteur | Description | CVSS | CVE/CWE |
|----|---------|-------------|------|---------|
| I-01 | Mot de passe en clair dans SQLite | `SELECT * FROM Users` retourne `openplc` en clair — aucun hashage | **7.5** | CWE-256 |
| I-02 | Header `Server` exposé | `Werkzeug/2.3.7 Python/3.12.3` révélé dans chaque réponse HTTP → fingerprinting facilité | **5.3** | CWE-200 |
| I-03 | Endpoint `/users` sans RBAC | Liste complète des comptes accessible à tout utilisateur authentifié | **6.5** | CWE-284 |
| I-04 | Configuration industrielle exposée | `/settings` révèle ports Modbus (502), EtherNet/IP (44818), polling rates | **5.3** | CWE-200 |

---

### D — Denial of Service (Déni de service)

| ID | Vecteur | Description | CVSS | CVE/CWE |
|----|---------|-------------|------|---------|
| D-01 | Interface web non protégée | Port 8080 accessible sans restriction IP → saturation possible du serveur Flask (Werkzeug monothread) | **5.3** | CWE-400 |
| D-02 | Stack Buffer Overflow Modbus | Si port 502 ouvert : CVE-2024-34026 permet un crash du service via paquet Modbus malformé | **9.0** | CVE-2024-34026 |
| D-03 | Stack Buffer Overflow EtherNet/IP | Si port 44818 ouvert : buffer overflow via paquet EtherNet/IP malformé → arrêt du PLC | **9.0** | CVE-2024-34027 |

---

### E — Elevation of Privilege (Élévation de privilèges)

| ID | Vecteur | Description | CVSS | CVE/CWE |
|----|---------|-------------|------|---------|
| E-01 | RCE root via CVE-2021-47770 | OpenPLC tourne en root (via `sudo`) → tout RCE via `/hardware` donne un accès root complet au système hôte | **8.8** | CVE-2021-47770 |
| E-02 | Pas de séparation des privilèges | Le webserver et le runtime PLC tournent sous le même processus root — compromise du web = compromise du PLC | **8.0** | CWE-250 |

---

## Sélection des 3 scénarios les plus critiques

### Scénario 1 — Accès admin immédiat (CVSS 9.8)
**Vecteurs :** S-01 + I-01
```
Attaquant réseau → login openplc/openplc → accès dashboard admin complet
Temps d'exploitation : < 30 secondes
Outils requis : curl
```

### Scénario 2 — RCE root via upload matériel (CVSS 8.8)
**Vecteurs :** S-01 → T-01 → E-01
```
Login par défaut → upload fichier hardware malveillant → compilation & exécution root
→ reverse shell root sur le système hôte
CVE : CVE-2021-47770
Temps d'exploitation : < 5 minutes
Outils requis : curl, Python
```

### Scénario 3 — Crash du PLC industriel (CVSS 9.0)
**Vecteurs :** D-02 / D-03 (si ports ICS exposés)
```
Envoi d'un paquet Modbus/EtherNet/IP malformé → stack buffer overflow → arrêt du PLC
→ impact direct sur le processus industriel contrôlé
CVE : CVE-2024-34026 / CVE-2024-34027
```

---

## Scoring global de criticité

| Catégorie STRIDE | Nb vulnérabilités | CVSS max | Niveau de risque global |
|-----------------|-------------------|----------|------------------------|
| Spoofing | 3 | 9.8 | CRITIQUE |
| Tampering | 3 | 8.8 | ÉLEVÉ |
| Repudiation | 2 | 5.3 | MOYEN |
| Information Disclosure | 4 | 7.5 | ÉLEVÉ |
| Denial of Service | 3 | 9.0 | CRITIQUE |
| Elevation of Privilege | 2 | 8.8 | ÉLEVÉ |

---

*Document rédigé dans le cadre du TP Cybersécurité ICS/SCADA — Mastère Cybersécurité 5ème année*
