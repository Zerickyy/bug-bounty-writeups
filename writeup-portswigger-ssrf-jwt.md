# Writeup: PortSwigger Web Security Academy - SSRF e JWT Attacks

## Ambiente
- Atacante: Kali Linux (VM VirtualBox)
- Alvo: Laboratórios do PortSwigger Web Security Academy (hospedados remotamente)
- Ferramentas: Burp Suite (Proxy, Intruder, Repeater, extensão JWT Editor)

---

## Parte 1 — SSRF (Server-Side Request Forgery)

### Lab 1: Basic SSRF against the local server (Apprentice)
Funcionalidade de "check stock" aceita URL completa via parâmetro
`stockApi`, sem validação. Exploração:

    stockApi=http://localhost/admin
    stockApi=http://localhost/admin/delete?username=carlos

Resultado: acesso ao painel administrativo e exclusão de usuário
via requisição forjada pelo próprio servidor (bypass de controle
de acesso baseado em origem "confiável" por vir do loopback).

### Lab 2: Basic SSRF against another back-end system (Apprentice)
Mesma vulnerabilidade, mas o alvo é um sistema interno na rede
privada (192.168.0.0/24), não o próprio servidor. Uso do Burp
Intruder para escanear a faixa de IP na porta 8080:

    stockApi=http://192.168.0.§1§:8080/admin

IP 192.168.0.1 identificado como único com resposta HTTP 200
(demais retornaram erro de conexão). Requisição enviada ao Burp
Repeater para deletar o usuário via:

    stockApi=http://192.168.0.1:8080/admin/delete?username=carlos

### Lab 3: SSRF with blacklist-based input filter (Practitioner)
Aplicação bloqueia duas camadas separadamente: o IP (`127.0.0.1`,
`localhost`) e o caminho (`/admin`). Bypasses:

    http://127.1/                          -> bypass do filtro de IP
                                               (notação abreviada)
    http://127.1/%2561dmin                 -> bypass do filtro de path
                                               (duplo URL-encoding da
                                               letra "a": %2561 decodifica
                                               uma vez para %61, e uma
                                               segunda vez no servidor
                                               real para "a")

Payload final:

    stockApi=http://127.1/%2561dmin/delete?username=carlos

### Lab 4: SSRF with whitelist-based input filter (Expert)
Aplicação valida que o hostname da URL corresponde exatamente a
`stock.weliketoshop.net`. Bypass usando sintaxe de credenciais
embutidas na URL (`usuario@host`) combinada com fragmento (`#`)
duplamente codificado:

    http://username@stock.weliketoshop.net/       -> aceito (parser
                                                      reconhece o host
                                                      esperado)
    http://username%2523@stock.weliketoshop.net/  -> "Internal Server
                                                      Error" (indica
                                                      tentativa de
                                                      conexão a
                                                      "username")

Payload final:

    stockApi=http://localhost:80%2523@stock.weliketoshop.net/admin/delete?username=carlos

O parser da whitelist identifica `stock.weliketoshop.net` como o
host (aprovando a URL), mas o servidor real, ao decodificar
%2523 -> %23 -> "#", interpreta tudo após o "#" como fragmento,
conectando de fato apenas a `localhost:80`.

---

## Parte 2 — JWT Attacks

### Lab 1: JWT authentication bypass via unverified signature (Apprentice)
Servidor decodifica o JWT sem verificar a assinatura. Edição
direta do payload via extensão JWT Editor no Burp Repeater:

    "sub": "wiener"  ->  "sub": "administrator"

Requisição enviada sem recalcular a assinatura — aceita
normalmente, concedendo acesso ao painel /admin.

### Lab 2: JWT authentication bypass via flawed signature verification (Apprentice)
Servidor aceita tokens não assinados (alg: none). Uso da função
"Attack -> None" da extensão JWT Editor, após editar o payload
para "sub": "administrator". Erro inicial identificado: JSON
malformado (aspa de fechamento ausente no valor do campo sub),
corrigido antes do reenvio bem-sucedido.

### Lab 3: JWT authentication bypass via weak signing key (Practitioner)
Servidor usa uma chave secreta HS256 fraca. Chave quebrada por
força bruta offline:

    john --format=HMAC-SHA256 --wordlist=jwt.secrets.list jwt_token.txt

(hashcat indisponível por falta de suporte a GPU na VM,
substituído por John the Ripper, mesma solução usada no crack de
hash MD5 do Dia 1)

Resultado: chave identificada como "secret1" em segundos.
Chave cadastrada na aba "JWT Editor Keys" do Burp como chave
simétrica, usada para re-assinar um token modificado
("sub": "administrator") com assinatura HMAC-SHA256 válida,
concedendo acesso legítimo (do ponto de vista do servidor) ao
painel administrativo.

---

## Conclusão geral
Sete laboratórios completos cobrindo os principais vetores de
SSRF (ataque direto, ataque a sistema interno, bypass de
blacklist, bypass de whitelist) e os três cenários mais comuns
de falha de verificação de JWT (ausência de verificação,
aceitação de "alg: none", e chave secreta fraca).

Recomendações gerais:
- SSRF: nunca confiar em requisições vindas do próprio servidor
  como automaticamente "confiáveis"; validar destino de URLs com
  parsing robusto de RFC 3986, não comparação de string simples
- JWT: sempre usar a função de verificação completa da biblioteca
  (nunca apenas decode); rejeitar explicitamente alg: none;
  usar chaves secretas geradas aleatoriamente com entropia
  suficiente (256+ bits), nunca strings previsíveis
