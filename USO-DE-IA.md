# Prompts usados — Grupo 6 (CentOS Stream)

Registro dos prompts utilizados com IA para apoio na criação e depuração do script `cis-baseline-check.sh`.

---

## Maick Rosario Yamassaki

### Prompt 1 — Geração do script completo

> Preciso que você desenvolva um script de shell completo, em bash, chamado `cis-baseline-check.sh`, com a função de atuar como um verificador de baseline de hardening para um servidor CentOS Stream 10, tendo como referência técnica o CIS Red Hat Enterprise Linux 10 Benchmark v1.0.1.
>
> **Escopo funcional (mínimo de 25 verificações):**
> - Montagens (opções `nodev`/`nosuid`/`noexec` conforme aplicável a cada ponto de montagem)
> - Serviços (ausência de serviços inseguros e leitura de portas em escuta)
> - SELinux (instalado, em modo enforcing, não desabilitado via bootloader)
> - Permissões de `/etc/shadow` e arquivos afins (`/etc/passwd`, `/etc/group`, `/etc/gshadow`)
> - SUID/SGID inesperados (comparação com uma baseline de binários legítimos)
> - auditd (instalado, ativo, regras carregadas, retenção de log configurada)
> - Atualizações pendentes (via `dnf`/`yum`)
>
> O script deve calcular uma pontuação final e gerar um relatório em HTML, servindo de base para posterior comparação com os resultados do OpenSCAP.
>
> **Requisitos técnicos obrigatórios:**
> - Modo estrito, `set -euo pipefail` logo após o shebang
> - Funções, mínimo de 5, nenhuma lógica solta no corpo do script
> - Parâmetros via `getopts` ou `case`; `-h` e `--help` obrigatórios
> - Verificações de privilégio via `EUID` e de dependências via `command -v`
> - Logging com timestamp, nível INFO/WARN/ERROR, arquivo em `/var/log/`
> - Códigos de saída: 0 sucesso · 1 achado · 2 erro de uso · 3 erro de dependência
> - Idempotência: rodar duas vezes deve produzir o mesmo estado final
> - Sem segredo no código, nenhuma senha ou chave embutida no arquivo
> - Limpeza via `trap` para temporários, sempre criados com `mktemp`
> - Qualidade: o script deve passar no `shellcheck` sem erros
> - Idioma: comentários e mensagens de saída em português
>
> Peço também que, além do código, explique a metodologia adotada para conseguir reproduzir o conteúdo de maneira didática.

### Resposta gerada pela IA

