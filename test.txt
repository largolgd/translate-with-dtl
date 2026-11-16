#!/usr/bin/env bash
#==============================================================================
# upgrade-tomcat-jahia.sh  (Jahia 8.2+, application dans ROOT, AJP)
#
# Migration de Tomcat selon https://academy.jahia.com/tomcat-upgrade-documentation
# Le nouveau Tomcat n'est PAS téléchargé : archive apache-tomcat-9.0.x.tar.gz
# (ou dossier décompressé) attendue dans NEW_TOMCAT_SRC_DIR.
#
# Usage : ./upgrade-tomcat-jahia.sh
# Jahia est arrêté/démarré par systemd (sudo systemctl stop|start jahia).
# Un backup global (/home/webuser/livraison/backup.sh) est lancé avant la sauvegarde de Tomcat.
# En cas d'erreur après l'arrêt de Tomcat : rollback automatique.
#==============================================================================
set -Eeuo pipefail

#------------------------------ VARIABLES GLOBALES ----------------------------
NEW_TOMCAT_SRC_DIR="${NEW_TOMCAT_SRC_DIR:-/home/webuser}"   # nouveau Tomcat ici
TOMCAT_HOME="${TOMCAT_HOME:-/appli/jahia/tomcat}"           # Tomcat actuel
BACKUP_ROOT="${BACKUP_ROOT:-/home/webuser/tomcat-backups}"
LOG_DIR="${LOG_DIR:-/home/webuser/logs/tomcat-upgrade}"
STOP_TIMEOUT="${STOP_TIMEOUT:-120}"                         # secondes

# Arrêt / démarrage de Jahia via systemd
JAHIA_SERVICE="${JAHIA_SERVICE:-jahia}"
SYSTEMCTL="${SYSTEMCTL:-sudo systemctl}"

# Backup global préalable (exécute ./backup.sh depuis ce dossier)
GLOBAL_BACKUP_DIR="${GLOBAL_BACKUP_DIR:-/home/webuser/livraison}"

# Historisation des logs : un log par exécution + history.log (1 ligne par exécution)
LOG_COMPRESS_DAYS="${LOG_COMPRESS_DAYS:-7}"      # logs compressés (gzip) après N jours
LOG_RETENTION_DAYS="${LOG_RETENTION_DAYS:-365}"  # logs supprimés après N jours

# Pilotes JDBC restaurés depuis l'ancien lib/ (tous types, ceux présents sont copiés)
DB_DRIVER_PATTERNS=(
  "derby*.jar"                   # Derby
  "mysql-connector-*.jar"        # MySQL
  "mariadb-java-client-*.jar"    # MariaDB
  "mssql-jdbc-*.jar"             # SQL Server
  "ojdbc*.jar" "orai18n*.jar"    # Oracle
  "postgresql-*.jar"             # PostgreSQL
)

#------------------------------ VARIABLES INTERNES ----------------------------
TS="$(date +%Y%m%d-%H%M%S)"
LOG_FILE="${LOG_DIR}/tomcat-upgrade-${TS}.log"
HISTORY_FILE="${LOG_DIR}/history.log"
OLD_VERSION="?" ; NEW_VERSION="?"
OLD_DIR="${TOMCAT_HOME}-old-${TS}"
FAILED_DIR="${TOMCAT_HOME}-failed-${TS}"
BACKUP_DIR="${BACKUP_ROOT}/$(basename "$TOMCAT_HOME")-backup-${TS}"
NEW_SRC=""
WAS_RUNNING=0 ; STOPPED=0 ; OLD_MOVED=0 ; SUCCESS=false ; ROLLBACK_DONE=false

#------------------------------ LOGS ------------------------------------------
log()   { printf '%s [%-5s] %s\n' "$(date '+%F %T')" "$1" "${*:2}"; }
info()  { log INFO "$@"; }
warn()  { log WARN "$@" >&2; }
error() { log ERROR "$@" >&2; }
step()  { log STEP "━━━ Étape $1/12 : $2"; }
die()   { error "$@"; exit 1; }

mkdir -p "$LOG_DIR"
exec > >(tee -a "$LOG_FILE") 2>&1
info "Journal : $LOG_FILE"

# Historique : compresse les vieux logs, supprime les très anciens (history.log conservé)
rotate_logs() {
  find "$LOG_DIR" -maxdepth 1 -type f -name 'tomcat-upgrade-*.log' -mtime +"$LOG_COMPRESS_DAYS" -exec gzip -q {} + || true
  find "$LOG_DIR" -maxdepth 1 -type f -name 'tomcat-upgrade-*.log*' -mtime +"$LOG_RETENTION_DAYS" -delete || true
}

