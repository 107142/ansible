[[_TOC_]]

# Client for Icinga API

This repository allows you to configure Icinga objects:

* hosts,
* services,
* contacts,
* notifications,
* groups of
  * hosts,
  * services
  * contacts
* dependencies between hosts and services

Everything is configured via variables.

## Structure

Entrypoint is the recipe `setup-monitoring.yml`, which runs the recipes pro individual configuration blocks.  
Configuration for these blocks can be found in `vars/` directory. Each configuration task to configure Icinga is named `icinga-*.yml` found inside directory `icinga/`.

### Used variables

Definitions of monitored objects is inside the files in `vars/` directory:

| File | Description |
| ---- | ----------- |
| `vars/groups.yml` | Contains the definitions of groups (hosts, services, contacts) |
| `vars/users.yml`  | Contains the definitions of contacts/users, who shall recieve notifications |
| `vars/hosts.yml`  | Contains the definitions of hosts, their services and notifications |
| `vars/deps.yml`   | Definitions of dependencies between services and hosts |
| `vars/vault.yml`  | Encrypted store of sensitive information (can contain Icinga API credentials) |

Objects can be configured via variables, which contain their respective definitions:

| Variable | Description |
| -------- | ----------- |
| `icinga_hostname`      | Hostname for Icinga API (defined in `setup-monitoring.yml`) |
| `icinga_apiuser`       | Username for Icinga API (saved in `vault.yml`) |
| `icinga_apipwd`        | Password for Icinga API (saved in `vault.yml`) |
| `icinga_usergroups`    | Definition of user groups |
| `icinga_hostgroups`    | Definition of host groups |
| `icinga_servicegroups` | Definition of service groups |
| `icinga_users`         | Definition of users |
| `icinga_hosts`         | Definition of hosts |
| `icinga_default_host_attrs`     | If you have multiple identical hosts, here you can define default values |
| `icinga_default_services`       | Default services monitored on the host |
| `icinga_default_host_notifs[1-4]`    | Default host notifications. Possible to set 5 defaults/templates |
| `icinga_default_service_notifs[1-4]` | Default service notifications. Possible to set 5 defaults/templates |

### Structure of variables

Každý soubor ve `vars/` obsahuje vzorový příklad jak definovat potřebné objekty. Struktura proměnných kopíruje volání API a je v podstatě vždy stejná. Každá proměnná je list položek, které jsou spravované. Každá položka pak obsahuje vždy stejné atributy.

Each file in `vars/` directoy contains an example how to define the necessary objects. The variables structure copies the API call and is essentially always the same. Each variable is a list of items, which are managed. Each item then contains always the same attributes.

```yaml
icinga_variable:
  - name: "object_name"
    state: "object_state"
    attrs:
      display_name: "Pretty Name"
      ...
```

* `name`: Name, which will be used to create object in Icinga
* `state`: Wheter to add or remove the object from Icinga, valid values are:
  * `present`: Object will be created, existing object will be updated.
  * `absent`: Existing object will be deleted
  * `recreate`: Deletes and then creates the existing object again (necessary due to API limitaions).
* `attrs`: For object creation.

Furhermore for hosts, services and notifications it's possible to define:  
* `template`: Templates/s used when creating objects

For hosts and services:  
* `cascade`: Used only during deletion. Removes dependant objects (services, notifications, etc.)
* `notifications`: Notifictions for host/service

Merge service with defaults:
* `services_combined`: Service will be merged with default defined in `icinga_default_services`.

Content of `attrs` depends on what is applicable for particular object. It is therefore necessary to respect the objects values according to [Icinga documentation](https://icinga.com/docs/icinga-2/latest/doc/09-object-types/).

### File structure

Ansible recipes are conceptualized in such a way as to be sufficiently flexible and easily extendable even for creation of other objects. The API call is realized in the file `icinga-base.yml`. Tasks which are part it are called with `icinga-base.yml` at the end of each remaining `icinga-*.yml`. Tasks in other `icinga-*.yml` files set variables to required values and at the end call `icinga-base.yml`.

If needed to create addional objects in Icinga it's possible to create an own task.

This system was created because Icinga API often requires during object manipulation specially formatted name. For example notifications have names in format: `host_name!service_name!notification_name`. You don't want to tpye this for every notifiction. Therefore `icinga-notifications.yml` will read from `icinga_hosts` data structure creating a list of notifications with everything in correct format and then passes this to tasks in `icinga-base.yml`.

## Usage

First you need credentials to access Icinga API. Those can be saved in `icinga_apiuser` and `icinga_apipwd`. `vault.yml` is password encrypted. That password cant be saved to file `.vault_pass`. Do NOT push that file to git. The easiest is therefore to save API credentials to `vars/vault.yml`.

```yaml
icinga_apiuser: "api_uzivatel"
icinga_apipwd: "heslo k api"
```

And then add password to `.vault_pass` to encrypt the file with command:

```yaml
ansible-vault encrypt vars/vault.yml
```

The encrypted file should be then save inside your repository. Ansible will automatically decrypt the content and read the variables thanks to the saved `.vault_pass` file. Should the `.vault_pass` not exists a prompt will appear.

Now inside `vars/*` you can define your	infrastructure as you see fit. Examples are in the respecitve files.

After configuring the infrastukture you can run Ansible to set everything up via API:

```
ansible-playbook setup-monitoring.yml
```


# Errata

This tool cannot process  hundreds of objects in one cycle. Possible because of inefficiency/limitations in [set_fact](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/set_fact_module.html) Ansible module. Approximate limit pro single cycle is cca 50 hosts with 10 service each.  
The easiest alevation is to use several playbook simultaneously.


# Credits

Created by Michal Heppler. Expanded by Marek Jaroš at Institute of Computer Science of Masaryk University.


# License

[GPL](LICENSE)