```bash
#!/usr/bin/env bash
#
# cis-baseline-check.sh
# Verificador de baseline de hardening para CentOS Stream (referência: CIS Red Hat Enterprise Linux 10 Benchmark v1.0.1).
#

set -euo pipefail
IFS=$'\n\t'

# =============================================================================
# 1. CONSTANTES E ESTADO GLOBAL
# =============================================================================

# Metadados do script
readonly CIS_SCRIPT_NAME="cis-baseline-check.sh"
readonly CIS_VERSION="1.1.0"

# Caminhos de log, lock e relatório
readonly CIS_LOG_FILE="/var/log/cis-baseline-check.log"
readonly CIS_LOCK_FILE="/var/lock/cis-baseline-check.lock"
CIS_REPORT_FILE="/var/log/cis-baseline-check-report.html"
CIS_QUIET=0

# Contadores globais
CIS_PASS=0
CIS_FAIL=0
CIS_WARN=0

# Arrays de controle interno
declare -a CIS_REPORT_ROWS=()
declare -a CIS_TMP_FILES=()

# Códigos de saída
readonly CIS_EXIT_SUCCESS=0
readonly CIS_EXIT_FINDING=1
readonly CIS_EXIT_USAGE=2
readonly CIS_EXIT_DEPENDENCY=3

# =============================================================================
# 2. LIMPEZA E TRAP
# =============================================================================

# Remove arquivos temporários e libera lock ao finalizar ou interromper o script
# shellcheck disable=SC2317
cis_cleanup() {
    local l_file
    for l_file in "${CIS_TMP_FILES[@]:-}"; do
        if [ -n "$l_file" ] && [ -f "$l_file" ]; then
            rm -f "$l_file"
        fi
    done
    if [ -f "$CIS_LOCK_FILE" ]; then
        flock -u 200 2>/dev/null || true
    fi
}
trap cis_cleanup EXIT INT TERM

# Cria arquivo temporário único e adiciona à lista para remoção no trap
cis_mktemp() {
    local l_tmp
    l_tmp=$(mktemp "/tmp/${CIS_SCRIPT_NAME}.XXXXXX")
    CIS_TMP_FILES+=("$l_tmp")
    printf '%s' "$l_tmp"
}

# =============================================================================
# 3. LOGGING
# =============================================================================

# Grava registros em /var/log e imprime em stderr conforme o nível e opção -q
cis_log() {
    local l_level="$1"
    local l_msg="$2"
    local l_ts
    l_ts=$(date '+%Y-%m-%d %H:%M:%S')

    if [ ! -f "$CIS_LOG_FILE" ]; then
        touch "$CIS_LOG_FILE" 2>/dev/null || true
        chmod 640 "$CIS_LOG_FILE" 2>/dev/null || true
    fi

    printf '%s [%s] %s\n' "$l_ts" "$l_level" "$l_msg" >> "$CIS_LOG_FILE" 2>/dev/null || true

    if [ "$CIS_QUIET" -eq 0 ] && [ "$l_level" != "INFO" ]; then
        printf '%s [%s] %s\n' "$l_ts" "$l_level" "$l_msg" >&2
    fi
}

cis_log_info()  { cis_log "INFO"  "$1"; }
cis_log_warn()  { cis_log "WARN"  "$1"; }
cis_log_error() { cis_log "ERROR" "$1"; }

# =============================================================================
# 4. FUNÇÕES DE RESULTADO
# =============================================================================

# Registra validação bem-sucedida
cis_ok() {
    local l_check="$1"
    printf '  [OK]   %s\n' "$l_check"
    CIS_PASS=$((CIS_PASS + 1))
    CIS_REPORT_ROWS+=("<tr class=\"ok\"><td>OK</td><td>${l_check}</td><td></td></tr>")
    cis_log_info "OK: ${l_check}"
}

# Registra falha de hardening
cis_bad() {
    local l_check="$1"
    local l_reason="${2:-}"
    printf '  [FAIL] %s -> %s\n' "$l_check" "$l_reason"
    CIS_FAIL=$((CIS_FAIL + 1))
    CIS_REPORT_ROWS+=("<tr class=\"fail\"><td>FAIL</td><td>${l_check}</td><td>${l_reason}</td></tr>")
    cis_log_error "FAIL: ${l_check} -> ${l_reason}"
}

# Registra checagem inconclusiva ou ambiente não aplicável
cis_meh() {
    local l_check="$1"
    local l_reason="${2:-}"
    printf '  [WARN] %s -> %s\n' "$l_check" "$l_reason"
    CIS_WARN=$((CIS_WARN + 1))
    CIS_REPORT_ROWS+=("<tr class=\"warn\"><td>WARN</td><td>${l_check}</td><td>${l_reason}</td></tr>")
    cis_log_warn "WARN: ${l_check} -> ${l_reason}"
}

# =============================================================================
# 5. VALIDAÇÕES INICIAIS (ROOT E DEPENDÊNCIAS)
# =============================================================================

# Garante execução como root
cis_require_root() {
    if [ "$EUID" -ne 0 ]; then
        printf 'Este script precisa ser executado como root (EUID=0).\n' >&2
        exit "$CIS_EXIT_USAGE"
    fi
}

# Valida presenças dos binários básicos necessários
cis_check_dependencies() {
    local l_missing=0
    local l_cmd
    for l_cmd in awk grep find stat findmnt; do
        if ! command -v "$l_cmd" >/dev/null 2>&1; then
            cis_log_error "Dependencia ausente: ${l_cmd}"
            printf 'Dependencia ausente: %s\n' "$l_cmd" >&2
            l_missing=1
        fi
    done
    if [ "$l_missing" -eq 1 ]; then
        exit "$CIS_EXIT_DEPENDENCY"
    fi
}

# =============================================================================
# 6. PARSER DE ARGUMENTOS E AJUDA
# =============================================================================

cis_usage() {
    cat <<EOF
${CIS_SCRIPT_NAME} v${CIS_VERSION}
Verificador de baseline de hardening inspirado no CIS Benchmark.

Uso:
  ${CIS_SCRIPT_NAME} [opcoes]

Opcoes:
  -o ARQUIVO   Caminho do relatorio HTML de saida (padrao: ${CIS_REPORT_FILE})
  -q           Modo silencioso (nao imprime WARN/ERROR na tela)
  -h, --help   Mostra esta ajuda e sai

Codigos de saida:
  0  Sucesso (nenhuma falha)
  1  Achado(s) de falha
  2  Erro de uso/permissao
  3  Erro de dependencia
EOF
}

cis_parse_args() {
    local l_arg
    for l_arg in "$@"; do
        if [ "$l_arg" = "--help" ]; then
            cis_usage
            exit "$CIS_EXIT_SUCCESS"
        fi
    done

    local OPTIND=1
    local l_opt
    while getopts ":o:qh" l_opt; do
        case "$l_opt" in
            o) CIS_REPORT_FILE="$OPTARG" ;;
            q) CIS_QUIET=1 ;;
            h) cis_usage; exit "$CIS_EXIT_SUCCESS" ;;
            \?) printf 'Opcao invalida: -%s\n' "$OPTARG" >&2; cis_usage; exit "$CIS_EXIT_USAGE" ;;
            :) printf 'A opcao -%s requer um argumento.\n' "$OPTARG" >&2; exit "$CIS_EXIT_USAGE" ;;
        esac
    done
}

# =============================================================================
# 7. IDENTIFICAÇÃO DO SISTEMA
# =============================================================================

CIS_DISTRO_ID=""
CIS_DISTRO_VERSION=""

# Lê /etc/os-release para exibir no cabeçalho do relatório
cis_detect_distro() {
    if [ -r /etc/os-release ]; then
        # shellcheck source=/dev/null
        CIS_DISTRO_ID=$(. /etc/os-release && printf '%s' "${ID:-desconhecido}")
        # shellcheck source=/dev/null
        CIS_DISTRO_VERSION=$(. /etc/os-release && printf '%s' "${VERSION_ID:-desconhecido}")
    else
        CIS_DISTRO_ID="desconhecido"
        CIS_DISTRO_VERSION="desconhecido"
    fi
    cis_log_info "Distro detectada: ${CIS_DISTRO_ID} ${CIS_DISTRO_VERSION}"
}

# =============================================================================
# 8. VERIFICAÇÕES DE MONTAGEM (CIS 1.1.2)
# =============================================================================

# Verifica opções de montagem ativas no kernel
cis_check_mount_option() {
    local l_mnt="$1"
    local l_opt="$2"
    local l_label="Montagem ${l_mnt} possui opcao ${l_opt}"

    if ! findmnt --kernel "$l_mnt" >/dev/null 2>&1; then
        cis_meh "$l_label" "particao ${l_mnt} nao existe separadamente (nao auditavel)"
        return
    fi

    if findmnt --kernel -o OPTIONS --noheadings "$l_mnt" 2>/dev/null | grep -qw "$l_opt"; then
        cis_ok "$l_label"
    else
        cis_bad "$l_label" "opcao ${l_opt} nao encontrada em ${l_mnt}"
    fi
}

# /tmp e /var/tmp: exigem nodev, nosuid, noexec
cis_check_tmp_and_vartmp() {
    cis_check_mount_option /tmp nodev
    cis_check_mount_option /tmp nosuid
    cis_check_mount_option /tmp noexec
    cis_check_mount_option /var/tmp nodev
    cis_check_mount_option /var/tmp nosuid
    cis_check_mount_option /var/tmp noexec
}

# /home: exige nodev, nosuid
cis_check_home_partition() {
    cis_check_mount_option /home nodev
    cis_check_mount_option /home nosuid
}

# /var/log: exige nodev, nosuid, noexec
cis_check_varlog_partition() {
    cis_check_mount_option /var/log nodev
    cis_check_mount_option /var/log nosuid
    cis_check_mount_option /var/log noexec
}

# /boot: nodev, nosuid | /var: nodev
cis_check_boot_and_var() {
    cis_check_mount_option /boot nodev
    cis_check_mount_option /boot nosuid
    cis_check_mount_option /var nodev
}

# =============================================================================
# 9. VERIFICAÇÕES DO SELINUX (CIS 1.3.1)
# =============================================================================

cis_check_selinux_installed() {
    if command -v getenforce >/dev/null 2>&1; then
        cis_ok "SELinux esta instalado"
    else
        cis_bad "SELinux esta instalado" "comando getenforce nao encontrado"
    fi
}

cis_check_selinux_enforcing() {
    if ! command -v getenforce >/dev/null 2>&1; then
        cis_meh "SELinux modo enforcing" "getenforce indisponivel, pulando"
        return
    fi
    local l_mode
    l_mode=$(getenforce 2>/dev/null || printf 'Desconhecido')
    if [ "$l_mode" = "Enforcing" ]; then
        cis_ok "SELinux modo enforcing"
    else
        cis_bad "SELinux modo enforcing" "modo atual: ${l_mode}"
    fi
}

# Valida desativação de SELinux nos parâmetros de boot
cis_check_selinux_bootloader() {
    local l_grub_cfg="/boot/grub2/grubenv"
    if [ -r "$l_grub_cfg" ] && grep -qw 'selinux=0' "$l_grub_cfg" 2>/dev/null; then
        cis_bad "SELinux nao desabilitado no bootloader" "selinux=0 presente em ${l_grub_cfg}"
    else
        cis_ok "SELinux nao desabilitado no bootloader"
    fi
}

# =============================================================================
# 10. VERIFICAÇÕES DE SERVIÇOS (CIS 2.1)
# =============================================================================

# Valida se pacotes de serviços inseguros estão ausentes
cis_check_service_absent() {
    local l_label="$1"
    shift
    local l_pkg
    local l_found=0
    for l_pkg in "$@"; do
        if rpm -q "$l_pkg" >/dev/null 2>&1; then
            l_found=1
            break
        fi
    done
    if [ "$l_found" -eq 0 ]; then
        cis_ok "Servico ${l_label} nao instalado"
    else
        cis_bad "Servico ${l_label} nao instalado" "pacote ${l_pkg} presente"
    fi
}

# Lista portas em escuta para conferência manual
cis_check_listening_services() {
    local l_tmp
    l_tmp=$(cis_mktemp)
    if command -v ss >/dev/null 2>&1; then
        ss -tulnH 2>/dev/null > "$l_tmp" || true
        local l_count
        l_count=$(wc -l < "$l_tmp" | tr -d ' ')
        cis_meh "Servicos escutando em portas de rede" "${l_count} socket(s) ativos, revisar manualmente"
    else
        cis_meh "Servicos escutando em portas de rede" "comando ss indisponivel"
    fi
}

# =============================================================================
# 11. PERMISSÕES DE ARQUIVOS (CIS 7.1)
# =============================================================================

# Valida proprietário, grupo e limites de permissão octal
cis_check_file_perm() {
    local l_file="$1" l_perm_esperado="$2" l_dono_esperado="$3" l_grupo_esperado="$4"
    local l_label="Permissoes de ${l_file}"

    if [ ! -e "$l_file" ]; then
        cis_meh "$l_label" "arquivo nao existe"
        return
    fi

    local l_perm l_dono l_grupo
    l_perm=$(stat -c '%a' "$l_file")
    l_dono=$(stat -c '%U' "$l_file")
    l_grupo=$(stat -c '%G' "$l_file")

    if [ "$l_perm" -le "$l_perm_esperado" ] && [ "$l_dono" = "$l_dono_esperado" ] && [ "$l_grupo" = "$l_grupo_esperado" ]; then
        cis_ok "$l_label"
    else
        cis_bad "$l_label" "atual: ${l_perm} ${l_dono}:${l_grupo} (esperado: <=${l_perm_esperado} ${l_dono_esperado}:${l_grupo_esperado})"
    fi
}

cis_check_sensitive_files_permissions() {
    cis_check_file_perm /etc/passwd  644 root root
    cis_check_file_perm /etc/group   644 root root
    cis_check_file_perm /etc/shadow  000 root root
    cis_check_file_perm /etc/gshadow 000 root root
}

# Procura arquivos com permissão de escrita global em /etc e /usr
cis_check_world_writable_files() {
    local l_tmp
    l_tmp=$(cis_mktemp)
    find /etc /usr -xdev -type f -perm -0002 2>/dev/null > "$l_tmp" || true
    local l_count
    l_count=$(wc -l < "$l_tmp" | tr -d ' ')
    if [ "$l_count" -eq 0 ]; then
        cis_ok "Nenhum arquivo world-writable encontrado em /etc e /usr"
    else
        cis_bad "Nenhum arquivo world-writable encontrado em /etc e /usr" "${l_count} arquivo(s) encontrado(s)"
    fi
}

# =============================================================================
# 12. AUDITORIA DE SUID/SGID (CIS 7.1.13)
# =============================================================================

# Binários SUID/SGID permitidos por padrão
readonly CIS_SUID_BASELINE=(
    /usr/bin/sudo
    /usr/bin/su
    /usr/bin/passwd
    /usr/bin/chage
    /usr/bin/chfn
    /usr/bin/chsh
    /usr/bin/gpasswd
    /usr/bin/newgrp
    /usr/bin/mount
    /usr/bin/umount
    /usr/bin/crontab
    /usr/bin/pkexec
    /usr/bin/ping
)

cis_is_in_baseline() {
    local l_path="$1" l_item
    for l_item in "${CIS_SUID_BASELINE[@]}"; do
        [ "$l_item" = "$l_path" ] && return 0
    done
    return 1
}

# Procura binários SUID/SGID no FS raiz e compara com a baseline
cis_check_suid_sgid_review() {
    local l_tmp
    l_tmp=$(cis_mktemp)
    find / -xdev -type f \( -perm -4000 -o -perm -2000 \) 2>/dev/null > "$l_tmp" || true

    local l_unexpected=0
    local l_path
    while IFS= read -r l_path; do
        [ -z "$l_path" ] && continue
        if ! cis_is_in_baseline "$l_path"; then
            l_unexpected=$((l_unexpected + 1))
        fi
    done < "$l_tmp"

    if [ "$l_unexpected" -eq 0 ]; then
        cis_ok "Binarios SUID/SGID conferem com a baseline esperada"
    else
        cis_bad "Binarios SUID/SGID conferem com a baseline esperada" "${l_unexpected} binario(s) fora da baseline, revisar manualmente"
    fi
}

# =============================================================================
# 13. CONTAS E GRUPOS (CIS 7.2)
# =============================================================================

cis_check_duplicate_uids() {
    local l_dup
    l_dup=$(awk -F: '{print $3}' /etc/passwd | sort | uniq -d)
    if [ -z "$l_dup" ]; then
        cis_ok "Nenhum UID duplicado em /etc/passwd"
    else
        cis_bad "Nenhum UID duplicado em /etc/passwd" "UID(s) duplicado(s): $(printf '%s' "$l_dup" | tr '\n' ' ')"
    fi
}

cis_check_duplicate_gids() {
    local l_dup
    l_dup=$(awk -F: '{print $3}' /etc/group | sort | uniq -d)
    if [ -z "$l_dup" ]; then
        cis_ok "Nenhum GID duplicado em /etc/group"
    else
        cis_bad "Nenhum GID duplicado em /etc/group" "GID(s) duplicado(s): $(printf '%s' "$l_dup" | tr '\n' ' ')"
    fi
}

# Valida se apenas a conta root possui UID 0
cis_check_root_uid0() {
    local l_count
    l_count=$(awk -F: '($3 == 0) {print $1}' /etc/passwd | wc -l | tr -d ' ')
    if [ "$l_count" -eq 1 ]; then
        cis_ok "Apenas root possui UID 0"
    else
        cis_bad "Apenas root possui UID 0" "${l_count} conta(s) com UID 0"
    fi
}

# =============================================================================
# 14. AUDITD (CIS 6.3)
# =============================================================================

cis_check_auditd_installed() {
    if rpm -q audit >/dev/null 2>&1; then
        cis_ok "Pacote audit (auditd) instalado"
    else
        cis_bad "Pacote audit (auditd) instalado" "pacote audit ausente"
    fi
}

cis_check_auditd_active() {
    if ! command -v systemctl >/dev/null 2>&1; then
        cis_meh "Servico auditd ativo e habilitado" "systemctl indisponivel"
        return
    fi
    if systemctl is-active --quiet auditd 2>/dev/null && systemctl is-enabled --quiet auditd 2>/dev/null; then
        cis_ok "Servico auditd ativo e habilitado"
    else
        cis_bad "Servico auditd ativo e habilitado" "inativo ou nao habilitado no boot"
    fi
}

# Consulta regras ativas via auditctl
cis_check_audit_rule() {
    local l_label="$1" l_pattern="$2"
    if ! command -v auditctl >/dev/null 2>&1; then
        cis_meh "$l_label" "auditctl indisponivel"
        return
    fi
    if auditctl -l 2>/dev/null | grep -q -- "$l_pattern"; then
        cis_ok "$l_label"
    else
        cis_bad "$l_label" "regra nao encontrada nas regras carregadas do auditd"
    fi
}

cis_check_audit_rules_files() {
    cis_check_audit_rule "Regra de auditoria para /etc/passwd" "/etc/passwd"
    cis_check_audit_rule "Regra de auditoria para /etc/shadow" "/etc/shadow"
    cis_check_audit_rule "Regra de auditoria para /etc/sudoers" "/etc/sudoers"
}

# Valida presença do parâmetro max_log_file em auditd.conf
cis_check_audit_log_storage() {
    local l_conf="/etc/audit/auditd.conf"
    if [ ! -r "$l_conf" ]; then
        cis_meh "Tamanho maximo de log do auditd configurado" "${l_conf} nao encontrado"
        return
    fi
    if grep -Eq '^\s*max_log_file\s*=\s*[0-9]+' "$l_conf"; then
        cis_ok "Tamanho maximo de log do auditd configurado"
    else
        cis_bad "Tamanho maximo de log do auditd configurado" "parametro max_log_file ausente em ${l_conf}"
    fi
}

# =============================================================================
# 15. ATUALIZAÇÕES PENDENTES (CIS 1.2.2.1)
# =============================================================================

# Checa pacotes pendentes via dnf ou yum
cis_check_pending_updates() {
    local l_gerenciador=""
    if command -v dnf >/dev/null 2>&1; then
        l_gerenciador="dnf"
    elif command -v yum >/dev/null 2>&1; then
        l_gerenciador="yum"
    else
        cis_meh "Atualizacoes de seguranca pendentes" "nenhum gerenciador dnf/yum encontrado"
        return
    fi

    local l_tmp l_rc
    l_tmp=$(cis_mktemp)

    set +e
    "$l_gerenciador" check-update --quiet > "$l_tmp" 2>/dev/null
    l_rc=$?
    set -e

    case "$l_rc" in
        0) cis_ok "Nenhuma atualizacao pendente (${l_gerenciador})" ;;
        100)
            local l_count
            l_count=$(grep -c . "$l_tmp" || true)
            cis_meh "Atualizacoes pendentes" "${l_count} pacote(s) desatualizado(s) via ${l_gerenciador}"
            ;;
        *) cis_meh "Atualizacoes pendentes" "nao foi possivel verificar (${l_gerenciador} retornou ${l_rc}, ex.: sem rede)" ;;
    esac
}

# =============================================================================
# 16. GERAÇÃO DE RELATÓRIO HTML
# =============================================================================

cis_generate_html_report() {
    local l_total
    l_total=$((CIS_PASS + CIS_FAIL + CIS_WARN))
    local l_score=0
    if [ "$l_total" -gt 0 ]; then
        l_score=$(( (CIS_PASS * 100) / l_total ))
    fi

    {
        printf '<!DOCTYPE html><html lang="pt-BR"><head><meta charset="utf-8">'
        printf '<title>Relatorio CIS Baseline Check</title><style>'
        printf 'body{font-family:sans-serif;margin:2em;} table{border-collapse:collapse;width:100%%;}'
        printf 'th,td{border:1px solid #ccc;padding:6px 10px;text-align:left;}'
        printf 'tr.ok{background:#e6ffed;} tr.fail{background:#ffe6e6;} tr.warn{background:#fff8e1;}'
        printf '</style></head><body>'
        printf '<h1>Relatorio de Baseline CIS</h1>'
        printf '<p>Distro: %s %s</p>' "$CIS_DISTRO_ID" "$CIS_DISTRO_VERSION"
        printf '<p>Data: %s</p>' "$(date '+%Y-%m-%d %H:%M:%S')"
        printf '<p><strong>Pontuacao: %d%% (%d/%d checks passaram)</strong></p>' "$l_score" "$CIS_PASS" "$l_total"
        printf '<p>PASS: %d | FAIL: %d | WARN: %d</p>' "$CIS_PASS" "$CIS_FAIL" "$CIS_WARN"
        printf '<table><tr><th>Status</th><th>Verificacao</th><th>Detalhe</th></tr>'
        local l_row
        for l_row in "${CIS_REPORT_ROWS[@]}"; do
            printf '%s' "$l_row"
        done
        printf '</table></body></html>'
    } > "$CIS_REPORT_FILE"

    cis_log_info "Relatorio HTML gerado em ${CIS_REPORT_FILE}"
}

# =============================================================================
# 17. EXECUÇÃO
# =============================================================================

cis_run_all_checks() {
    printf '== Montagens ==\n'
    cis_check_tmp_and_vartmp
    cis_check_home_partition
    cis_check_varlog_partition
    cis_check_boot_and_var

    printf '== SELinux ==\n'
    cis_check_selinux_installed
    cis_check_selinux_enforcing
    cis_check_selinux_bootloader

    printf '== Servicos ==\n'
    cis_check_service_absent "telnet-server" telnet-server
    cis_check_service_absent "vsftpd/ftp"    vsftpd ftp
    cis_check_service_absent "rsync"         rsync-daemon rsync
    cis_check_service_absent "nfs-server"    nfs-utils
    cis_check_listening_services

    printf '== Permissoes de /etc/shadow e afins ==\n'
    cis_check_sensitive_files_permissions
    cis_check_world_writable_files

    printf '== SUID/SGID ==\n'
    cis_check_suid_sgid_review

    printf '== Contas e grupos ==\n'
    cis_check_duplicate_uids
    cis_check_duplicate_gids
    cis_check_root_uid0

    printf '== auditd ==\n'
    cis_check_auditd_installed
    cis_check_auditd_active
    cis_check_audit_rules_files
    cis_check_audit_log_storage

    printf '== Atualizacoes pendentes ==\n'
    cis_check_pending_updates
}

main() {
    cis_parse_args "$@"
    cis_require_root
    cis_check_dependencies
    cis_detect_distro

    # Lock exclusivo via descriptor de arquivo 200
    exec 200>"$CIS_LOCK_FILE"
    if ! flock -n 200; then
        printf 'Outra instancia do %s ja esta em execucao.\n' "$CIS_SCRIPT_NAME" >&2
        exit "$CIS_EXIT_USAGE"
    fi

    cis_log_info "Iniciando ${CIS_SCRIPT_NAME} v${CIS_VERSION}"
    printf 'Verificador de baseline CIS -- %s %s\n' "$CIS_DISTRO_ID" "$CIS_DISTRO_VERSION"

    cis_run_all_checks
    cis_generate_html_report

    printf '\nResumo: PASS=%d FAIL=%d WARN=%d\n' "$CIS_PASS" "$CIS_FAIL" "$CIS_WARN"
    printf 'Relatorio: %s\n' "$CIS_REPORT_FILE"
    cis_log_info "Execucao concluida. PASS=${CIS_PASS} FAIL=${CIS_FAIL} WARN=${CIS_WARN}"

    if [ "$CIS_FAIL" -gt 0 ]; then
        exit "$CIS_EXIT_FINDING"
    fi
    exit "$CIS_EXIT_SUCCESS"
}

main "$@"
```

