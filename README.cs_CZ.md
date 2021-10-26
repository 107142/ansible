[[_TOC_]]

# Klient pro Icinga API

Tento repozitář umožňuje konfigurovat v Icinze:

* hosty
* služby
* kontakty
* notifikace
* skupiny
  * hostů
  * služeb
  * kontaktů
* závislosti mezi službami a hosty

Vše je konfigurovatelné přes proměnné.

## Struktura

Vstupní bod pro spuštění je předpis `setup-monitoring.yml`, který pouští předpisy pro jednotlivé konfigurační bloky.  
Konfigurace pro tyto bloky je v adresáři `vars/`. Jednotlivé úlohy pro konfiguraci Icingy jsou pojmenovány `icinga-*.yml` v adresáři `icinga/`.

### Použité proměnné

Definice monitorovaných objektů je v souborech v adresáři `vars/`:

| Soubor | Popis |
| ------ | ----- |
| `vars/groups.yml` | Obsahuje definici skupin (hostů, služeb, kontaktů) |
| `vars/users.yml`  | Obsahuje definici kontaktů/uživatelů, kteří budou dostávat notifikace |
| `vars/hosts.yml`  | Obsahuje definici hostů, jejich služeb a notifikací |
| `vars/deps.yml`   | Definice závislostí mezi službami a stroji |
| `vars/vault.yml`  | Šifrované úložiště citlivých informací (může obsahovat přístupové údaje k API Icingy) |

Nastavení objektů je možné provádět přes proměnné, které obsahují jednotlivé definice:

| Proměnná | Popis |
| -------- | ----- |
| `icinga_hostname`      | Hostname pro Icinga API (definováno v `setup-monitoring.yml`) |
| `icinga_apiuser`       | Uživatelské jméno pro přístup k API (uloženo ve vault.yml) |
| `icinga_apipwd`        | Heslo pro přístup k API (uloženo ve vault.yml) |
| `icinga_usergroups`    | Definice skupin uzivatelů |
| `icinga_hostgroups`    | Definice skupin hostů |
| `icinga_servicegroups` | Definice skupin služeb |
| `icinga_users`         | Definice kontaktů |
| `icinga_hosts`         | Definice hostů |
| `icinga_default_host_attrs`     | Pokud máte více stejných hostů, zde je defaultní nastavení |
| `icinga_default_services`       | Defaultní služby monitorované na hostovi |
| `icinga_default_host_notifs[1-4]`    | Defaultní notifikace hosta. Je možné nastavit až 5 šablon |
| `icinga_default_service_notifs[1-4]` | Defaultní notifikace služeb. Je možné nastavit ař 5 šablon |

### Struktura proměnných

Každý soubor ve `vars/` obsahuje vzorový příklad jak definovat potřebné objekty. Struktura proměnných kopíruje volání API a je v podstatě vždy stejná. Každá proměnná je list položek, které jsou spravované. Každá položka pak obsahuje vždy stejné atributy.

```yaml
icinga_proměnná:
  - name: "jmeno_objektu"
    state: "stav objektu"
    attrs:
      display_name: "Jméno Objektu"
      ...
```

* `name`: Jméno, pod kterým bude vytvořen v Icinze.
* `state`: Jestli má být objekt do Icingy přidán, nebo odebrán, platné hodnoty jsou:
  * `present`: Objekt bude vytvořen, existující aktualizován.
  * `absent`: Existující objekt bude zrušen.
  * `recreate`: Odstraní a znovu vytvoří existující objekt (implementováno kvůli omezením API).
* `attrs`: Pro vytvoření objektu.

Dále u hosta, služeb a notifikací je možné definovat:  
* `template`: Šablona/šablony použité při vytváření objektu.

U hosta a služby:

* `cascade`: Použitelné jen u odstraňování objektů, odstraní závislé objekty (služby, notifikace, etc.)
* `notifications`: Notifikace pro hosta/službu.

Sloučení služby s defaultními hodnotami:
* `services_combined`: Služby budou spojeny s defaulními `icinga_default_services`.

Obsah `attrs` se řídí tím, co je platné pro daný objekt. Je tedy potřeba respektovat povinné volby dle [dokumentace k Icinze](https://icinga.com/docs/icinga-2/latest/doc/09-object-types/).

### Struktura souborů

Předpisy pro Ansible jsou koncipovány tak, aby byly dostatečně flexibilní a jednoduše rozšiřitelné i pro vytváření dalších objektů. Samotné volání API Icingy je realizováno v souboru `icinga-base.yml`. Úlohy, které, které jsou jeho součástí, jsou volané pomocí `include_tasks` na konci každého zbývajícího `icinga-*.yml`. Tasky v ostatních `icinga-*.yml` souborech nastavují proměnné na potřebné hodnoty a na konec volají `icinga-base.yml`

Při potřebě vytváření dalších objektů v Icinze je tedy možné vytvořit si vlastní úlohu.

Tento systém je vytvořen kvůli tomu, že API vyžaduje při manipulaci s objekty často speciálně formátované jméno. Například notifikace mají jméno ve formátu: `jméno_hosta!jméno_služby!jméno_notifikace`. To nechcete u každé notifikace psát. `icinga-notifications.yml` tudíž napřed z datové struktury `icinga_hosts` vytvoří seznam notifikací se vším ve správném formátu, a potom toto předá úlohám v `icinga-base.yml`.


## Použití

Prvně je potřeba mít přístupové údaje k API Icingy. Ty mohou být uložené v proměnných `icinga_apiuser` a `icinga_apipwd`. `vault.yml` je šifrovaný heslem. Toto heslo je možné uložit do souboru `.vault_pass`. Tento soubor NEpřidávejte do Git-u. Nejjednodušší postup je tedy do `vars/vault.yml` uložit přístupové údaje k API:

```yaml
icinga_apiuser: "api_uzivatel"
icinga_apipwd: "heslo k api"
```

Následně do `.vault_pass` uložit heslo pro zašifrování pomocí příkazu:

```yaml
ansible-vault encrypt vars/vault.yml
```

Takto zašifrovaný soubor si potom uložte do repozitáře. Ansible při vykonání díky uloženému heslu ve `.vault_pass` automaticky rozšifruje obsah a načte proměnné. Pokud `.vault_pass` nebude existovat, na heslo se zeptá.

Nyní si do `vars/*` nadefinujte infrastrukturu jak Vám vyhovuje. Příklady jsou v jednotlivých souborech.

Po nakonfigurování infrastruktury zbývá jen nechat Ansible, ať vše přes API nastaví.

```
ansible-playbook setup-monitoring.yml
```


# Errata

Nástroj není schopen zpracovat stovky objektů v jednom cyklu. Patrně se jedná o neefektivitu/limitaci [set_fact](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/set_fact_module.html) Ansible modulu. Přibližný limit pro jeden běh je cca 50 strojů po 10-ti službách.  
Nejjednodušší řešení je použití několika playbooků současně.


# Credits

Vytvořil Michal Heppler. Rozšířil Marek Jaroš pro Ústav výpočetní techniky Masarykovy univerzity.


# Licence

[GPL](LICENSE)