#------------------------------ JAHIA : arrêt / démarrage (systemd) -----------
tomcat_running() {
  local state
  state="$($SYSTEMCTL is-active "$JAHIA_SERVICE" 2>/dev/null || true)"
  [[ "$state" == active || "$state" == activating ]]
}

stop_tomcat() {
  info "Arrêt de Jahia : $SYSTEMCTL stop $JAHIA_SERVICE"
  $SYSTEMCTL stop "$JAHIA_SERVICE" || warn "systemctl stop a renvoyé une erreur"
  local deadline=$((SECONDS + STOP_TIMEOUT))
  while tomcat_running; do
    if (( SECONDS >= deadline )); then error "Jahia toujours actif après ${STOP_TIMEOUT}s"; return 1; fi
    sleep 2
  done
  info "Jahia arrêté."
}

start_tomcat() {
  info "Démarrage de Jahia : $SYSTEMCTL start $JAHIA_SERVICE"
  $SYSTEMCTL start "$JAHIA_SERVICE"
}

#------------------------------ ROLLBACK --------------------------------------
rollback() {
  [[ "$ROLLBACK_DONE" == true ]] && return 0
  ROLLBACK_DONE=true
  trap - ERR
  set +e
  warn "════════════ ROLLBACK ════════════"
  tomcat_running && stop_tomcat

  if (( OLD_MOVED )); then
    if [[ -e "$TOMCAT_HOME" ]]; then
      warn "Nouveau Tomcat mis de côté : $FAILED_DIR"
      mv "$TOMCAT_HOME" "$FAILED_DIR" || { error "mv du nouveau Tomcat impossible"; return 1; }
    fi
    if [[ -d "$OLD_DIR" ]]; then
      mv "$OLD_DIR" "$TOMCAT_HOME" || { error "Restauration depuis $OLD_DIR impossible"; return 1; }
    elif [[ -d "$BACKUP_DIR" ]]; then
      warn "Restauration depuis la sauvegarde $BACKUP_DIR"
      cp -a "$BACKUP_DIR" "$TOMCAT_HOME" || { error "Restauration depuis la sauvegarde impossible"; return 1; }
    else
      error "Aucune source de restauration disponible !"; return 1
    fi
    info "Ancien Tomcat remis en place : $TOMCAT_HOME"
  fi

  if (( WAS_RUNNING )); then
    start_tomcat || { error "Redémarrage de l'ancien Tomcat en échec : intervention manuelle."; return 1; }
    info "Ancien Tomcat redémarré."
  fi
}

record_history() {
  printf '%s | %-32s | %s -> %s | %s\n' "$(date '+%F %T')" "$1" "$OLD_VERSION" "$NEW_VERSION" "$LOG_FILE" >> "$HISTORY_FILE" || true
}

on_exit() {
  local rc=$?
  trap - EXIT
  if [[ "$SUCCESS" == true ]]; then
    record_history "SUCCES"
    return 0
  fi
  (( rc != 0 )) || return 0
  error "ÉCHEC de la migration (code $rc). Journal : $LOG_FILE"
  if (( STOPPED || OLD_MOVED )); then
    if rollback; then
      record_history "ECHEC - rollback OK"
    else
      record_history "ECHEC - ROLLBACK EN ECHEC"
      error "Rollback en échec. Manuel : arrêter Jahia ; mv $TOMCAT_HOME $FAILED_DIR ; mv $OLD_DIR $TOMCAT_HOME (ou cp -a $BACKUP_DIR $TOMCAT_HOME) ; redémarrer Jahia."
    fi
  else
    record_history "ECHEC - aucune modification"
  fi
}

trap 'error "Erreur ligne $LINENO : « $BASH_COMMAND »"' ERR
trap on_exit EXIT
trap 'exit 130' INT
trap 'exit 143' TERM

#------------------------------ OUTILS ----------------------------------------
# restore_file <chemin relatif> : restaure depuis l'ancien Tomcat (garde l'original en .tomcat-dist)
restore_file() {
  local src="$OLD_DIR/$1" dst="$TOMCAT_HOME/$1"
  [[ -f "$src" ]] || return 1
  [[ -f "$dst" ]] && cp -a "$dst" "${dst}.tomcat-dist"
  cp -a "$src" "$dst"
  info "  restauré : $1"
}