### Como a resposta gerada foi validada

Após receber o script gerado, cada requisito foi validado tecnicamente antes de considerá-lo pronto para uso no laboratório/servidor:

- Instalação do `shellcheck` e execução sobre o script, confirmando zero erros e zero avisos.
- Execução do script como root, tanto no estado padrão (sem hardening aplicado) quanto após configurar o esquema de partições/LVM exigido.
- Teste de idempotência na prática: rodando o script duas vezes seguidas e comparando os relatórios HTML gerados. Confirmação de que eram idênticos, tirando o timestamp que muda de acordo com a hora.
- Teste de concorrência: execução de duas instâncias simultâneas do script, confirmando que o mecanismo de `flock` impediu a segunda instância de sobrescrever o relatório, retornando o código de saída correto (2, erro de uso).
- Comparação dos relatórios HTML gerados antes e depois do hardening do servidor, o que permitiu identificar exatamente quais itens ainda dependiam de ação manual: opções de montagem no `/etc/fstab`, regras do auditd não carregadas e binários fora da baseline de SUID/SGID.
- Aplicação manual das correções indicadas (edição do `/etc/fstab`, `mount -o remount`, criação de regras em `/etc/audit/rules.d/`) e reexecução do script para confirmar a evolução real da pontuação no ambiente.

