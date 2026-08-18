# Writeup: NFS Root Squashing Desabilitado - Metasploitable2

## Ambiente
- Atacante: Kali Linux (VM VirtualBox)
- Alvo: Metasploitable2 (VM VirtualBox)
- Rede: Internal Network isolada (192.168.56.0/24)

## 1. Reconhecimento
Identificado na varredura inicial com Nmap: NFS rodando na porta
2049/tcp.

## 2. Identificação da vulnerabilidade
Verificação dos compartilhamentos exportados:

    showmount -e 192.168.56.10
    -> Export list for 192.168.56.10: / *

O diretório raiz inteiro (/) está exportado para qualquer host
(*), sem restrição de rede e sem "root squashing" (mecanismo que
normalmente rebaixa o usuário root remoto para um usuário sem
privilégio ao acessar via NFS).

## 3. Exploração
Montagem do compartilhamento na máquina atacante:

    mkdir /tmp/nfs_mount
    sudo mount -t nfs 192.168.56.10:/ /tmp/nfs_mount

Como o NFS confia no UID do cliente sem autenticação adicional,
assumir o UID 0 (root) localmente concede as mesmas permissões
remotamente:

    sudo su -
    echo "teste root via NFS" > /tmp/nfs_mount/root/teste_nfs.txt
    cat /tmp/nfs_mount/root/teste_nfs.txt

Resultado: escrita bem-sucedida dentro de /root do sistema
remoto, sem qualquer autenticação.

## 4. Impacto
Esse tipo de acesso permitiria, por exemplo, escrever uma chave
pública SSH em /root/.ssh/authorized_keys, obtendo login SSH
como root sem senha — uma via de escalação de privilégio total
mesmo em cenários onde o acesso inicial (como visto nos
exploits do DistCC e PostgreSQL) fosse limitado a um usuário
sem privilégios.

## Conclusão
Falha crítica de configuração: NFS exportando a raiz do sistema
para qualquer host, sem root squashing. Como correção: nunca
exportar o diretório raiz via NFS, restringir exports a
diretórios específicos e hosts confiáveis, e sempre habilitar
root squashing (root_squash) na configuração do NFS.
