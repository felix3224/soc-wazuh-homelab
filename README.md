# 🛡️ SOC Wazuh Homelab

Laboratório prático de SOC (Security Operations Center) utilizando **Wazuh** como SIEM/EDR, integrado com **Shuffle** e **VirusTotal** para automação de resposta a incidentes. O ambiente simula ameaças reais com Kali Linux e monitora endpoints Windows com agente Wazuh.

---

## 🧱 Arquitetura do Ambiente

```
┌─────────────────────────────────────────────────────────┐
│                      HOME LAB                           │
│                                                         │
│  ┌──────────────┐    ┌──────────────┐    ┌───────────┐  │
│  │     Wazuh    │◄──►│  Windows 11  │    │Kali Linux │  │
│  │  (SIEM/EDR)  │    │ (Wazuh Agent)│◄───│ (Atacante)│  │
│  └──────┬───────┘    └──────────────┘    └───────────┘  │
│         │                                               │
│         ▼                                               │
│  ┌──────────────┐    ┌──────────────┐                   │
│  │   Shuffle    │───►│  VirusTotal  │                   │
│  │  (SOAR/Auto) │    │    (API)     │                   │
│  └──────────────┘    └──────────────┘                   │
└─────────────────────────────────────────────────────────┘
```

### Máquinas Virtuais

| VM | Função | SO |
|----|--------|----|
| **Wazuh** | Servidor SIEM/EDR — coleta, correlaciona e analisa logs | Ubuntu 22.04 lts |
| **TheHIve** | Servidor SIEM/EDR — coleta, correlaciona e analisa logs | Ubuntu 22.04 lts |
| **Shuffle** | Servidor SIEM/EDR — coleta, correlaciona e analisa logs | Ubuntu 22.04 lts |
| **Windows 11** | Endpoint monitorado — roda o Wazuh Agent | Windows 11 |
| **Kali Linux** | Máquina do atacante — executa os testes de intrusão | Kali Linux |

---

## 🔗 Ferramentas & Integrações

### [Wazuh](https://wazuh.com/)
SIEM/EDR open-source que centraliza logs dos endpoints, gera alertas baseados em regras e permite resposta ativa. Neste lab, o servidor roda via OVA e o agente fica instalado no Windows 10.

### [Shuffle](https://shuffler.io/)
Plataforma SOAR (Security Orchestration, Automation and Response) que automatiza os fluxos de trabalho gerados pelos alertas do Wazuh. Quando um alerta é disparado, o Shuffle aciona a API do VirusTotal automaticamente.

### [VirusTotal](https://www.virustotal.com/)
Serviço de análise de arquivos e IPs maliciosos. Integrado via API ao Shuffle para enriquecer os alertas do Wazuh com reputação de IPs e hashes automaticamente.

---

## 🧪 Cenários de Teste

### Teste 1 — Brute Force em RDP (Windows)
**Ferramenta:** Hydra / Crowbar (Kali Linux)  
**Objetivo:** Simular ataque de força bruta na porta RDP (3389) do Windows 11 e validar se o Wazuh detecta as múltiplas tentativas de autenticação falha.  
**Regras acionadas:** Wazuh Rule ID `60106`, `60122` (Windows authentication failures)


```bash
# Exemplo de ataque com Hydra (Kali)
hydra -l administrator -P /usr/share/wordlists/rockyou.txt rdp://<IP-WINDOWS>
```

---

### Teste 2 — FIM (File Integrity Monitoring)
**Ferramenta:** Wazuh Agent (Windows)  
**Objetivo:** Monitorar alterações em arquivos e diretórios críticos do Windows. Qualquer criação, modificação ou deleção em pastas sensíveis gera um alerta no dashboard do Wazuh.  
**Configuração:** `ossec.conf` no agente com diretórios monitorados via `<directories check_all="yes">`.

```xml
<!-- ossec.conf no agente Windows -->
<syscheck>
  <directories check_all="yes">C:\Users\Administrator\Desktop</directories>
  <directories check_all="yes">C:\Windows\System32</directories>
</syscheck>
```

---

### Teste 3 — Bloqueio Automático de IP por Brute Force
**Ferramenta:** Wazuh Active Response  
**Objetivo:** Configurar uma regra que bloqueia automaticamente o IP de origem após um número definido de tentativas falhas de autenticação, usando o recurso de Active Response do Wazuh.  
**Mecanismo:** `firewall-drop` command no Wazuh + regra personalizada de threshold.

```xml
<!-- ossec.conf no servidor Wazuh -->
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>60122</rules_id>
  <timeout>600</timeout>
</active-response>
```

---

## 🔄 Fluxo de Automação (Shuffle + VirusTotal)

```
Wazuh Alert disparado
        │
        ▼
  Webhook → Shuffle
        │
        ▼
  Extrai IP do alerta
        │
        ▼
  Consulta VirusTotal API
        │
        ▼
  IP Malicioso? ──► Sim ──► Cria relatório / Notificação
        │
        Não
        │
        ▼
     Log registrado
```

---

## 🚀 Como Reproduzir

### 1. Subir o Servidor Wazuh
- Baixe a [OVA oficial do Wazuh](https://documentation.wazuh.com/current/deployment-options/virtual-machine/virtual-machine.html)
- Importe no VirtualBox ou VMware
- Acesse o dashboard: `https://<IP-WAZUH>` (usuário: `admin`)

### 2. Instalar o Agente no Windows 10
```powershell
# No Windows 10, via PowerShell (admin)
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.x.x-1.msi -OutFile wazuh-agent.msi
msiexec.exe /i wazuh-agent.msi WAZUH_MANAGER="<IP-WAZUH>" /q
NET START WazuhSvc
```

### 3. Configurar Shuffle
- Deploy via Docker ou conta em [shuffler.io](https://shuffler.io)
- Criar workflow com trigger via Wazuh webhook
- Adicionar ação de consulta à API do VirusTotal

### 4. Configurar chave da API VirusTotal
- Criar conta gratuita em [virustotal.com](https://www.virustotal.com)
- Gerar API Key e inserir no workflow do Shuffle

---

## 📁 Estrutura do Repositório

```
soc-wazuh-homelab/
├── README.md
├── configs/
│   ├── ossec-server.conf       # Configuração do servidor Wazuh
│   ├── ossec-agent.conf        # Configuração do agente Windows
│   └── active-response.conf    # Regras de resposta ativa
├── rules/
│   └── custom-rules.xml        # Regras personalizadas Wazuh
├── shuffle/
│   └── workflow-virustotal.json # Exportação do workflow Shuffle
└── screenshots/
    ├── brute-force-alert.png
    ├── fim-alert.png
    └── ip-blocked.png
```

---

## 📚 Referências

- [Documentação Wazuh](https://documentation.wazuh.com/)
- [Shuffle Documentation](https://shuffler.io/docs)
- [VirusTotal API](https://developers.virustotal.com/reference)
- [Wazuh Active Response](https://documentation.wazuh.com/current/user-manual/capabilities/active-response/)

---

## 📄 Licença

MIT License — veja o arquivo [LICENSE](LICENSE) para detalhes.