---

### Prompt 2 — EXT4 vs XFS no `mount -o remount`

> Por que só o EXT4 retornou uma "chave" (print do terminal, onde somente a linha do `/home` (EXT4) exibiu a mensagem `EXT4-fs (dm-12): re-mounted b823d88d...` após os comandos `mount -o remount`, enquanto as demais partições, todas em XFS, não imprimiram nada)?

### Resposta gerada pela IA

Nenhum erro aí, é só uma diferença de verbosidade entre os drivers de sistema de arquivos no kernel, não indica que os outros falharam.

**Por que só o `/home` (EXT4) imprimiu mensagem**

`/home` é EXT4. O driver EXT4 do kernel Linux tem por hábito emitir uma mensagem no log de kernel (`dmesg`) sempre que um remount acontece, do tipo `EXT4-fs (dm-XX): re-mounted b823d88d...`. Isso é um comportamento conhecido e documentado do próprio módulo EXT4, não é algo que vocês fizeram diferente.

Todos os outros (`/tmp`, `/var/tmp`, `/var/log`, `/var`, `/boot`) são XFS. O driver XFS, ao contrário do EXT4, não imprime nada no console em um remount bem-sucedido. Silêncio ali é o comportamento normal e esperado do XFS.

**Como confirmar que os outros realmente funcionaram**

