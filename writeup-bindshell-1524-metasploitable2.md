# Writeup: Bindshell Root Exposto - Porta 1524 - Metasploitable2

## Ambiente
- Atacante: Kali Linux (VM VirtualBox)
- Alvo: Metasploitable2 (VM VirtualBox)
- Rede: Internal Network isolada (192.168.56.0/24)

## 1. Reconhecimento
Identificado na varredura inicial com Nmap: porta 1524/tcp aberta,
identificada pelo próprio Nmap como "Metasploitable root shell".

## 2. Natureza da falha
Diferente das explorações anteriores (vsftpd e UnrealIRCd), essa
não é uma vulnerabilidade de código explorada via exploit — é um
erro de configuração (misconfiguration): um shell root foi
deixado escutando diretamente nessa porta, sem qualquer forma de
autenticação.

## 3. "Exploração"
Conexão direta via netcat, sem necessidade de exploit:

    nc 192.168.56.10 1524

## 4. Confirmação de acesso
    whoami          -> root

## Conclusão
Acesso root obtido apenas conectando na porta, sem exploit e sem
credenciais. Esse tipo de falha (serviço administrativo exposto
sem autenticação) é tão ou mais perigoso que uma vulnerabilidade
de código, porque não exige nenhum conhecimento técnico avançado
para ser explorado — só é preciso encontrar a porta aberta.
Como correção: nunca expor shells/serviços administrativos sem
autenticação, e usar firewall para restringir acesso a portas
internas.