#------------------------------ ÉTAPE 1 : contrôles préalables ----------------
check_prerequisites() {
  local c
  for c in tar cp mv find sort sed grep awk gzip df du stat perl; do
    command -v "$c" >/dev/null 2>&1 || die "Commande requise absente : $c"
  done
  [[ -f "$TOMCAT_HOME/bin/catalina.sh" && -f "$TOMCAT_HOME/conf/server.xml" ]] \
    || die "Installation Tomcat introuvable dans $TOMCAT_HOME"
  [[ ! -L "$TOMCAT_HOME" ]] || die "$TOMCAT_HOME est un lien symbolique : indiquez le dossier réel"
  [[ -d "$TOMCAT_HOME/webapps/ROOT" ]] || die "webapps/ROOT absent de $TOMCAT_HOME"
  [[ -x "$GLOBAL_BACKUP_DIR/backup.sh" ]] || die "Backup global introuvable ou non exécutable : $GLOBAL_BACKUP_DIR/backup.sh"
  [[ -w "$(dirname "$TOMCAT_HOME")" ]] || die "Pas de droit d'écriture sur $(dirname "$TOMCAT_HOME")"
}

locate_new_tomcat() {
  local -a cands=()
  mapfile -t cands < <(find "$NEW_TOMCAT_SRC_DIR" -maxdepth 1 -type f -name 'apache-tomcat-*.tar.gz' | sort -V)
  (( ${#cands[@]} )) || mapfile -t cands < <(find "$NEW_TOMCAT_SRC_DIR" -maxdepth 1 -type d -name 'apache-tomcat-*' | sort -V)
  (( ${#cands[@]} )) || die "Aucun apache-tomcat-* trouvé dans $NEW_TOMCAT_SRC_DIR"
  (( ${#cands[@]} == 1 )) || warn "Plusieurs candidats (${cands[*]}) : le plus récent est retenu."
  NEW_SRC="${cands[-1]}"

  NEW_VERSION="$(basename "$NEW_SRC" .tar.gz | sed 's/^apache-tomcat-//')"
  OLD_VERSION="$(grep -m1 'Apache Tomcat Version' "$TOMCAT_HOME/RELEASE-NOTES" 2>/dev/null | awk '{print $NF}' || true)"
  info "Tomcat actuel : $TOMCAT_HOME ($OLD_VERSION) -> nouveau : $NEW_SRC ($NEW_VERSION)"
  [[ "$NEW_VERSION" =~ ^9\.0\. ]] || warn "La doc Jahia cible Tomcat 9.0.x (version détectée : $NEW_VERSION)"
}

verify_checksum() {
  local algo
  if [[ -d "$NEW_SRC" ]]; then
    [[ -f "$NEW_SRC/bin/catalina.sh" ]] || die "$NEW_SRC n'est pas un Tomcat valide"
    return 0
  fi
  for algo in sha512 sha256 sha1; do
    if [[ -f "$NEW_SRC.$algo" ]]; then
      ( cd "$(dirname "$NEW_SRC")" && "${algo}sum" -c "$(basename "$NEW_SRC").$algo" ) || die "Checksum $algo invalide"
      info "Checksum $algo OK"
      return 0
    fi
  done
  warn "Pas de fichier de checksum à côté de l'archive : vérification ignorée."
}

check_disk_space() {
  local need_kb avail_kb
  mkdir -p "$BACKUP_ROOT"
  need_kb="$(du -sk "$TOMCAT_HOME" | awk '{print $1}')"
  avail_kb="$(df -Pk "$BACKUP_ROOT" | awk 'NR==2 {print $4}')"
  (( avail_kb > need_kb * 11 / 10 )) || die "Espace insuffisant dans $BACKUP_ROOT (${avail_kb} Ko dispo, ${need_kb} Ko requis)"
}

do_preflight() {
  check_prerequisites
  TOMCAT_HOME="$(readlink -f "$TOMCAT_HOME")"
  locate_new_tomcat
  verify_checksum
  check_disk_space
  if tomcat_running; then WAS_RUNNING=1; info "Tomcat est en cours d'exécution."; fi
}

#------------------------------ ÉTAPES 2 à 5 : arrêt, sauvegarde, installation ----
do_stop() {
  if (( WAS_RUNNING )); then STOPPED=1; stop_tomcat; else info "Déjà arrêté."; fi
}

do_global_backup() {
  info "Backup global : cd $GLOBAL_BACKUP_DIR && ./backup.sh"
  ( cd "$GLOBAL_BACKUP_DIR" && ./backup.sh )
  info "Backup global terminé."
}

do_backup() {
  cp -a "$TOMCAT_HOME" "$BACKUP_DIR"
  info "Sauvegarde : $BACKUP_DIR"
}

do_move_old() {
  mv "$TOMCAT_HOME" "$OLD_DIR"
  OLD_MOVED=1
  info "Ancien Tomcat : $OLD_DIR"
}

fix_ownership() {
  local owner
  owner="$(stat -c '%U:%G' "$OLD_DIR")"
  if [[ $EUID -eq 0 ]]; then
    chown -R "$owner" "$TOMCAT_HOME"
    info "Propriétaire : $owner"
  elif [[ "$owner" != "$(id -un):$(id -gn)" ]]; then
    warn "Propriétaire de l'ancien Tomcat ($owner) ≠ utilisateur courant : vérifiez les droits."
  fi
}

do_install_new() {
  mkdir -p "$TOMCAT_HOME"
  if [[ -f "$NEW_SRC" ]]; then
    tar -xzf "$NEW_SRC" -C "$TOMCAT_HOME" --strip-components=1
  else
    cp -a "$NEW_SRC"/. "$TOMCAT_HOME"/
  fi
  [[ -f "$TOMCAT_HOME/bin/catalina.sh" ]] || die "Extraction invalide (bin/catalina.sh absent)"
  chmod +x "$TOMCAT_HOME"/bin/*.sh
  fix_ownership
}

#------------------------------ ÉTAPES 6 à 10 : restauration ------------------
do_restore_root() {
  rm -rf "${TOMCAT_HOME:?}/webapps/ROOT"
  cp -a "$OLD_DIR/webapps/ROOT" "$TOMCAT_HOME/webapps/ROOT"
  info "  webapps/ROOT restauré"
}

do_restore_db_drivers() {
  local pat f copied=0
  for pat in "${DB_DRIVER_PATTERNS[@]}"; do
    for f in "$OLD_DIR"/lib/$pat; do
      [[ -e "$f" ]] || continue
      cp -a "$f" "$TOMCAT_HOME/lib/"
      info "  pilote restauré : $(basename "$f")"
      copied=$((copied + 1))
    done
  done
  (( copied > 0 )) || die "Aucun pilote JDBC trouvé dans $OLD_DIR/lib : Jahia ne pourrait pas démarrer."
}

# AJP : secretRequired="false" sur les connecteurs AJP qui n'ont pas cet attribut
patch_ajp_connector() {
  local server_xml="$TOMCAT_HOME/conf/server.xml"
  perl -0pi.bak -e 's{(<Connector\b(?![^>]*secretRequired)[^>]*\bprotocol="AJP[^"]*"[^>]*?)(\s*/?>)}{$1 secretRequired="false"$2}g' "$server_xml"
  if grep -q 'protocol="AJP' "$server_xml"; then
    info "  AJP : secretRequired=\"false\" vérifié dans server.xml"
  else
    warn "  Aucun connecteur AJP trouvé dans server.xml"
  fi
}

do_restore_conf() {
  restore_file conf/server.xml          || die "conf/server.xml absent de l'ancien Tomcat"
  restore_file conf/catalina.properties || die "conf/catalina.properties absent de l'ancien Tomcat"
  restore_file conf/web.xml     || info "  (conf/web.xml non présent à l'origine)"
  restore_file conf/context.xml || info "  (conf/context.xml non présent à l'origine)"
  patch_ajp_connector
}

do_restore_setenv() {
  if restore_file bin/setenv.sh; then
    chmod +x "$TOMCAT_HOME/bin/setenv.sh"
  else
    warn "bin/setenv.sh absent de l'ancien Tomcat"
  fi
}

#------------------------------ ÉTAPE 12 : démarrage --------------
do_start() {
  start_tomcat
  info "À faire : contrôler les logs Jahia et surveiller les performances."
}

print_summary() {
  info "=== Migration réussie : $OLD_VERSION -> $NEW_VERSION ==="
  info "Ancien Tomcat conservé : $OLD_DIR"
  info "Sauvegarde complète    : $BACKUP_DIR"
}

#------------------------------ MAIN ------------------------------------------
main() {
  rotate_logs
  info "=== Migration Tomcat Jahia — début ($TS) ==="

  step 1  "Nouveau Tomcat et contrôles préalables"; do_preflight
  step 2  "Arrêt de Jahia";                         do_stop
  step 3  "Backup global (backup.sh)";              do_global_backup
  step 4  "Sauvegarde de l'actuel Tomcat";          do_backup
  step 5  "Mise de côté de l'ancienne installation"; do_move_old
  step 6  "Installation du nouveau Tomcat";         do_install_new
  step 7  "Restauration de webapps/ROOT";           do_restore_root
  step 8  "Libs portlet ignorées (Jahia >= 8.2)"
  step 9  "Restauration du pilote de base de données"; do_restore_db_drivers
  step 10 "Restauration de la configuration (AJP inclus)"; do_restore_conf
  step 11 "Restauration de setenv.sh";              do_restore_setenv
  step 12 "Démarrage de Jahia";                       do_start

  SUCCESS=true
  print_summary
}

main "$@"