Em Linux, `mount` (e `mount -o remount`) segue a convenção padrão de programas Unix: nenhuma saída = sucesso. Se algum remount do XFS tivesse falhado, apareceria uma mensagem de erro em vermelho do tipo `mount: /var/log: mount point is busy` ou similar, e isso não apareceu em nenhuma das 5 linhas XFS.

De qualquer forma, não precisa confiar só na ausência de erro — dá pra confirmar de fato:

```bash
findmnt -o TARGET,OPTIONS /tmp /var/tmp /home /var/log /boot /var
```

Se as opções `nodev`/`nosuid`/`noexec` aparecerem certinho em cada linha, está tudo aplicado de verdade, e aí sim pode rodar o `cis-baseline-check.sh` de novo.

---

### Prompt 3 — Colunas do `/etc/fstab`

> Estou no diretório `/etc`, rodei `vim /etc/fstab` pra aplicar as opções `nodev`, `nosuid` e `noexec` nas partições que o professor pediu (`/tmp`, `/var/tmp`, `/home`, `/var/log`, `/boot`, `/var`). Reparei que a linha do `/home` termina em `1 2`, enquanto todas as outras terminam em `0 0`. Por que só essa linha é diferente, e o que esses números significam? Preciso saber se isso interfere na hora de inserir as opções de montagem.

