# IaC Magnum Opus – FiveM-server met Ansible

Dit project zet met **Ansible** volledig automatisch een **FiveM FXServer** (GTA V-multiplayerserver) op een Linux-server op. Eén playbook regelt alles: een aparte gebruiker aanmaken, de benodigde pakketten installeren, FXServer downloaden en configureren, de server als systemd-service draaien en dagelijkse back-ups inplannen.

Het project is gemaakt als *Magnum Opus* voor het vak Infrastructure as Code (AP Hogeschool).

---

## Inhoud

- [Projectstructuur](#projectstructuur)
- [Vereisten](#vereisten)
- [Configuratie](#configuratie)
- [Gebruik](#gebruik)
- [Rollen](#rollen)
- [Na de installatie](#na-de-installatie)
- [Bekende beperkingen](#bekende-beperkingen)
- [Documentatie](#documentatie)

---

## Projectstructuur

```text
IaC-MagnumOpus/
├── Documentie/
│   ├── Magnum Opus.docx          # Projectdocumentatie
│   └── Tijdsbijhouding.xlsx      # Tijdsregistratie
└── ansible-fivem/
    ├── ansible.cfg               # Ansible-instellingen (inventory, roles_path)
    ├── inventory.ini             # Doelserver(s) in de groep [fivem]
    ├── playbook.yml              # Hoofdplaybook
    ├── group_vars/
    │   └── all.yml               # Variabelen (database, licentiesleutel)
    └── roles/
        ├── user_setup/           # Gebruiker + mappen
        ├── dependencies/         # Pakketten installeren
        ├── fivem_install/        # FXServer downloaden en configureren
        ├── databese/             # MariaDB-database en -gebruiker (staat uit)
        ├── server_management/    # systemd-service
        └── backup/               # Dagelijkse back-up via cron
```

## Vereisten

**Op je eigen machine (control node)**

- Ansible 2.9 of nieuwer
- De collectie `community.mysql` (alleen nodig voor de rol `databese`):

  ```bash
  ansible-galaxy collection install community.mysql
  ```

- Een SSH-sleutel waarmee je als `root` op de doelserver kunt inloggen

**Doelserver**

- Debian/Ubuntu of een RedHat-gebaseerde distributie (de rol `dependencies` ondersteunt beide)
- Python 3 op `/usr/bin/python3`
- Poort **30120** (TCP en UDP) open in de firewall
- Een FiveM-licentiesleutel van de [Cfx.re Keymaster](https://keymaster.fivem.net/)

## Configuratie

### 1. Inventory

Zet het IP-adres van je server en het pad naar je SSH-sleutel in `ansible-fivem/inventory.ini`:

```ini
[fivem]
<IP-van-je-server> ansible_user=root ansible_ssh_private_key_file=~/.ssh/<jouw-sleutel>
```

### 2. Variabelen

Pas `ansible-fivem/group_vars/all.yml` aan:

| Variabele           | Beschrijving                                    |
|---------------------|-------------------------------------------------|
| `license`           | Je FiveM-licentiesleutel (`cfxk_...`)           |
| `db_login_user`     | Bestaande MariaDB-admin waarmee Ansible inlogt  |
| `db_login_password` | Wachtwoord van die admin                        |
| `db_name`           | Naam van de database voor de server             |
| `db_user`           | Databasegebruiker voor FiveM                    |
| `db_password`       | Wachtwoord van die databasegebruiker            |

> **Let op:** dit bestand bevat geheimen. Zet het niet in een openbare repository, of versleutel het met Ansible Vault:
>
> ```bash
> ansible-vault encrypt group_vars/all.yml
> ```

## Gebruik

```bash
cd ansible-fivem

# Verbinding testen
ansible fivem -m ping

# Droge run (laat zien wat er zou veranderen)
ansible-playbook playbook.yml --check

# Playbook uitvoeren
ansible-playbook playbook.yml

# Met een Vault-versleuteld variabelenbestand
ansible-playbook playbook.yml --ask-vault-pass
```

## Rollen

De rollen worden in deze volgorde uitgevoerd:

| # | Rol                 | Wat doet hij?                                                                                   |
|---|---------------------|-------------------------------------------------------------------------------------------------|
| 1 | `user_setup`        | Maakt de gebruiker `fivemuser` aan en de map `/home/fivemuser/server-data`.                     |
| 2 | `dependencies`      | Installeert `git`, `curl`, `screen`, `unzip`, `tar`, `mariadb-server`, PyMySQL en pip (apt of yum, plus EPEL op RedHat). |
| 3 | `fivem_install`     | Downloadt FXServer (build 7290) naar `/opt/fivem`, pakt het uit en schrijft een standaard `server.cfg` met je licentiesleutel. |
| – | `databese`          | Start MariaDB en maakt de database en de databasegebruiker aan. **Staat standaard uitgecommentarieerd in `playbook.yml`.** |
| 4 | `server_management` | Maakt de systemd-service `fivem.service` aan, zet hem aan bij het opstarten en start hem.       |
| 5 | `backup`            | Plaatst `/usr/local/bin/backup_fivem.sh` en een cronjob die elke dag om 03:00 een back-up maakt van `server-data` en de database naar `/var/backups`. |

Wil je de database-rol gebruiken? Haal dan het `#` weg voor `databese` in `playbook.yml`.

## Na de installatie

```bash
# Status van de server bekijken
systemctl status fivem

# Logs live volgen
journalctl -u fivem -f

# Server herstarten
systemctl restart fivem

# Handmatig een back-up maken
/usr/local/bin/backup_fivem.sh
```

Spelers kunnen verbinden via `connect <IP-van-je-server>:30120` in de FiveM-console (F8).

## Bekende beperkingen

- De FXServer-build staat vast op **7290** in `roles/fivem_install/tasks/main.yml`. Voor een nieuwere versie pas je daar de download-URL aan.
- Het back-upscript gebruikt eigen, vaste databasegegevens (`fivemuser` / database `fivem`), die niet overeenkomen met `group_vars/all.yml`. Dat moet nog gelijkgetrokken worden.
- De rol `databese` heet zo door een typfout (bedoeld: *database*).
- De README's in de afzonderlijke rollen zijn nog de standaardtemplates van `ansible-galaxy init`.

## Documentatie

In de map `Documentie/` staan de projectdocumentatie (`Magnum Opus.docx`) en de tijdsregistratie (`Tijdsbijhouding.xlsx`).

---

**Auteur:** Jason Van de Berckt – AP Hogeschool
