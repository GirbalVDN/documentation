Voici une procédure structurée que vous pouvez copier-coller dans un document (Word ou Markdown) pour l'exporter en **PDF**. Elle récapitule les étapes "universelles" pour migrer PostgreSQL sur une distribution de type RHEL/AlmaLinux avec les dépôts PGDG.

---

# Procédure de Migration Majeure PostgreSQL (Version $X$ vers $Y$)

**Serveur :** `wapt`  
**Objectif :** Migrer le moteur de base de données sans perte de données lorsque le paquet logiciel a été mis à jour avant les données.

---

## 1. Pré-requis et Préparation
* **Vérification :** S'assurer que les paquets `postgresqlX-server` et `postgresqlY-server` sont installés.
* **Sauvegarde :** Effectuer une copie physique du dossier de données actuel.
    ```bash
    tar -cvzf /backup/pgsql_data_old.tar.gz /var/lib/pgsql/data
    ```
* **Désactivation des modules :** Si `dnf` bloque l'installation des anciennes versions :
    ```bash
    sudo dnf -y module disable postgresql
    ```

## 2. Isolation des Données
PostgreSQL ne peut pas initialiser une nouvelle version dans un dossier non vide. Il faut déplacer l'ancienne version.

1.  **Arrêter le service :**
    ```bash
    sudo systemctl stop postgresql
    ```
2.  **Déplacer les anciennes données :**
    ```bash
    sudo mkdir -p /var/lib/pgsql/X/data
    sudo mv /var/lib/pgsql/data/* /var/lib/pgsql/X/data/
    sudo chown -R postgres:postgres /var/lib/pgsql/X/
    ```

## 3. Initialisation de la Nouvelle Version
Création d'un cluster vide en version $Y$ (cible).
```bash
sudo /usr/bin/postgresql-setup --initdb
```
*Le dossier `/var/lib/pgsql/data` est maintenant prêt et contient les fichiers de configuration de la version $Y$.*

## 4. Migration des Données (pg_upgrade)
L'outil `pg_upgrade` fait le lien entre l'ancien dossier et le nouveau.

```bash
sudo -u postgres pg_upgrade \
  --old-datadir=/var/lib/pgsql/X/data \
  --new-datadir=/var/lib/pgsql/data \
  --old-bindir=/usr/pgsql-X/bin \
  --new-bindir=/usr/bin
```
> **Note :** Si vous craignez pour l'espace disque, ajoutez l'option `--link` pour utiliser des liens physiques au lieu de copier les fichiers.

## 5. Finalisation et Optimisation
Une fois le message **"Upgrade Complete"** affiché :

1.  **Démarrer le nouveau service :**
    ```bash
    sudo systemctl start postgresql
    sudo systemctl enable postgresql
    ```
2.  **Recalculer les statistiques (Indispensable pour les performances) :**
    ```bash
    sudo -u postgres /usr/bin/vacuumdb --all --analyze-in-stages
    ```
3.  **Mettre à jour les extensions :**
    ```bash
    sudo -u postgres psql -f update_extensions.sql
    ```

## 6. Nettoyage
Après 24h de fonctionnement sans erreur :
* Supprimer l'ancien dossier : `rm -rf /var/lib/pgsql/X/data`
* Supprimer les anciens paquets : `dnf remove postgresqlX-server`

---

**Souhaitez-vous que je génère également un script Bash automatisé basé sur cette procédure pour vos prochaines mises à jour ?**