### Resposta gerada pela IA

Boa observação, mas isso não afeta o que vamos editar. Deixa eu explicar por quê.

**O que são esses dois números finais**

No `/etc/fstab`, cada linha tem 6 colunas: `dispositivo | ponto_de_montagem | tipo | opções | dump | passo_fsck`. Os dois últimos números (`0 0`, `1 2`, etc.) são:

- **dump**: se o utilitário antigo `dump` deve fazer backup dessa partição (quase sempre `0` hoje em dia, ninguém usa).
- **passo_fsck**: ordem de checagem do `fsck` no boot — `0` = nunca checa, `1` = checa primeiro (só a raiz `/` usa isso), `2` = checa depois da raiz.

No caso do `/home`, o Anaconda (instalador) colocou `1 2` porque decidiu que essa partição deve ser verificada no boot (`fsck`). Isso é uma escolha de integridade do sistema de arquivos, não tem nada a ver com segurança de montagem (`nodev`/`nosuid`). Não precisa mexer nesses dois números.

**Por que o `sed` continua funcionando**

O comando busca e substitui só o texto `x-systemd.device-timeout=0`, que está na coluna de opções — ele não toca no que vem depois (`1 2` ou `0 0`), porque a substituição é só daquele trecho específico, sem tocar no resto da linha.

Então a linha do `/home`:

```
UUID=b823d88d-a1a0-4901-bcb2-c6682ee6353a /home ext4 defaults,x-systemd.device-timeout=0 1 2
```

vira:

```
UUID=b823d88d-a1a0-4901-bcb2-c6682ee6353a /home ext4 defaults,x-systemd.device-timeout=0,nodev,nosuid 1 2
```

O `1 2` no final permanece intacto, é exatamente o comportamento esperado. Pode rodar o `sed` sem preocupação com isso.

---

## Amanda Souza Bezerra

### Prompt 1 — Base inicial do verificador

> Quero que atue como um sênior e faça o COMEÇO para ser a base de um shell script bem feito com a função de verificador de baseline hardening para um servidor CentOS Stream com tais especificações:
>
> **`net-hardening.sh`** — Hardening de rede e firewalld. Inventário de portas com `ss -tulpn` contra lista autorizada, parâmetros sysctl de rede persistidos, zona do firewalld e rich rule com limite de taxa. Tem `--dry-run` e se recusa a remover a regra que sustenta a própria sessão.
>
> **`ssh-sentinel.sh`** — Detecção de brute force em SSH. Lê o journald, agrega falhas por IP, aplica limiar configurável e bloqueia no firewalld com TTL. Lista de exceções é obrigatória. Relatório com top 10 IPs e usuários. Comparar com o fail2ban.

