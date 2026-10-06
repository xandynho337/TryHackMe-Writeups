# Writeup: Enterprise - TryHackMe

**Dificuldade:** Difícil  
**OS:** Windows  
**Categorias:** Active Directory e PrivEsc  
**Data de Conclusão:** 06/10/2025  

---

## Resumo Executivo
> Visão geral do impacto técnico e de negócio das vulnerabilidades encontradas.

O objetivo deste laboratório foi obter acesso com privilégios máximos (`root` / `SYSTEM`) no alvo. A intrusão inicial ocorreu através de [Nome da Vulnerabilidade / Vetor], permitindo [impacto, ex: Execução Remota de Código (RCE)]. A escalada de privilégios foi alcançada explorando [Mecânica/Falha de Configuração].

---

## 1. Reconhecimento & Enumeração

### Varredura de Portas (Nmap)
Sintaxe utilizada para a varredura inicial:

```bash
nmap -sC -sV -oA nmap/initial <IP_ALVO>
```

**Portas Abertas Encontradas:**
* **80/HTTP:** Servidor Web Apache 2.4.41
* **445/SMB:** Compartilhamento de arquivos habilitado

### Enumeração do Serviço Web / SMB
Descreva brevemente o que foi descoberto nas ferramentas (ex: `ffuf`, `gobuster`, `smbclient`).

---

## 2. Intrusão Inicial (Initial Access)

### Exploração da Vulnerabilidade
* **Vulnerabilidade:** [Nome da falha, ex: Unauthenticated Remote Code Execution]
* **Vetor de Ataque:** Explicação técnica do motivo da falha existir (ex: falta de higienização no parâmetro X).

Payload / Comando utilizado para a exploração:

```bash
python3 exploit.py -t http://<IP_ALVO>/login -p payload.sh
```

> **Acesso obtido:** Shell reversa estabelecida como o usuário `www-data`.

---

## 3. Escalada de Privilégios (Privilege Escalation)

### Enumeração Interna
Identificação do vetor de elevação de privilégios:

```bash
sudo -l
```

### Exploração e Obtenção de Root / SYSTEM
Explicação técnica da execução do bypass ou exploração do binário/permissão SUID.

```bash
# Comando utilizado para obter acesso privilegiado
```

> **Acesso obtido:** Acesso completo de administrador (`root` / `NT AUTHORITY\SYSTEM`).

---

## 4. Recomendações de Mitigação (Remediação)

1. **Aplicação Web:** Sanear e validar todas as entradas do usuário no backend antes do processamento.
2. **Hardening do SO:** Aplicar o princípio do menor privilégio, removendo permissões `sudo` desnecessárias para contas de serviço web.
