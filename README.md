# Ansible Deployment for Polywiki

Scripts Ansible pour le déploiement de l'application polywiki sur les serveurs de production EPFL.

## Prérequis

- Ansible 2.9+
- Accès SSH aux serveurs cibles (kis@kissrv121.epfl.ch, kis@kissrv124.epfl.ch)
- Le fichier WAR buildé (`build/libs/polywiki.war`)

Sur les serveurs cibles (déjà installés) :
- Tomcat 10+
- Apache httpd avec mod_ssl, mod_proxy, mod_rewrite
- MySQL/MariaDB
- Java 17+
- Certificats SSL dans `/var/www/vhosts/wiki.epfl.ch/ssl/`

## Structure

```
ansible/
├── ansible.cfg                 # Configuration Ansible
├── playbook.yml                # Playbook principal
├── inventories/
│   └── prod/
│       ├── hosts               # Inventaire des serveurs
│       └── group_vars/
│           └── all.yml         # Variables d'environnement
├── group_vars/
│   ├── all.yml                 # Variables globales
│   ├── vault.yml               # Secrets (chiffré)
│   └── vault.yml.example       # Template pour créer le vault
└── roles/
    ├── polywiki_tomcat/        # Configuration Tomcat
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── templates/
    │       ├── server.xml.j2
    │       ├── connector-http.xml.j2
    │       └── web.xml.j2
    ├── polywiki_app/           # Déploiement application
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── templates/
    │       ├── hibernate.cfg.xml.j2
    │       ├── appilis.application.properties.j2
    │       └── logrotate-polywiki.j2
    └── polywiki_apache/        # Configuration Apache
        ├── tasks/main.yml
        ├── handlers/main.yml
        └── templates/
            ├── polywiki-vhost.conf.j2
            └── polywiki-proxy.conf.j2
```

## Rôles

### polywiki_tomcat

Configure Tomcat :
- `server.xml` avec le connecteur HTTP
- `web.xml` avec les servlets par défaut
- Connecteur HTTP sur le port 8080 avec proxy headers

### polywiki_app

Déploie l'application :
1. Extrait le WAR dans `/srv/tomcat/wiki/private/polywiki-{version}`
2. Injecte les secrets dans `hibernate.cfg.xml` et `appilis.application.properties`
3. Met à jour le symlink `/srv/tomcat/wiki/webapps/ROOT`
4. Archive l'ancienne release
5. Nettoie les anciennes archives (garde les 5 dernières)
6. Redémarre Tomcat

### polywiki_apache

Configure Apache :
- VirtualHost HTTP (redirect vers HTTPS)
- VirtualHost HTTPS avec SSL
- Reverse proxy HTTP vers Tomcat
- Règles de réécriture pour les URLs wiki
- Redirections des anciens domaines

## Configuration initiale

### 1. Créer le fichier vault avec les secrets

```bash
cd ansible

# Créer le vault (première fois)
ansible-vault create group_vars/vault.yml

# Copier le contenu de vault.yml.example et modifier les valeurs
```

Les secrets requis :
- Credentials base de données (`vault_db_*`)
- Configuration OIDC Entra ID (`vault_entraid_*`)
- Configuration edu-ID (`vault_eduid_*`)
- Credentials FileService (`vault_fileservice_*`)

### 2. Éditer le vault existant

```bash
ansible-vault edit group_vars/vault.yml
```

### 3. Vérifier la configuration

```bash
# Vérifier la syntaxe
ansible-playbook playbook.yml --syntax-check

# Dry-run
ansible-playbook playbook.yml --ask-vault-pass --check
```

## Déploiement

### Build du WAR

```bash
# Depuis la racine du projet
make war
# ou
./gradlew war
```

### Déployer sur production

```bash
cd ansible

# Déploiement complet (Tomcat config + App + Apache)
ansible-playbook playbook.yml --ask-vault-pass

# Forcer la mise à jour si la version existe déjà
ansible-playbook playbook.yml --ask-vault-pass -e "polywiki_force_update=true"

# Déployer une version spécifique
ansible-playbook playbook.yml --ask-vault-pass -e "polywiki_version=2024-01-15"
```

### Déploiement partiel

```bash
# Uniquement la configuration Tomcat
ansible-playbook playbook.yml --ask-vault-pass --tags tomcat

# Uniquement l'application (WAR + secrets)
ansible-playbook playbook.yml --ask-vault-pass --tags app

# Uniquement la configuration Apache
ansible-playbook playbook.yml --ask-vault-pass --tags apache
```

### Cibler un serveur spécifique

```bash
# Déployer uniquement sur kissrv121
ansible-playbook playbook.yml --ask-vault-pass --limit kissrv121.epfl.ch
```

## Variables principales

| Variable | Description | Défaut |
|----------|-------------|--------|
| `polywiki_version` | Version/date de la release | Date du jour |
| `polywiki_force_update` | Écraser si la version existe | `false` |
| `polywiki_archive_keep_count` | Nombre d'archives à garder | `5` |
| `polywiki_tomcat_http_port` | Port HTTP Tomcat | `8080` |
| `polywiki_apache_server_name` | Hostname Apache | `wiki.epfl.ch` |

## Système d'archivage

Le déploiement archive automatiquement les anciennes releases :

- Les releases sont archivées dans `/srv/tomcat/wiki/private/_archive/`
- Format : `polywiki-{version}_{timestamp}`
- Les 5 dernières archives sont conservées (configurable via `polywiki_archive_keep_count`)

## Rollback

Pour revenir à une version précédente :

```bash
# Sur le serveur cible
sudo systemctl stop tomcat-wiki

# Lister les archives disponibles
ls -la /srv/tomcat/wiki/private/_archive/

# Restaurer une archive
rm /srv/tomcat/wiki/webapps/ROOT
ln -s /srv/tomcat/wiki/private/_archive/polywiki-2024-01-14_1705234567 /srv/tomcat/wiki/webapps/ROOT

sudo systemctl start tomcat-wiki
```

Ou redéployer une ancienne version avec Ansible :

```bash
ansible-playbook playbook.yml --ask-vault-pass -e "polywiki_version=2024-01-14" -e "polywiki_force_update=true"
```

## Troubleshooting

### Voir les logs Tomcat

```bash
ssh kis@kissrv121.epfl.ch
tail -f /srv/tomcat/wiki/logs/catalina.out
tail -f /srv/tomcat/wiki/logs/polywiki.log
```

### Voir les logs Apache

```bash
tail -f /var/www/vhosts/wiki.epfl.ch/logs/error.log
tail -f /var/www/vhosts/wiki.epfl.ch/logs/access.log
```

### Tester la connectivité

```bash
# Tester le playbook en mode verbose
ansible-playbook playbook.yml --ask-vault-pass -vvv

# Tester la connexion SSH
ansible polywiki_prod -m ping
```

### Vérifier les certificats SSL

```bash
ssh kis@kissrv121.epfl.ch
ls -la /var/www/vhosts/wiki.epfl.ch/ssl/
openssl x509 -in /var/www/vhosts/wiki.epfl.ch/ssl/wiki.epfl.ch.crt.pem -noout -dates
```