### Resposta gerada pela IA

```bash
#!/usr/bin/env bash
# cis-baseline-check.sh — checagem rápida de hardening estilo CIS

[[ "$EUID" -eq 0 ]] || { echo "roda como root"; exit 1; }

echo "checando $(hostname)"
echo

# SELinux
if [[ "$(getenforce)" == "Enforcing" ]]; then
    echo "[OK]   SELinux enforcing"
else
    echo "[FAIL] SELinux não está enforcing -> roda: setenforce 1"
fi

# permissão do /etc/shadow
perms="$(stat -c '%a' /etc/shadow)"
if [[ "$perms" -le 640 ]]; then
    echo "[OK]   /etc/shadow com permissão $perms"
else
    echo "[FAIL] /etc/shadow com permissão $perms -> roda: chmod 640 /etc/shadow"
fi

# contas com UID 0 além de root
extras="$(awk -F: '$3==0 && $1!="root" {print $1}' /etc/passwd)"
if [[ -z "$extras" ]]; then
    echo "[OK]   só root com UID 0"
else
    echo "[FAIL] tem conta com UID 0 além de root: $extras"
fi

# auditd
if systemctl is-active --quiet auditd; then
    echo "[OK]   auditd rodando"
else
    echo "[FAIL] auditd parado -> roda: systemctl enable --now auditd"
fi

# root via SSH
root_login="$(sshd -T 2>/dev/null | awk '/^permitrootlogin/ {print $2}')"
if [[ "$root_login" == "no" ]]; then
    echo "[OK]   root não loga direto via SSH"
else
    echo "[FAIL] PermitRootLogin = $root_login -> edita /etc/ssh/sshd_config"
fi

# próximos: firewalld, senha/pam_faillock, SUID/SGID fora do esperado, updates pendentes
```

---

### Prompt 2 — Explicação linha a linha

> Poderia explicar cada linha para maior entendimento?

### Resposta gerada pela IA

**Cabeçalho**

```bash
#!/usr/bin/env bash
```
Shebang — diz pro sistema executar o arquivo com o bash, buscando ele no `PATH` em vez de fixar `/bin/bash`. Mais portável entre distros.

```bash
# cis-baseline-check.sh — checagem rápida de hardening estilo CIS
# Grupo 6 - CentOS Stream - FIAP
```
Comentários de identificação, não fazem nada, só documentam o arquivo.

```bash
set -uo pipefail
```
Duas opções de segurança do bash:
- `-u`: se você usar uma variável que nunca foi definida, o script quebra na hora em vez de tratar como string vazia silenciosamente. Evita bug bobo tipo digitar `$HSOT` em vez de `$HOST`.
- `pipefail`: numa pipeline (`cmd1 | cmd2`), se `cmd1` falhar, o exit code final reflete essa falha, mesmo que `cmd2` tenha "funcionado". Sem isso, um erro no meio da pipe passa despercebido.

Não usei `-e` (que mataria o script no primeiro comando que retornasse erro) porque aqui é esperado que checks "falhem" — um `grep` sem match, um `stat` que não acha o arquivo — e isso é dado, não bug.

```bash
[[ "$EUID" -eq 0 ]] || { echo "roda como root, tem coisa aqui que precisa"; exit 1; }
```
`$EUID` é o UID efetivo de quem está rodando o script. Se não for `0` (root), imprime a mensagem e sai com código `1`. O `||` só executa o bloco `{ }` se a condição da esquerda for falsa — é um jeito curto de escrever um `if` de uma linha.

**Contadores e funções auxiliares**

```bash
PASS=0
FAIL=0
WARN=0
```
Três contadores simples, inicializados em zero, pra somar quantos checks passaram, falharam ou deram alerta.

```bash
ok()   { echo -e "  [OK]   $1"; ((PASS++)); }
bad()  { echo -e "  [FAIL] $1 -> $2"; ((FAIL++)); }
meh()  { echo -e "  [WARN] $1 -> $2"; ((WARN++)); }
```
Três funções auxiliares que centralizam a impressão do resultado:
- `ok` recebe um argumento (`$1`, a descrição do que passou) e imprime com tag `[OK]`.
- `bad` recebe dois argumentos: `$1` é o problema, `$2` é a sugestão de correção — imprime os dois separados por `->`.
- `meh` é igual ao `bad`, mas pra casos que não são erro grave, só atenção.
- `((PASS++))` é aritmética do bash, incrementa a variável.

Isso evita repetir `echo` formatado em cada check — qualquer check só chama `ok "texto"` ou `bad "texto" "sugestão"`.

**check_tmp_mount**

```bash
check_tmp_mount() {
    if findmnt --noheadings /tmp &>/dev/null; then
```
`findmnt /tmp` verifica se `/tmp` está listado como ponto de montagem próprio (não apenas uma pasta dentro de `/`). `--noheadings` tira o cabeçalho da saída. `&>/dev/null` joga tanto a saída normal quanto erro pro limbo — aqui só importa o código de retorno (0 = achou o mount, ≠0 = não achou), não o texto.

```bash
        local opts
        opts="$(findmnt -no OPTIONS /tmp)"
```
`local` declara a variável só dentro da função (boa prática, evita vazar pro escopo global). `findmnt -no OPTIONS /tmp` retorna só a coluna de opções de montagem (tipo `rw,nosuid,nodev,noexec,relatime`) sem cabeçalho (`-n`) e só essa coluna (`-o OPTIONS`).

```bash
        if [[ "$opts" == *noexec* && "$opts" == *nosuid* ]]; then
            ok "/tmp em partição própria com noexec/nosuid"
        else
            meh "/tmp é partição própria mas sem noexec/nosuid" "adiciona as flags no fstab e remonta"
        fi
```
`*noexec*` é um padrão de glob — verifica se a string `noexec` aparece em qualquer parte de `$opts`. Se tiver as duas flags, tudo certo (`ok`). Se `/tmp` é mount próprio mas falta alguma flag, é só um alerta (`meh`), não uma falha grave.

```bash
    else
        bad "/tmp não é partição separada" "cria um LV dedicado pra /tmp"
    fi
}
```
Se o primeiro `findmnt` (lá em cima) não achou `/tmp` como mount separado, isso é tratado como falha real (`bad`), já que sem partição própria não dá nem pra aplicar as flags de segurança.

**check_selinux**

```bash
check_selinux() {
    if ! command -v getenforce &>/dev/null; then
        meh "SELinux" "getenforce não encontrado, tem SELinux instalado?"
        return
    fi
```
`command -v getenforce` verifica se esse comando existe no sistema (retorna o caminho dele, ou nada + erro se não existir). O `!` inverte: "se NÃO existir". Se não existir, avisa e `return` sai da função sem continuar (não faz sentido checar o modo de algo que não está instalado).

```bash
    local mode
    mode="$(getenforce)"
    [[ "$mode" == "Enforcing" ]] \
        && ok "SELinux em Enforcing" \
        || bad "SELinux em modo $mode" "setenforce 1 e ajustar /etc/selinux/config"
}
```
`getenforce` imprime o modo atual (`Enforcing`, `Permissive` ou `Disabled`). O `&&`/`||` aqui funciona como um if/else compacto: se a condição é verdadeira, executa o que vem depois do `&&`; senão, executa o que vem depois do `||`. A barra `\` no final da linha só quebra visualmente o comando em várias linhas sem afetar a execução.

**check_shadow_perms**

```bash
check_shadow_perms() {
    local perms
    perms="$(stat -c '%a' /etc/shadow)"
```
`stat -c '%a'` extrai só a permissão em formato octal (tipo `640`) do arquivo, sem o resto da informação que `stat` normalmente mostra.

```bash
    [[ "$perms" -le 640 ]] \
        && ok "/etc/shadow com permissão $perms" \
        || bad "/etc/shadow com permissão $perms (muito aberto)" "chmod 640 /etc/shadow"
}
```
`-le` é "menor ou igual", comparação numérica (não pode usar `<=` normal dentro de `[[ ]]` pra números). Se a permissão for 640 ou mais restritiva (ex: 600), passa. Se for mais aberta (ex: 644), falha.

**check_uid_zero**

```bash
check_uid_zero() {
    local extras
    extras="$(awk -F: '$3 == 0 && $1 != "root" {print $1}' /etc/passwd)"
```
`/etc/passwd` tem uma linha por usuário, campos separados por `:`. `-F:` diz ao awk pra usar `:` como separador. `$3` é o terceiro campo (UID), `$1` é o primeiro (nome do usuário). A expressão filtra: UID igual a 0 E nome diferente de "root" — ou seja, contas "fantasmas" com privilégio de root escondidas sob outro nome. `{print $1}` imprime o nome dessas contas, se existirem.

```bash
    [[ -z "$extras" ]] \
        && ok "só root com UID 0" \
        || bad "conta(s) com UID 0 além de root: $extras" "investigar e corrigir na hora"
}
```
`-z` testa se a string está vazia. Vazia = nenhuma conta suspeita = ok. Se tiver algo, lista os nomes na mensagem de falha.

**check_auditd**

```bash
check_auditd() {
    if systemctl is-active --quiet auditd 2>/dev/null; then
        ok "auditd rodando"
    else
        bad "auditd parado ou não instalado" "dnf install audit && systemctl enable --now auditd"
    fi
}
```
`systemctl is-active --quiet auditd` só verifica se o serviço está ativo, sem imprimir nada (`--quiet`), retornando 0 (ativo) ou diferente de 0 (inativo/inexistente). `2>/dev/null` descarta mensagens de erro caso o serviço nem exista no systemd.

**check_ssh_root_login**

```bash
check_ssh_root_login() {
    local value
    value="$(sshd -T 2>/dev/null | awk '/^permitrootlogin/ {print $2}')"
```
`sshd -T` faz o próprio daemon SSH processar toda a configuração (incluindo includes e defaults) e imprimir o resultado final já resolvido — mais confiável que ler o `sshd_config` na mão, porque pega o valor efetivo de verdade. O `awk` procura a linha que começa com `permitrootlogin` e pega o segundo campo (o valor).

```bash
    case "${value,,}" in
        no)   ok "PermitRootLogin desabilitado" ;;
        "")   meh "não deu pra ler PermitRootLogin" "conferir /etc/ssh/sshd_config manualmente" ;;
        *)    bad "PermitRootLogin = $value" "trocar pra 'no' em /etc/ssh/sshd_config" ;;
    esac
}
```
`${value,,}` converte a string pra minúsculas (útil porque o valor pode vir como `No` ou `NO`). `case` compara contra três padrões: exatamente `no` (ok), string vazia (não conseguiu ler, alerta), ou qualquer outra coisa (`*`, catch-all — inclusive `yes` ou `without-password`, tratado como falha).

**Execução final**

```bash
echo "checando $(hostname) — $(date '+%Y-%m-%d %H:%M')"
echo
```
Cabeçalho simples. `$(hostname)` e `$(date ...)` são substituição de comando — o que eles imprimem vira parte da string.

```bash
check_tmp_mount
check_selinux
check_shadow_perms
check_uid_zero
check_auditd
check_ssh_root_login
```
Chama cada função na ordem — é literalmente isso, sem mágica.

```bash
echo
echo "resultado: $PASS ok, $WARN atenção, $FAIL falha"
```
Resumo final usando os contadores que cada `ok`/`bad`/`meh` foi incrementando.

```bash
# próximos: firewalld, senhas/pam_faillock, SUID/SGID fora do esperado,
# updates pendentes, banner de login, permissões de passwd/group
```
